---
layout: post
title:  "Apache Flink using Checkpoints"
date:   2023-10-25 21:00
categories: aws
tags: ["s3", "apache","flink"]
---

When running some Flink Jobs that perform CDC from a relational database, there were times that a Job would restart. The restart could be caused by a source database restarting or hitting the connection limit of the database user, or restarting a Task manager container.

When that happened, the Flink Job would start over. It would re-snapshot from the source database table which wasn’t great. It took time to do and added pressure to the source database as it read the data from the table again.

I tried out [Checkpointing](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/) in some Jobs that looked like it would help.

```
execution.checkpointing.mode: AT_LEAST_ONCE
execution.checkpointing.interval: 60min
execution.checkpointing.timeout: 10min
```

At first the Checkpoints saved data to the disk. It worked well in handling Job restarts, however the checkpoint data added up quickly and filled up the disk on the instance.

I changed to using RocksDB as that allowed incremental checkpoints instead of full checkpoints. This helped to make the checkpoint data smaller but it still added up over time.

```
execution.checkpointing.mode: AT_LEAST_ONCE
execution.checkpointing.interval: 60min
execution.checkpointing.timeout: 10min
state.backend: rocksdb
state.backend.incremental: true
```

I was confused as to why the checkpoint files weren’t being cleaned up when jobs were cancelled. I think the Job manager was responsible for cleaning up the old checkpoints but that it couldn’t connect to the files on the Task Manager containers to clean up the checkpoint folders there. 

I added the settings to store the checkpoint data on s3. That seems to work. I created some jobs and cancelled them after running for a while. The checkpoint data was cleaned up alright.

```
execution.checkpointing.mode: AT_LEAST_ONCE
execution.checkpointing.interval: 60min
execution.checkpointing.timeout: 10min
state.backend: rocksdb
state.backend.incremental: true
state.checkpoints.dir: s3://{{ s3_state_bucket }}/flink/checkpoints/{{ region_short }}/
s3.access.key: {{ s3_key }}
s3.secret.key: {{ s3_secret }}
s3.fs.impl: org.apache.hadoop.fs.s3a.S3AFileSystem
s3.endpoint: s3.amazonaws.com
```

The data mounted up over time on s3 still, though that was due to having a lot of jobs running and each one of them storing frequent checkpoints.  

I created the s3 bucket using Terraform and updated it to include a Lifecycle to delete files in the bucket that were older than a week to keep the size of the bucket under control.

I have moved on a little since trying checkpoints to pulling the data from Kafka instead of from a database. With the data in Kafka, I don’t know if I need to use Checkpoints anymore. For now, I have changed the interval to several hours apart to see how that works out.

The full docker compose for this is available here https://github.com/gordonmurray/apache_flink_using_checkpoint
