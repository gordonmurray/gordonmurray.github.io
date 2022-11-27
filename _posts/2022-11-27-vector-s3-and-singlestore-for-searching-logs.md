---
layout: post
title:  "Vector.dv, AWS S3 and SingleStore for fast log search"
date:   2022-11-27 19:00
categories: aws
tags: ["aws", "s3", "vector.dev", "singlestore", "terraform"]
---

I wanted to explore the possibility of using AWS S3 as a back end for a logging solution. Not only the storage of logs which s3 handles very well but also to have a way for applications to send logs to s3 and to search those logs too, without any significant delay or cost.

S3 is ideal for log storage. It provides high availability and as much storage as you're willing to pay for. It also provides ways to mange those costs too with different storage tiers and Lifecycle's to archive or remove older files.

The next step is to find a process to send logs to s3 that could be suitable for any programming language. AWS API Gateway is one option. With AWS API Gateway an API endpoint could be set up that could receive data via a POST request. A lambda function could then be triggered to receive the data and store it to s3.

I worked on this for a while and created the resources using Terraform, available on Github at [https://github.com/gordonmurray/terraform_aws_api_gateway_proxy](https://github.com/gordonmurray/terraform_aws_api_gateway_proxy) While this process works and would be ok for a new web application to use, an existing web application would need to be updated to post its logs to the endpoint instead of whatever it used currently. The Lambda cost could be significant too depending on the volume of logs coming in.

One evening I found out about [vector.dev](https://vector.dev) via a comment on Reddit. 

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Tried out <a href="https://t.co/jRl2jbX8Ks">https://t.co/jRl2jbX8Ks</a> yesterday. It can send logs and metrics directly to s3 and other destinations. Could be a great alternative to potentially expensive saas based logging tools. Nice and easy to set up. Created by Datadog.</p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1581594725521334277?ref_src=twsrc%5Etfw">October 16, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

Vector.dev is a logging and metrics agent written in Rust that can collect local metrics and logs and send them to a destination, including s3. I tried it one evening and it was quick and easy to set up. It wouldn't require any changes to an existing app as it could read existing logs that an app produced.

The next part to solve was a way to search logs on s3. I was already away of SingleStore, a high performance in-memory, mysql compatible database. I was aware of a Pipelines feature that allowed data to be sucked in SingleStore so this looked like a good solution to try.

There was a SingleStore Hackathon coming up at the time and that seemed like good motivation to try out this vector.dev + s3 + SingleStore Pipeline idea

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">I wonder if a combination of logs + <a href="https://t.co/jRl2jbG5Is">https://t.co/jRl2jbG5Is</a> + s3 + Singlestore Pipelines + a UI for fast log analysis would be a good entry for the SingleStore Hackathon <a href="https://t.co/RiW1XcLWrA">https://t.co/RiW1XcLWrA</a></p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1581679366991667205?ref_src=twsrc%5Etfw">October 16, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I spent some time learning and creating Pipelines in SingleStore. I needed a SingleStore instance to work with. [I wrote a quick post on creating a single node cluster in the past](https://gordonmurray.com/aws/2022/03/12/single-node-singlestore-cluster-on-aws.html) but I wanted to create it fully using Packer and Terraform in the hopes of creating everything from s3 to SingleStore in one go. I worked on it and the resulting Terraform is available on Github here https://github.com/gordonmurray/terraform_aws_singlestore. SingleStore also provide a hosted solution, but I'll take any excuse to write some Terraform.

With a SingleStore instance in place, I needed some logs. I created a simple nginx web server and shared a link on Twitter to get some hits and generate some access logs.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Testing something http://54.229.14.41/</p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1585386073844514816?ref_src=twsrc%5Etfw">October 26, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I used Vector to monitor the nginx access logs and send them to an s3 bucket. I used the default access logs from nginx and worked on creating a Pipeline in SingleStore to pull in the data.

The vector.dev config is straight forward:

```
data_dir = "/var/lib/vector"

[api]
enabled = true

[sources.nginx]
type = "file"
include = [ "/var/log/nginx/access.log" ]
read_from = "beginning"

[sinks.s3]
type = "aws_s3"
inputs = [ "nginx" ]
bucket = "xxxxxxxxxx"
key_prefix = "nginx/"
compression = "none"
region = "eu-west-1"
batch.timeout_secs = 60 # default is 300 seconds

[sinks.s3.encoding]
  codec = "text"
```

With some trial and error, I had a working pipeline that was ingesting the logs from s3. I needed to create a suitable table for the logs, then create the Pipeline to populate the table. I kept the table simple, with all varchars. Not the most efficient table structure but it was enough to get a working pipeline.

The table I created for nginx logs was:

```
CREATE TABLE `nginx` (
  `agent` varchar(200) DEFAULT NULL,
  `client` varchar(100) DEFAULT NULL,
  `file` varchar(200) DEFAULT NULL,
  `host` varchar(100) DEFAULT NULL,
  `message` text DEFAULT NULL,
  `method` varchar(100) DEFAULT NULL,
  `path` varchar(100) DEFAULT NULL,
  `protocol` varchar(100) DEFAULT NULL,
  `request` varchar(100) DEFAULT NULL,
  `size` varchar(100) DEFAULT NULL,
  `source_type` varchar(100) DEFAULT NULL,
  `status` varchar(100) DEFAULT NULL,
  `timestamp` varchar(100) DEFAULT NULL
);
```

The pipeline I created was:

```
CREATE OR REPLACE PIPELINE nginx_pipeline AS
LOAD DATA S3 'xxxxxxx/nginx/'
CREDENTIALS '{"aws_access_key_id": "xxxxxxxxxxxxxx", "aws_secret_access_key": "xxxxxxxxxxxx"}'
SKIP DUPLICATE KEY ERRORS
INTO TABLE nginx(agent <- agent, client <- client, file <- file, host <- host, message <- message, method <- method,path <- path,protocol <- protocol,request <- request,size <- size,source_Type <- source_type,status <- status,timestamp <- timestamp) format JSON;
```

One a Pipeline was created, it needed ot be started so it would pull in data:

```
START PIPELINE nginx_pipeline;
```

AT first, I was getting plenty errors in my pipeline tests, the following query was useful to list pipeline errors. Usually my errors were related to an incorrect number of columns in the incoming data, due to the parsing of the logs.

```
SELECT DATABASE_NAME, PIPELINE_NAME, BATCH_ID, PARTITION, BATCH_SOURCE_PARTITION_ID,
ERROR_KIND, ERROR_CODE, ERROR_MESSAGE, LOAD_DATA_LINE_NUMBER, LOAD_DATA_LINE
FROM information_schema.PIPELINES_ERRORS;
```

Got it going:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Took me a while fiddling with <a href="https://t.co/jRl2jbG5Is">https://t.co/jRl2jbG5Is</a>&#39;s toml file, but nice to have an ongoing pipeline of logs from s3 directly in to SingleStore for super fast searching ⚡️ <a href="https://t.co/lUlmsTjumt">pic.twitter.com/lUlmsTjumt</a></p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1585389373255979009?ref_src=twsrc%5Etfw">October 26, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I don't have a lot of logs in this test but the pipeline still took some time to import 1000 to 2000 log entries from a few dozen files on s3.

I'm not aware of any way to speed that up unfortunately. It doesn't sound ideal if I was trying to import months of logs 

On the other hand, with the pipeline running it collects any new log files that continue to land in s3. 

I had high hopes for this process. I had a working title of this combination of vector.dev, s3 and SingleStore. I called it 'SingleLog' and even registered the domain single-log.com.

I ran in to difficulty with the Pipelines however as the logs were not consistent. It was related inconsistent fields from different browsers or bots that didn't specify a user agent, or some spammy text that messed up the importing.

It was no fault of SingleStore, I needed to be explicit when creating the table and the Pipeline so SingleStore knew what to expect to store it successfully.

I considered having a table with just 2 columns, an Id field and a large text field that I could store a json array of the entire log line and them maybe use a SingleStore User Defined Function (UDF) later to split up the json and populate other fields. I didn't take that route however, as that would mean I would have to create several different UDFs for different log entry formats and it sounded like a never ending battle to take on.

I spent time playing around with vector.devs settings to try to structure the logs that it was processing as well as customizing the logs that nginx produced in the first place. 

I brought it all together in a repo using Packer and Terraform. This produces a single node SingleStore cluster, an s3 bucket and a nginx webserver to generate some logs when accessing the server [https://github.com/gordonmurray/singlelog](https://github.com/gordonmurray/singlelog)

The domain I purchased early on redirects to the Github repo now. [http://single-log.com](http://single-log.com). 

Even though this mini project to make a cost effective logging solution backed by s3 didn't really work out the way I hoped, I learned a lot in the process. 

While looking different options when storing logs on s3, I was aware of file formats like CSV and JSON and only barely aware of a Parquet format. Parquet format is attractive as it is a binary format that will result in a smaller file size than CSV, it wil cost less on s3 since the files are smaller and searching parquet is faster too from what I read.

I found a really great project called Benthos to search Parquet files on s3. Unfortunately for my needs it only performs batch jobs. When performing a search it returns the results and then exits. It doesn't perform an ongoing consumption of log files like vector for example

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">While searching for logging tools that support logging to s3 in parquet format I came across <a href="https://t.co/BCOlAG12NC">https://t.co/BCOlAG12NC</a>. &quot;Fancy stream processing made operationally mundane&quot;. Unfortunately for my needs, it runs and finishes and doesn&#39;t run as an agent like Vector or Fluent</p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1591871584469676038?ref_src=twsrc%5Etfw">November 13, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I also found an Apache project called Drill that can search semi structured data directly from s3, with no need for an import process like a pipeline

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Yet another handy Apache project I wasn&#39;t aware of. &quot;Apache Drill&quot;. Found it while reading up on the Parquet format. Query almost anything. Nice and easy to query nginx logs as if a mysql table <a href="https://t.co/r7Z4lqDaIL">pic.twitter.com/r7Z4lqDaIL</a></p>&mdash; Gordon Murray (@gortron) <a href="https://twitter.com/gortron/status/1590447741075062785?ref_src=twsrc%5Etfw">November 9, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

I created a project again using Packer and Terraform to create a small Drill cluster as I continue to learn what Drill can do [https://github.com/gordonmurray/terraform_aws_apache_drill_zookeeper](https://github.com/gordonmurray/terraform_aws_apache_drill_zookeeper)

My plans to make a super fast, low cost logging solution are on hold for now. I think the next best step is to look in to ways to have logs stored in parquet format ( or better ) from the outset rather than converting using something like Lambdas as that will add some latency and cost to the overall process.

There is an [https://github.com/vectordotdev/vector/issues/1374](open PR with Vector.dev to support the Parquet format) which could be excellent if it goes ahead. 