---
layout: post
title:  "NoSQL Overview: Background, CAP/BASE, and Representative Data Models"
date:   2025-08-31 18:00:00 +0900
categories: nosql
---

*This post is a translated version of [the blog post]({% post_url 2025-08-31-nosql-basic-concepts-and-theory-ko %}) originally written in Korean.*

*This article is based on my seminar "NoSQL Basic Concepts and Theory" presented on August 31, 2025.*

---

## Definition and Background of NoSQL

**NoSQL** is an abbreviation for 'Not only SQL' and represents a **movement** toward using database systems based on data models other than relational databases (RDBMS). NoSQL emerged to overcome the limitations of RDBMS's strict schema and **ACID (Atomicity, Consistency, Isolation, Durability)** transaction model.

The volume and velocity of data generated from web, mobile, and Internet of Things (IoT) devices are explosively increasing, and the heterogeneity of this generated data is also growing dramatically. The characteristics of this data can be summarized as **big data's 3Vs**.

- **Volume**: The amount of data to be processed has increased tremendously.
- **Velocity**: The speed at which data is generated and consumed has become very fast.
- **Variety**: Data types have diversified to include not only structured data but also semi-structured and unstructured data.

Traditional RDBMS faces several limitations in handling these 3V characteristics.

- **Volume & Velocity**: Traditional RDBMS incurs overhead in distributed processing due to its strict ACID transaction model. This makes horizontal scaling difficult and reveals limitations in processing rapidly generated large volumes of data.
- **Variety**: RDBMS requires fixed schemas, making it difficult to store unstructured or semi-structured data with inconsistent formats.

## General Characteristics of NoSQL

NoSQL systems have several common characteristics that distinguish them from RDBMS.

- **Flexible Schema**: Data can be stored without strict schemas, making it easy to manage various types of data.
- **Aggregation Oriented**: Unlike RDBMS normalization, related data is grouped and stored together to simplify data access. This means managing data in denormalized form.
- **Weak Consistency Model**: Unlike RDBMS's **ACID**, it follows a consistency model called **BASE**. This partially relaxes strong consistency to achieve high availability and low latency.
- **Distributed Processing Friendly**: NoSQL is designed and optimized for distributed environments from the beginning, making it much more efficient at distributing and processing data across multiple servers. This enables easy horizontal scaling in large-scale environments.

## NoSQL Theories: CAP and BASE

To understand NoSQL's weak consistency model, we must first examine **CAP theorem**. CAP was first proposed by Eric Brewer in 2000 as **CAP conjecture** and was subsequently proven by Seth Gilbert and Nancy Lynch in 2002. This theory explains that distributed systems can simultaneously satisfy at most two of three properties: **Consistency**, **Availability**, and **Partition tolerance**.

- **Consistency**: Ensures that all clients can access the same latest data regardless of which node they access.
- **Availability**: Ensures that all nodes in the system can always respond to requests.
- **Partition Tolerance**: Ensures that the system operates normally even in failure situations like network partitions (communication disconnection between nodes).

NoSQL generally tends to choose **Availability** and **Partition Tolerance** while partially sacrificing **Consistency**. The **BASE** model adopted by many NoSQL systems is the result of this availability-focused approach.

- **Basically Available**: Ensures that all nodes in the system can process data. The system continues to operate even when node failures occur.
- **Soft State**: The state of data may not be immediately consistent even after transaction completion, which includes the meaning that the system's state can change even without input. The final state of data is determined only after some time passes.
- **Eventually Consistent**: Once data updates are propagated to all nodes, all nodes will eventually have the same data. This consistency is not immediate and takes time.

## Major NoSQL Data Stores

NoSQL is roughly classified into four types based on the form of data they store.
This section explains the characteristics and representative examples of each data store.

### Key-Value Stores

Key-value stores model data as a pair of unique **key** and their associated **value**.
This is the simplest form of data model, providing very fast read and write performance.
There are no constraints on the form of values, so various data types like strings, JSON objects, images, and files can be stored.
Due to this simplicity, it's widely used in systems like **Redis, RocksDB, and AWS DynamoDB**.
Most key-value stores internally use **LSM-Tree (Log-Structured Merge-Tree)** to optimize write performance.

### Wide-Column Stores

Wide-column stores organize data like the table structures RDBMS uses, but allow adding columns dynamically without predefining them in a schema.
That is, each row can have different sets of columns.
This is advantageous for managing sparse data.
**Apache Cassandra and HBase** are representative examples, suitable for environments where various columns are added per row, such as large-scale log data or time-series data.
Most wide-column stores use a key-value store as a storage engine.

### Document Stores

Document stores model data in **document** form.
A document typically follows semi-structured data formats like JSON.
Values within documents can contain nested documents, creating a **tree** structure that has hierarchical organization.
Document stores require no fixed schema, allowing data with different structures to be stored in the same collection, making them much more flexible than RDBMS's fixed table structures. 
**MongoDB and Couchbase** are widely used document stores. 
MongoDB internally uses the **WiredTiger** storage engine. This engine efficiently manages documents using **BSON**, a binary format similar to JSON, when storing data.

### Graph Stores

Graph stores are based on the graph model where **vertices** represent data entities, and **edges** define relationships between vertices. 
This structure excels at efficiently traversing complex relationships and is utilized in social networks, recommendation systems, and knowledge graphs. 
**Neo4j** is a representative example. 
Neo4j uses storage called **index-free adjacency** that directly connects data nodes and relationships with pointers to ensure fast traversal speeds.

## Conclusion

This article provided an overview of NoSQL's definition, background, general characteristics, and data models.
Starting from the next article, we plan to discuss the **storage engines** of each model's data stores in greater depth.