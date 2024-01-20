---
layout: post
title:  "Excluding sensitive data from Debezium CDC"
date:   2024-01-20 16:00
categories: data
tags: ["debezium", "cdc","exclude","blacklist","pii"]
---

When using Debezium to source data from a relational database like Mariadb, it might be neccessary to block some data from being copied. It might be for privacy reasons like removing personally identifiable information, or it might be for effecienty reasons to remove some verbose fields from being copied.

The main questions I had in mind for this were (a) if I could get it to work and (b) would it exclude the field(s) in both the initial snapshot phase (whereby all existing data in a table is copied) or in the ongoing CDC phase (whereby ongoing changes are copied) that Debezium performs, or both.

The good news is that I got it working well and that a field is excluded in both the snapshot and the ongoing CDC phase, which is great. I was worried it might might only work in one of those phases.

I lost some time down a rabbit hole at the start. I had read somewhere that the field to use in the connector to exlude a field was `column.blacklist` and no matter what values I used in this, I couldn't get it to work. After some Googling I found that it had been renamed to `column.exclude.list`.

I created a project on Github with the files. It uses Docker Compose to create a mariadb instance, populate it with some sample data and then a connector config to CDC from a table in to Kafka, with some fields to be excluded.

The sample data I used is a users table with some generated data:

```
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50),
    surname VARCHAR(50),
    email VARCHAR(100),
    date_of_birth DATE,
    signed_up DATETIME,
    user_type ENUM('user', 'admin')
);

INSERT INTO users (first_name, surname, email, date_of_birth, signed_up, user_type) VALUES
('John', 'Doe', 'john.doe@example.com', '1990-01-01', '2022-01-15 08:30:00', 'user'),
('Jane', 'Smith', 'jane.smith@example.com', '1985-05-20', '2022-02-10 09:45:00', 'admin'),
('Alice', 'Johnson', 'alice.johnson@example.com', '1992-07-11', '2022-03-05 10:00:00', 'user'),
('Bob', 'Brown', 'bob.brown@example.com', '1988-09-30', '2022-04-20 11:20:00', 'user'),
('Charlie', 'Davis', 'charlie.davis@example.com', '1995-11-15', '2022-05-25 13:45:00', 'admin');
```

I then created a connector for Debezium that included the following values to exclude the surname, email and date of birth fields from being copied to kafka.

```
"column.exclude.list": "mydatabase.users.surname, mydatabase.users.email, mydatabase.users.date_of_birth",
```

Once I started the connector, I was able to query the messages in the resulting topic to see if the fields were present.

```
➜  ~ ./kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic testing.mydatabase.users --from-beginning | jq
```

One of the records now looks like the following example. Not much left after the sensitivie fields have been removed, but its working well.

```
  "payload": {
    "id": 5,
    "first_name": "Charlie",
    "signed_up": 1653486300000,
    "user_type": "admin"
  }
```

To test the CDC part, I added and updated a record or two in the source database, to see what the data might look like:

```

INSERT INTO users (first_name, surname, email, date_of_birth, signed_up, user_type) VALUES
('Bobby', 'Tables', 'bobby.tables@example.com', '1995-07-01', '2023-03-15 08:30:00', 'user');

update users set email = 'bobby.tables.new@example.com' where first_name = 'Bobby' and surname = 'Tables';
```

The topic content was still good, no sign of the sensitive fields being carried over:

```
  "payload": {
    "id": 6,
    "first_name": "Bobby",
    "signed_up": 1678869000000,
    "user_type": "user"
  }
```

So it works well. A 1-line change in a source connector can remove any unwanted or sensitive fields, so theres no excuse for sensitive data getting in to your Datalake!

The files are available on [https://github.com/gordonmurray/debezium_exclude_columns](https://github.com/gordonmurray/debezium_exclude_columns)
