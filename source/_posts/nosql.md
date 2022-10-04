---
title: Scalable SQL and NoSQL Data Stores
date: 2022-10-04 21:45:24
index_img: /img/cover/nosql.png
banner_img: /img/bg/seashore.jpeg
tags: [Distributed Storage, NoSQL, RDMS]
---

Motivated by **Web 2.0**, the application was designed to support thousands of millions of users write and read concurrently. In contrast to traditional DBMS, NoSQL accepts BASE when designing, which means that they sacrifice some of the dimensions like transaction or strong consistency in order to maintain high availability or scalability.

Generally, there are several key features of NoSQL:

- Horizontally Scale.

- Fault Tolerant and increasing throughput by partition and replication.

- A weaker consistent model compared to ACID.

- Efficient use of RAM and distributed index.

Among the various NoSQL products, Google Bigtable, Amazon Dynamo and Memcached provide a "proof of concept" that inspired many deriving data stores.

- **BigTable** pioneered the idea of partitioning persistent data into thousands of nodes. 

- **Memcached** demonstrated that in-memory indexes can be highly scalable, distributing and replicating objects over multiple nodes.

- **Dynamo** is the first one to design a DB with eventually consistency and increase the service scalability and availability.

NoSQL has a key feature called **shared-nothing** horizontal scaling by replicating data over many servers. But now NewSQL (e.g Amazon Aurora [Amazon Aurora: Design for HighThroughput Cloud-Native Relational Databases.](https://lo845xqmx7.feishu.cn/docx/doxcnLtzOhq6AwnL8BIbNdc5pGb)) has started the trend of shared-storage architecture where server nodes don't store data locally but in Distributed File System.

## Category

**Key-Value Store**: These systems store values and an index to find them, based on a programmer defined Key. (Redis, Memcached, Voldemort.)

**Document Store**: The documents are **indexed** and a simple query mechanism is provided. (MongoDB, CouchDB)

**Extensible Record Stores**: The systems store extensible records that can be partitioned vertically and horizontally across nodes (Wide-Column-Stores). (BigTable, HBase, Cassandra)

**Relational Database**: (MySQL, PostgreSQL)

## Key Value Stores

Suitable for Query only by the primary key.

|           | Data Model             | Concurrence Control | Consistency            | Partition    | Sync Partition | Logic Clock            |
| --------- | ---------------------- | ------------------- | ---------------------- | ------------ | -------------- | ---------------------- |
| Voldemort | KV Pairs               | MVCC                | Eventually Consistency | Cons Hashing | Async          | Dynamo Vector Clock    |
| Riak      | Json                   | MVCC                | Quorum (R, W, N)       | Cons Hashing | Async          | Dynamo Vector Clock    |
| Redis     | Various Data Structure | Lock                | Eventually Consistency | Cons Hashing | Async          | No need (Leader Based) |
| Scalaris  | Various Data Structure | ACID                | Strong Consistency     | Range        | Sync           |                        |

## Document Stores

Unlike KV Store, the document store provides the ability to query by multiple columns.

|          | Concurrence Control | Consistency            | Partition                              | Sync Partition     | Feature                      |
| -------- | ------------------- | ---------------------- | -------------------------------------- | ------------------ | ---------------------------- |
| SimpleDB | N.A                 | Eventually Consistency | Manually                               | Async              | Does not allowed nested docs |
| CouchDB  | MVCC on single Doc  | ACID on one document   | Manually                               | Async              |                              |
| MongoDB  |                     | Atomic Writes on field | Automatic sharding key defined by user | Master-Slave Async |                              |

## Extensible Records Stores

The extensible record stores have been motivated by Google Bigtable. Their basic data model is rows and columns and their basic scalability model is splitting both rows and columns over multiple nodes.

- Rows are split across nodes through sharding on the primary key. They typically split by range rather than hash so that the range query can be partition pruned.

- Columns of the table are distributed over multiple nodes by using "Column Groups". These are for the relevant columns to be grouped together. The column groups don't have to be stored on the same node, that is kind of vertical partitioning.

### HBase

HBase is a distributed table store system patterned directly after Bigtable.

- HBase has a Log Structured merge file index allowing fast range queries and sorting

- HBase persistent storage uses HDFS instead of GFS, it writes to the RAM immediately and periodically flushes the data to the Storage

- Row operations are atomic with row level locks and there's optional support for transaction within a wide range based on MVCC. If multiple versions have conflict, the following ones will be aborted.

### Cassandra

- Cassandra is similar to HBase in the storage process that immediately writes to RAM and periodically flushes to disk.

- Cassandra has a weaker consistency model where there's no locking mechanism and replicas are updated asynchronously.

- Cassandra uses an ordered hash index which should give most of the benefit of both hash and BTree but the sorting will be slower than BTree.

- Cassandra uses quorum read to provide a way to get the latest data.

## Scalable Relational Database

From MySQL Cluster, several new products have come out, e.g VoltDB and Clustrix. It appears that the RDMS are restricted by two provisions:

- Small Scope Operation: Join over many tables will not scale well with sharding.

- Small transactions: Transactions that span many nodes are going to be very insufficient with the communication of 2PC overhead.

### MySQL Cluster

MySQL Cluster works by replacing the InnoDB layer with NDB layer and the data is sharded over multiple database servers (shared nothing architechture). Every shard s replicated to support recovery.

MySQL Cluster support in-memory storage as well as disk storage.

### VoltDB

VolteDB's scalability and availability features are competitive with MySQL Cluster and NoSQL systems

- Tables are partitioned over multiple servers and clients can call any servers.

- Selected tables can be replicated over servers, e.g for fast access to read-mostly data.

- Fault Tolerance Data can be recovered from peer nodes in the event of one node crashs.

VoltDB eliminates nearly all "waits" in SQL execution, allowing a very efficient implementation:

- The system is designed for a database that fits in the RAM on servers so that the system need never wait for the disk. Indexes and record structures are designed for RAM rather than disk and the overhead of Disk cache/buffer is eliminated as well.