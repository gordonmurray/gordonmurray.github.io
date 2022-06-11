---
layout: post
title:  "Starting a Kafka journey"
date:   2022-05-29 22:35
categories: kafka
---

My first exposure to binary logs, or binlogs were just something that database replicas needed to use to keep up to date. I didn't need to know anything more than that to keep a database and its replicas online.

Over time I learned that a binlog had different formats such as Mixed or Row based and could be read by more than just replicas. Tools like AWS's [Database Migration Service (DMS)](https://aws.amazon.com/dms/) could read from a source database and use the binlog for Change Data Capture (CDC) to keep up to date with any changes to the data in the source database.

While reading about CDC, things like events and streams are mentioned and tools like [Apache Spark](https://spark.apache.org/) and [Apache Kafka](https://kafka.apache.org/) too.

Apache Spark seems to be for large scale data needs and Kafka seems more approachable for event streaming so I started to look in to it.

I picked up "[Kafka - The Definitive Guide](https://www.amazon.co.uk/Kafka-Definitive-Real-Time-Stream-Processing/dp/1492043087/ref=sr_1_5?crid=1BPYEQA8AX2SR)" and "[Mastering Kafka Streams and ksqlDB](https://www.amazon.co.uk/Mastering-Kafka-Streams-ksqlDB-real-time/dp/1492062499/ref=sr_1_1?crid=1BPYEQA8AX2SR)" through work and started reading.

I quickly learned of Apache Zookeeper used alongside Kafka to help manage a leader in a cluster, I created a small Kafka + Zookeeper project using Terraform [https://github.com/gordonmurray/terraform_aws_kafka_zookeeper](https://github.com/gordonmurray/terraform_aws_kafka_zookeeper) to gain some first hand experience.

I was able to get a zookeeper cluster and a Kafka cluster running, though not fully in an automated way that Id like through Terraform. I needed to SSH in and update the config on each instance to make each node aware of itself and other nodes.  I wanted to automate it fully so I could tear it down each night and create it again a few days later when I had time to try it again.

I read that in Kafka circles, Zookeeper was a bit of a pain and that later versions of Kafka would have its own method for mananging itself and would drop zookeeper. I started another Terraform project using only Kafka [https://github.com/gordonmurray/terraform_aws_kafka](https://github.com/gordonmurray/terraform_aws_kafka). While I was working on that, I read about AWS hosted version of Kafka called MSK and a just-released [serverless MSK](https://aws.amazon.com/msk/features/msk-serverless/) option.

The serverless option isn't yet available in Terraform, theres an [open PR for it here](https://github.com/hashicorp/terraform-provider-aws/issues/22058) so I created a small cluster using the aws_msk_cluster terraform resource. It takes 15 minutes or so to make a cluster, but still faster than my own projects so far. I was interested in getting data in to the cluster and consuming/transforming data so I gave up the self hosted option for now and started using MSK.

```
# 3 x t3.small: $142.28/month
resource "aws_msk_cluster" "kafka" {
  cluster_name           = "kafka"
  kafka_version          = "2.6.2"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.t3.small"
    ebs_volume_size = 100
    client_subnets = [
      aws_subnet.private_subnet_1a.id,
      aws_subnet.private_subnet_1b.id,
      aws_subnet.private_subnet_1c.id
    ]

    security_groups = [
      aws_security_group.kafka.id,
      aws_security_group.vpn.id,
    ]
  }

  configuration_info {
    arn      = aws_msk_configuration.configuration_debezium.arn
    revision = 1
  }

  client_authentication {
    sasl {
      iam   = false
      scram = false
    }
  }

  encryption_info {
    encryption_at_rest_kms_key_arn = aws_kms_key.kafka_key.arn

    encryption_in_transit {
      client_broker = "TLS_PLAINTEXT"
      in_cluster    = true
    }
  }

  open_monitoring {
    prometheus {
      jmx_exporter {
        enabled_in_broker = true
      }
      node_exporter {
        enabled_in_broker = true
      }
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.kafka_broker_logs.name
      }

    }
  }

  tags = {
    Name      = "kafka"
  }
}

resource "aws_msk_configuration" "configuration_debezium" {
  kafka_versions = ["2.6.2"]
  name           = "kafka-configuration"
  description    = "MSK config, initially the auto create topic for Debezium"

  server_properties = <<PROPERTIES
auto.create.topics.enable = true
zookeeper.connection.timeout.ms = 1000
PROPERTIES
}
```

I needed to get some data in to the new kafka cluster. I heard of [Debezium](https://debezium.io/) in the past and wasn't exactly sure what it was. Turns out its developed by Redhat. It uses a Kafka connector framework to read from MySQL / MariaDBs binlog and creates/populates Kafka topics ready to consume.

This post on Medium walked me though setting up Debezium https://garrett-jester.medium.com/build-a-real-time-backend-infrastructure-with-aws-msk-rds-ec2-and-debezium-1900c3ee5e67

I started a new ec2 intance and followed the steps:

### Created a database user

For debezium on an existing database I wanted to read from

```
CREATE USER 'debezium'@'%' IDENTIFIED BY '{ password }';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT, LOCK TABLES ON *.* TO 'debezium' IDENTIFIED BY '{ password }';
FLUSH PRIVILEGES;
```

### Install Kafka Connect

```
sudo apt-get update
wget -qO - https://packages.confluent.io/deb/5.3/archive.key | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.3 stable main"
sudo apt-get update && sudo apt-get install confluent-hub-client confluent-common confluent-kafka-2.12
```

### Some changes to /etc/kafka/connect-distributed.properties

```
bootstrap.servers=${COPIED_BOOTSTRAP_SERVER_ADDRESSES}
group.id=debezium-cluster
offset.storage.replication.factor=2
config.storage.replication.factor=2
status.storage.replication.factor=2
plugin.path=/usr/share/java,/usr/share/confluent-hub-components
```

### Install the Debezium Connector

```
sudo nano /lib/systemd/system/confluent-connect-distributed.service
sudo apt install default-jre -y
sudo confluent-hub install debezium/debezium-connector-mysql:latest
sudo systemctl enable confluent-connect-distributed
sudo systemctl start confluent-connect-distributed
```

### Created a connector config.json:

```
{
    "name": "master-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "{ RDS address }",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "{ Password }",
        "database.server.id": "1",
        "database.server.name": "database",
        "database.include.list": "database",
        "table.include.list": "database.table",
        "database.history.kafka.bootstrap.servers": "{ bootstrap servers from MSK }",
        "database.history.kafka.topic": "history.database",
        "include.schema.changes": "false"
    }
}
```

Add my connector:

```
curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" http://localhost:8083/connectors -d @config.json
```

I could now list some topics created by Debezium:

```
list topics:
./kafka/bin/kafka-topics.sh --list --bootstrap-server { bootstrap servers from MSK  }
```

And follow a topic using

```
./kafka/bin/kafka-console-consumer.sh --bootstrap-server { bootstrap servers from MSK  } --from-beginning --topic topic.database.table
```

Data was flowing!

Playing around with the data would need to wait til another evening


