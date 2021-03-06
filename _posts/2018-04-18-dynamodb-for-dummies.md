---
layout: post
title:  Dynamo DB for dummies
---

I've been been working with [DynamoDB](https://aws.amazon.com/dynamodb/) for a while now. I want to share some of my findings about it's little quirks with you all as well as summarise my thoughts and learnings about how it works for my own future reference. It's a really great system for storing per user information and in situations where you don't particularly want to query across multiple tables at a time. This is going to be quite a high level overview but I'll try to link more detailed articles as I go so you can research the topics raised here further.

First of all, What *is* DynamoDB? It's a data storage system built and provided by Amazon. It's a [NoSQL](https://en.wikipedia.org/wiki/NoSQL) system which means it doesn't work like conventional database systems you might be familar with such as [Access](https://en.wikipedia.org/wiki/Microsoft_Access), [MySQL](https://www.mysql.com/), [MariaDB](https://mariadb.org/) or [SQL Server](https://en.wikipedia.org/wiki/Microsoft_SQL_Server). It has more in common with [MongoDB](https://www.mongodb.com/).

The first difference is you can't host your own DynamoDB server - you have to set it up via [Amazon Web Services](https://aws.amazon.com/) and access/pay for it via [Amazon](https://en.wikipedia.org/wiki/Amazon_(company)) directly.

The second, larger difference is that it doesn't store data in the same relational manner you may have used before. The only part you have to define about the schema ahead of time is the primary key, which at its most basic can simply be one field called the **Hash or Partition Key**. When you define this each record on your table must have a unique instance of this value and you can only select objects directly via this key, or search the entire table for records by other values on table rows. This is known as a [table scan](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SQLtoNoSQL.ReadData.Scan.html) and can cost you more money than necessary as Amazon [charge for reads, writes and total storage used in DynamoDB.](https://aws.amazon.com/dynamodb/pricing/) It's far more efficient for your wallet if searches are optimised and they will be faster as well.

Once you have defined your hash key, each record in the table can be completely different. Let's imagine I had a table called users that had an integer as the primary key/record identifier. In a normal SQL database I'd also have to define the kind of data I'd want to store for every record - if I wanted to store the user's name and age I'd have to define that ahead of time like so:

<br>

| Id (Key)  | Name          | Age   |
|:---------:|:-------------:|:-----:|
| 1         | John          |    16 |
| 2         | Sarah         |    22 |
| 3         | Bob           |    56 |

<br>

If I wanted to add a new field to one record, say to store a user's favorite color, I'd have to update the table definition. Depending on my database system I may have to give a default value or even update every existing record.

In DynamoDB things work differently. Each record can have completely different data as long as it has a unique key, like so:

<br>

| Id (Partition Key)    | Name          | Age   | Favorite Color    |
|:---------------------:|:-------------:|:-----:|:-----------------:|
| 1                     | John          | 16    | Green             |
| 2                     |               |       | Blue              |
| 3                     | Bob           | 56    |                   |

<br>

The concept of a table is completely different. I can insert a record with completely different fields to those that came before it without updating the table definition in any way as long as I include a unique partition key. This is a powerful feature for storing data in a system that is still growing where you are unsure as to what exactly you will need to store in the future. This kind of loosely defined data template storage is known as a [document store.](https://en.wikipedia.org/wiki/Document-oriented_database)

Dynamo tables can be made to be [eventually consistent](https://stackoverflow.com/questions/10078540/eventual-consistency-in-plain-english), meaning that reads may not match up exactly to the data in the table at the current moment in time but will always "catch up" eventually. You can also request strong consistency which acts more like a tradition database in that any data returned matches all data that has been entered so far. I prefer to demand strong consistency by default but there are many cases where eventually consistency can be advantageous, especially those where you just need to get large amounts of data out as quickly as possible without worrying about individual records.

In SQL, you might be used to searching on any column in any way. You would expect to be able to search on a non primary key such as `Age` and sort by it with something like the following query:

<br>

```sql
SELECT * FROM USERS
WHERE AGE < 30
ORDER BY AGE
```

<br>

This would return the following data:

<br>

| Id (Key)              | Name          | Age   |
|:---------------------:|:-------------:|:-----:|
| 1                     | John          |    16 |
| 2                     | Sarah         |    22 |

<br>

To do this in DynamoDB, I'd have to select every record and then filter them. To get a more performant search I'd need to define a secondary key, known in dynamo as the *Range* or *Sort Key*. If you define one of these, each record has to be unique on the Partition Key AND the Sort Key as shown below:

<br>

| Id (Partition Key)  | Name          | Age (Sort Key)  | Favorite Color    |
|:-------------------:|:-------------:|:---------------:|:-----------------:|
| 1                   | John          | 16              | Green             |
| 3                   |               | 10              |                   |
| 3                   | Sarah         | 56              | Black             |
| 3                   | Smith         | 99              |                   |
| 4                   | Bob           | 56              | Grey              |

<br>

As you can see, we now have records with the same partition key and the same name but each record has a different combination of both. They are also both mandatory if they have been defined. This is known as a *composite primary key* or *hash-range key*. We can now sort/query by the sort key but not by any other values, not even the partition key itself unless we want to do an expensive full table scan.

To understand how this works, it's important to understand how Dynamo stores data under the hood. It stores data in locations per *partition key*, and then orders everything in that partition by the *sort key*. Without the partition key you aren't going to be able to access any data at all. I could however, now search for each item with an `Id` of 3 and an `Age` above 10. This would return the following data:

<br>

| Id (Partition Key)  | Name          | Age (Sort Key)  | Favorite Color    |
|:-------------------:|:-------------:|:---------------:|:-----------------:|
| 3                   | Sarah         | 56              | Black             |
| 3                   | Smith         | 99              |                   |

<br>

Most people however will want to sort and search through their data on more than one field. How can we achieve this? Dynamo provides two options to do do, known as *secondary indexes.* These are similar to what you may have come across as an [index](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html) in a sql database but are slightly different.

The first option is known as a [Local Secondary Index.](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html) This allows you define another sort key that you pair with the same partition key to sort by. You are allowed up to five of these on each table. They always include the *sort key* from the original table as this is how they link back to the original record(s):

<br>

| Id (Partition Key)  | Name (Sort Key)                 | Age (Projected Key)  |
|:-------------------:|:-------------------------------:|:--------------------:|
| 1                   | John                            | 16                   |
| 2                   | Sarah                           | 12                   |
| 3                   | Sarah                           | 56                   |
| 3                   | John                            | 99                   |
| 3                   | John                            | 56                   |

<br>

Now we could now search for records with an `Id` of 3 and the `Name` 'John' which would return the following data:

<br>

| Id (Partition Key)  | Name (Sort Key)                 | Age (Projected Key)  |
|:-------------------:|:-------------------------------:|:--------------------:|
| 3                   | John                            | 99                   |
| 3                   | John                            | 56                   |

<br>

The limitations of the *Local Secondary Index* are that they MUST be defined when the table is created and cannot be deleted afterwards so you have to plan ahead to use them. There is also a size limit per *partition key* of 10 GB so you can't store too much data under one key. A *Local Secondary Index* is updated when the main table is, and consumes any throughput or provisioning limits you have set on that table. You can set a local secondary index to be eventually or strongly consistent.

The other kind of index you can create is a [Global Secondary Index.](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html) They act in most ways like a copy of the original table with a different set of keys. They are charged and provisioned seperately to the original table. You are also allowed 5 of these per table. They always include the *primary key(s)* from the original table as this is how they link back to the original record(s):

<br>

| Id (Projected)                  | Age (Projected)       | Name (Partition)       | Favorite Color (Sort)    |
|:-------------------------------:|:---------------------:|:----------------------:|:------------------------:|
| 1                               | 11                    | John                   | Black                    |
| 2                               | 42                    | Joe                    | Blue                     |
| 3                               | 55                    | Sarah                  | Black                    |
| 3                               | 27                    | John                   | Blue                     |
| 4                               | 73                    | John                   | Grey                     |

<br>

We could now search for all records with the name "John" with a favourite color beginning with 'B' which would return the following data:

<br>

| Id (Projected)                  | Age (Projected)       | Name (Partition)       | Favorite Color (Sort)    |
|:-------------------------------:|:---------------------:|:----------------------:|:------------------------:|
| 1                               | 11                    | John                   | Black                    |
| 3                               | 27                    | John                   | Blue                     |

<br>

The primary benefits of a global secondary index are that you can have a completely different partition key to the original table and you do not have to define a range key if you do not want to. You can define them long after you have created a table and delete them at any time.

The downside to them is that global secondary indexes use more resources than local ones, and cost a bit more (but still far less than full table scans). You can only retrieve the columns defined in the global secondary index and cannot ask for columns from the main table like you can with a local secondary index. They are also **only** eventually consistent, meaning you can not guarantee data you retrieve is completely up to date. [Just Eat have a good article on how to use them effectively.](https://tech.just-eat.com/2014/03/19/using-dynamodb-global-secondary-indexes/)

You can also do a few other other neat tricks in dynamo. You can make records [expire/be deleted after a set amount of time](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) by defining one column to represent the amount of time to wait before expiry. This is really useful if you just want to store data for a few days or if you want to make sure you don't use up too much space in dynamo and incur higher costs. [Optimistic concurrency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.OptimisticLocking.html) is built in - again all you need to do is define a column which holds a version number and dynamo will prevent multiple clients modifying the same record at the same time.

I hope this has helped explain to you the basics of DynamoDB and how to model some of your data! If you would like to learn about it in more detail, I can heartily recommend [Amazon's own documentation.](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.html) If you have any feedback about my explanation please comment below and let me know.
