---
layout: post
title:  "Use AWS Redshift for fast count queries"
date:   2022-06-25 17:00
categories: aws
tags: aws rds dms redshift terraform
---

I work with a database that stores Sendgrid event information. [Sendgrid](https://sendgrid.com/) is a popular service for sending emails and for each email sent to a recipient, Sendgrid provides a webhook facility to send you feedback on each email. It doesn't contain the email content, just some high level infromation including an event field showing if an email was processed, delivered, dropped or labelled as spam by the recipient. This is important to monitor to investiage any emails that have been labelled as spam to follow up on and try to help prevent those in the future.

This particular database has grown quite large, over 1 billion records. Each sent email will provide at least 2 events back, one for Processed and one for Delivered.

The current database is on AWS, an r6g.large MariaDB MultiAZ RDS instance with 732gb of storage. It has 2 vCPUs and 16gb of memory. It has over 1 billion records and growing. It currently costs $531.22/month and will increase as storage grows.

I have a background process to query the database a few times a day to perform a count of the events and pops a summary message in to our Chat applicaton like the following:

![Simple Sendgrid Summary]({{ site.url }}/images/sendgrid_summary.png)

The query performed to gather this information is straight forward, to count records with an event type of 'spamreport', 'dropped', 'deferred', 'bounce' or 'delivered' in a recent time period, for example:

> select count(*) from sendgrid.sendgrid_events where event = 'spamreport' and timestamp >= '2022-06-23'


This was fine for the first few million records, with an appropriate index on the event field. However, with over a billion records the queries took longer and longer to return and have reached a point where they that too long to complete to be useful. They cause the RDS CPU and Read IOPS to rise and burst balance to drop, this can interfere with new data going in.

![RDS CPU from running a single query]({{ site.url }}/images/rds_cpu_from_a_query.png)

I tried upgrading the RDS instance temporarilty, doubling the CPU and memory with the same result. I'm interested to see what other database systems might be able to perform the same counts more effeciently instead of trying to break up the data in to separate tables per year or similar manual sharding.

Redshift, SingleStore, Apache Druid and Apache Pinot are on my list to try out for this and other use cases. Since AWS Redshift is on my doorstep so to speak as I use AWS, I am trying Redshift first.

I used AWS DMS to copy all existing data from the RDS instance in to an existing Redshift instance, a dc2.large ( the smallest Redshift instance available. ) It took just over 8 hours using default settings to copy across the 1 billion records. I learned along the way that the disk space on a Redshift instance can't be scaled independetly and that more nodes are needed to increas space which is a shame.

I used Terraform to create the DMS replication instance, DMS source and target endpoints and the task to copy the data to Redshift. I added the Terraform to a repo here in case its useful in the future.

https://github.com/gordonmurray/terraform_dms_redshift

Copying the data took just over 8 hours using default settings. Once the data was copied over, I ran a couple of queries to compare the MariaDB instance and Redshift instance:

### SQL query on RDS:

Counting records with an event of 'delivered' in the last few days

```
select SQL_NO_CACHE count(*) from sendgrid.sendgrid_events where event = 'delivered' and timestamp >= '2022-06-22';
```

After leaving it for 2 hours and still not finished I cancelled it.

### SQL query on Redshift:

Same query, counting emails delivered in the last few days. Redshift doesn't support indexes and AWS DMS created a Key on the id field itself.

```
select  count(*) from sendgrid.sendgrid_events where event = 'delivered' and timestamp >= '2022-06-22'
[2022-06-25 13:07:24] 1 row retrieved starting from 1 in 283 ms (execution: 238 ms, fetching: 45 ms)
```

Not even 1 second to perform the query, giving a count of 2,421,872

So Redshift due to its columnar storage is considerably faster for this particular use case with no tuning at all.

On top of that, Redshift is cheaper than the RDS instance in this particular scenario, partly due to storage costs. The index data alone is over 100gb on the RDS instance.


| Region | Description | Service  | Monthly | First 12 months total | Currency | Configuration summary |
|----------|------------|----------|-----------|--------|--------|--------|
| EU (Ireland) | 2 node Redshift in EU | Amazon Redshift | 441.84 | 5302.08 | USD | "Nodes ( 2 instances of type dc2.large  OnDemand )
| EU (Ireland) | RDS instance in EU | Amazon RDS for MariaDB | 531.216 | 6374.59 | USD | "Storage volume (General Purpose SSD (gp2))

Costs visible here on the [AWS Calculator](https://calculator.aws/#/estimate?id=54a430e10fdcc8c5cc432934bb9b1279e6903e3a)

It may be hard for Singlestore, Druid or Pinot to beat that speed when I get to try those for this and similar projects!
