---
layout: post
title:  "Quick and simple steps to get a single node Singlestore cluster running on AWS"
date:   2022-03-12 13:00
categories: aws
tags: aws singlestore
---

Use at least an m5.xlarge ec2 instance for 4 CPUs

> sudo apt update

Get Singlestore server and toolbox

> wget https://release.memsql.com/production/debian/pool/singlestoredb-server7.6.10_a2ab594895_amd64.deb
> wget https://release.memsql.com/production/debian/pool/singlestoredb-toolbox_1.13.4_497e69e4f6_amd64.deb

Install Toolbox (it will install singlestore server too)

> sudo dpkg -i singlestoredb-toolbox_1.13.4_497e69e4f6_amd64.deb

create a cluster.yml file with the following content: ( Add in your licence and EC2 private IP )

```
license: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
high_availability: false
root_password: password
memsql_server_file_path: /home/ubuntu/singlestoredb-server7.6.10_a2ab594895_amd64.deb
hosts:
  - hostname: xxx.xxx.xxx.xxx
    nodes:
      - role: Master
        config:
          port: 3306
      - role: Leaf
        config:
          port: 3307
    localhost: true
```

Deploy the cluster

> sdb-deploy setup-cluster --cluster-file cluster.yml

Install the Singlestore Studio UI

> wget https://release.memsql.com/production/debian/pool/singlestoredb-studio_4.0.3_586732ae33_amd64.deb
> sudo dpkg -i singlestoredb-studio_4.0.3_586732ae33_amd64.deb
> sudo memsql-studio

Clean up

> sdb-deploy destroy-cluster

Troubleshooting

> stderr: Error 2448: BOOTSTRAP AGGREGATOR FORCE failed to connect to itself: Self check failed. Verify that network connections from the node it itself are allowed.

Update cluster.yml to include your instance private IP

> ERROR: Registered hosts detected. SingleStore DB Toolbox supports managing only one cluster per instance. To view them, run 'sdb-toolbox-config list-hosts' To remove them, run 'sdb-toolbox-config unregister-host'

Install Singlestore studio

Add a Leaf ( on a separate EC2 instance )

On the current master: ( Where xxx.xxx.xxx.xxx is the Private IP of the new EC2 instance )

> sdb-toolbox-config register-host --host xxx.xxx.xxx.xxx -i key.pem

> sdb-deploy install --host xxx.xxx.xxx.xxx --version 7.6.10

sdb-admin create-node --host xxx.xxx.xxx.xxx --password {password}

sdb-admin add-leaf --memsql-id <MemSQL_ID> ( output from previous step )

Verify:

sdb-admin list-nodes
