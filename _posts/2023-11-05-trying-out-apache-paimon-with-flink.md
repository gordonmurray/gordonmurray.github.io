---
layout: post
title:  "Trying out Apache Paimon with Flink"
date:   2023-11-05 21:25
categories: data
tags: ["apache","flink","paimon", "cdc", "datalake"]
---

I've been working with Apache Flink recently processing data from Kafka topics. While creating pipelines I wanted to see if I could also send the data from the topics to some longer term storage instead of taking up space on the kafka cluster for less important data. I also wanted to see if I could do this using Flink rather than another tool re-consuming the same topics.

There are a number of options available to send data from Flink to other storage locations such as sinking to a relational database or to s3. While I was reading about Flink Catalogs after some recent Catalog use, I found out about [Apache Paimon.](https://paimon.apache.org/)

> Streaming data lake platform with high-speed data ingestion, changelog tracking and efficient real-time analytics.

In its Getting Started section, it had a guide to working with Flink, so I tried it out.

I used docker compose to get Flink up and running. I added a database to stream some database from and added the Paimon Jars and also some Jars for supporting s3 storage.

The docker compose file is:

```
version: '3.7'

services:
  mariadb:
    image: mariadb:10.6.14
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - ./sql/mariadb.cnf:/etc/mysql/mariadb.conf.d/mariadb.cnf
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"

  jobmanager:
    image: flink:1.17.1
    container_name: jobmanager
    environment:
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
    ports:
      - "8081:8081"
    command: jobmanager
    volumes:
      - ./jars/flink-sql-connector-mysql-cdc-2.4.1.jar:/opt/flink/lib/flink-sql-connector-mysql-cdc-2.4.1.jar
      - ./jars/flink-connector-jdbc-3.1.0-1.17.jar:/opt/flink/lib/flink-connector-jdbc-3.1.0-1.17.jar
      - ./jars/paimon-flink-1.17-0.6-20231030.002108-52.jar:/opt/flink/lib/paimon-flink-1.17-0.6-20231030.002108-52.jar
      - ./jars/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar:/opt/flink/lib/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar
      - ./jars/flink-s3-fs-hadoop-1.17.1.jar:/opt/flink/plugins/s3-fs-hadoop/flink-s3-fs-hadoop-1.17.1.jar
      - ./jars/paimon-s3-0.6-20231030.002108-57.jar:/opt/flink/lib/paimon-s3-0.6-20231030.002108-57.jar
      - ./jobs/job.sql:/opt/flink/job.sql
    deploy:
          replicas: 1
  taskmanager:
    image: flink:1.17.1
    environment:
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
    depends_on:
      - jobmanager
    command: taskmanager
    volumes:
      - ./jars/flink-sql-connector-mysql-cdc-2.4.1.jar:/opt/flink/lib/flink-sql-connector-mysql-cdc-2.4.1.jar
      - ./jars/flink-connector-jdbc-3.1.0-1.17.jar:/opt/flink/lib/flink-connector-jdbc-3.1.0-1.17.jar
      - ./jars/paimon-flink-1.17-0.6-20231030.002108-52.jar:/opt/flink/lib/paimon-flink-1.17-0.6-20231030.002108-52.jar
      - ./jars/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar:/opt/flink/lib/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar
      - ./jars/flink-s3-fs-hadoop-1.17.1.jar:/opt/flink/plugins/s3-fs-hadoop/flink-s3-fs-hadoop-1.17.1.jar
      - ./jars/paimon-s3-0.6-20231030.002108-57.jar:/opt/flink/lib/paimon-s3-0.6-20231030.002108-57.jar
    deploy:
          replicas: 2
```

I started the mini Flink cluster using:

```
docker compose up -d
```

With the cluster running, I added some SQL commands to try out Paimon via a Catalog.

I created the following SQL to CDC from the database and send it to a table in Paimon.

```
USE CATALOG default_catalog;

CREATE CATALOG s3_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 's3://my-test-bucket/paimon',
    's3.access-key' = '',
    's3.secret-key' = ''
);

USE CATALOG s3_catalog;

CREATE DATABASE my_database;

USE my_database;

CREATE TABLE myproducts (
    id INT PRIMARY KEY NOT ENFORCED,
    name VARCHAR,
    price DECIMAL(10, 2)
);

create temporary table products (
    id INT,
    name VARCHAR,
    price DECIMAL(10, 2),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'connection.pool.size' = '10',
    'hostname' = 'mariadb',
    'port' = '3306',
    'username' = 'root',
    'password' = 'rootpassword',
    'database-name' = 'mydatabase',
    'table-name' = 'products'
);

SET 'execution.checkpointing.interval' = '10 s';

INSERT INTO myproducts (id,name) SELECT id, name FROM products;
```

The SQL creates a catalog called "s3_catalog" and inside it creates a database "my_database" and a table "myproducts".

Using Paimon was as easy as creating the catalog with suitable s3 credentials and then creating and querying tables as normal to populate data in s3:

```
CREATE CATALOG s3_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 's3://my-test-bucket/paimon',
    's3.access-key' = '',
    's3.secret-key' = ''
);
```

I submitted the job to Flink:

```
docker exec -it jobmanager /opt/flink/bin/sql-client.sh embedded -f job.sql
```

With the Job up and running, I checked on s3 using the AWS CLI to list the contents of my s3 test bucket:

```
aws s3 ls my-test-bucket/paimon/my_database.db/myproducts/
```

The job had created the following folders in the bucket:

```
PRE bucket-0/
PRE manifest/
PRE schema/
PRE snapshot
```

The schema it stored for the products table on s3 was in JSON format:

```
{
  "id" : 0,
  "fields" : [ {
    "id" : 0,
    "name" : "id",
    "type" : "INT NOT NULL"
  }, {
    "id" : 1,
    "name" : "name",
    "type" : "STRING"
  }, {
    "id" : 2,
    "name" : "price",
    "type" : "DECIMAL(10, 2)"
  } ],
  "highestFieldId" : 2,
  "partitionKeys" : [ ],
  "primaryKeys" : [ "id" ],
  "options" : { },
  "timeMillis" : 1696694538055
}
```

And in a folder called `bucket-0` there was the data from my test database, in ORC format which stands for Optimized Row Columnar.

```
2023-11-05 21:11:27       1279 data-19c71b4d-91c2-45fa-b9f2-b7403e2269e4-0.orc
2023-11-05 20:46:56       1279 data-dec5ca81-ad69-4619-9180-99267f6c60f5-0.orc
```

ORC is comparable to the Parquet file format and AWS have a quick comparison here on their respective strengths: [https://docs.aws.amazon.com/athena/latest/ug/columnar-storage.html](https://docs.aws.amazon.com/athena/latest/ug/columnar-storage.html)

In the end it was quick and easy to get Paimon running to help Flink send data to s3 in a structured format.

However when I first tried this a few days ago I didn't save my data and when I went back t o try it again, I couldn't for the life of me get it to write data to s3 again.

After trying different Jars for Paimon and s3, I even submitted an [issue to the Paimon Github repo](https://github.com/apache/incubator-paimon/issues/2263). Only to close it a few minutes later after I re-read the docs and found the all important line, with a comment showing how important it is:

![Paimon Checkpoint Interval]({{ site.url }}/images/paimon_checkpoint_interval.png)

Once I added that, data was writing to the s3 bucket again.

All I need to do now it to make sure I can read the ORC data using Flink. Hopefully Flink can pull long term data back in from s3 quickly and easily for longer term type queries rather than keeping it all in Kafka.

The files used for this are on Github at [https://github.com/gordonmurray/apache_flink_and_paimon](https://github.com/gordonmurray/apache_flink_and_paimon)