---
layout: post
title:  "Misusing Catalogs in Apache Flink for identifying Jobs"
date:   2023-10-28 15:00
categories: data
tags: ["apache","flink","catalogs"]
---

When jobs are created in Flink using SQL, they show up in the jobs list with default names such as `insert-into_default_catalog.default_database.sink_name`. If you're pulling records from multiple sources and sinking them to the same place such as a Redis cache it can be hard to tell which one is which if a job needs some attention. As far as I can tell you can only provide names when submitting jobs via Java.

![Flink jobs with default naming]({{ site.url }}/images/flink_jobs_default_naming.png)

I was looking in to Catalogs to see what they could do. I wanted to store CDC data somewhere to avoid re-snaphotting data from source databases if I could or to share data between jobs. When I created a couple of new jobs using a Catalog I noticed the jobs had different naming.

![Flink jobs using a catalog]({{ site.url }}/images/flink_jobs_named_using_a_catalog.png)

I know this isn't a proper use of Catalogs in Flink. Catalogs can do more than just helping with the naming of jobs though for me it's definitely effective to label dozens of jobs sinking to a central place.

I used the following to create a Catalog and a database and use them when creating any tables. Its an in-memory Catalog so it isn't helping with my initial hope of storing the CDC data to avoid snapshots unfortunately though it gives me some useful naming at a glance.

Theres a Docker Compose file and related files in Github here to reacreate this : [https://github.com/gordonmurray/apache_flink_catalog_misuse](https://github.com/gordonmurray/apache_flink_catalog_misuse)

```sql
USE CATALOG default_catalog;

CREATE CATALOG myproject WITH ('type'='generic_in_memory');

USE CATALOG myproject;

CREATE DATABASE mydatabase;

USE mydatabase;

CREATE TABLE []..]

```

I've been using Flink with Kafka since to help take the pressure off the databases which works well.

For a proper use of Catalogs, I tried out Apache Paimon briefly for storing data on S3 in ORC format and plan to revisit it again soon. Theres definitely more to learn about Catalogs.

[https://github.com/gordonmurray/apache_flink_and_paimon](https://github.com/gordonmurray/apache_flink_and_paimon)