---
layout: post
title:  "In-Memory Columnar Storage and SQL Query Engine: Technical Analysis of Apache Arrow and DataFusion"
date:   2025-04-26 18:00:00 +0900
categories: dbms
---

*This post is a translated version of [the blog post]({% post_url 2025-04-26-apache-arrow-and-datafusion-ko %}) originally written in Korean.
This article is based on my seminar "Databend Seminar: Technical Features of Apache Arrow and DataFusion and their Relationship with Database Systems" presented on April 26, 2025.*

---

This article examines the characteristics of Apache Arrow and Apache DataFusion, discussing them in comparison with Database Management Systems (DBMS).

## Database Management Systems

First, let's briefly overview what a Database Management System is and what components it contains. A Database Management System is also called a database system or simply a database. Here, I'll use the abbreviation DBMS.

![DBMS architecture](/assets/images/2025-04-26-apache-arrow-and-datafusion/dbms-arch.png){: width="300" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DBMS Architecture Overview ([source](https://www.oreilly.com/library/view/database-internals/9781492040330/ch01.html))* 
{: refdef}

The above diagram shows a typical DBMS architecture.
Transport layer, Query Processor, Execution Engine, and Storage Engine are commonly referred to as a single DBMS or instance.

The topmost Transport layer handles communication with client programs (e.g., `psql`, `mysql` command programs) and other DBMS instances.
When a user inputs a query through a client program, that query is passed to the DBMS's Query Processor. In the case of Relational Database Management Systems (RDBMS), queries are typically expressed using Structured Query Language (SQL), which users use to make query requests. Since this is a human readable language, the Query Parser converts this SQL into a form that computers can understand.

The **Query Parser** takes an SQL string as input and generates a **logical plan** that computers can understand. This is typically represented as a tree containing relational algebra operations. It includes logical operators such as Project (corresponding to `SELECT` clauses), Select (corresponding to `WHERE` clauses), Join, and Sort. Think of it roughly like storing arithmetic expressions as trees in data structures class, but for SQL.

The **Query Optimizer** takes this logical plan as input and generates an **optimized physical plan** (also called an execution plan). In this process, it optimizes the logical plan or physical plan to achieve better performance. The physical plan contains information about how to execute the logical plan. For example, while the logical plan contains information like "perform a Join," the physical plan contains information like "process the Join using Hash Join."

The physical plan is passed to and processed by the Execution Engine (in the diagram). **The Execution Engine processes data according to the data formats provided by the Storage Engine.** For example, if using a storage engine that manages data with a column-oriented layout, the internal data format will also be managed as column-oriented data structures (e.g., for an `int` type column, data would be managed in a form like `vector<int> columnChunk`).

In this section, we briefly explored the roles of Query Processor, Execution Engine, and Storage Engine. Apache Arrow corresponds to the data management format of the Storage Engine, while Apache DataFusion corresponds to the Query Processor and Execution Engine. Let's examine the details ahead.

## Apache Arrow

Apache Arrow is a **columnar format** and **multi-language toolbox** for **fast data exchange** and **in-memory analytics**. Many users utilize it as a bridge between different processes or languages, or as a tool for fast data analysis.

This section will focus on Apache Arrow's ability to **1) support multi-language and fast data exchange**, and **2) process data quickly using columnar in-memory format**.

### Multi-language Support and Fast Data Exchange

Apache Arrow can move data extremely fast through **Zero-copy data exchange**. Zero-copy data exchange means no data copying occurs during data exchange. When we want to move data from one process to another, there are several convenient methods available. We can write data to a file and read it from another process, or open a communication channel to transfer data. However, these methods involve moving data from one process's memory to disk or network, then reading it back into another process's memory. In other words, data is copied from one process's memory to another process's memory. While this isn't problematic for small data sizes, as data sizes grow in large-scale data analysis, the time spent on data copying and memory usage can ultimately lead to performance degradation.

![Shared Memory](/assets/images/2025-04-26-apache-arrow-and-datafusion/shared-mem.webp){: width="500" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*Shared Memory ([source](https://medium.com/@rohitkumar_55648/linux-shared-memory-a01c6a8121e))* 
{: refdef}

Fortunately, this can be solved using **Shared Memory**, which we learned about in operating systems class. As shown in the diagram above, shared memory allows multiple processes to access a single memory space simultaneously. One process can write data to the memory space while another process can access that memory space to read the data. This enables information exchange without data copying.

Apache Arrow realizes zero-copy data exchange through this shared memory functionality. Apache Arrow allocates shared memory and sets up its data structures and states there, with all subsequent data managed within this shared memory. Later, when data exchange with other processes is needed, we simply need to set up access to the shared memory for those processes. Of course, for access from different processes (different languages or systems), functionality to interpret the data structures and states managed in shared memory is needed. Apache Arrow provides this as well, which seems to be why they use the term **multi-language toolbox**.

For data exchange between different computers, data must be transmitted to other computers, which inevitably involves data copying. Apache Arrow Flight is used for this purpose.

So how does Apache Arrow manage user data in memory? Let's explore this in the next section.

### Columnar In-Memory Format

Apache Arrow is useful not only for data exchange but also as a fast in-memory analytics tool. For example, it's frequently used to improve data analysis task performance in libraries like Pandas and Polars. So how can Apache Arrow achieve such performance? The columnar format plays a key role here.

Columnar format is one method of storing table data. Traditional RDBMS managed table data in row-oriented format. For example, consider a table `Student(sid INTEGER, name STRING, age INTEGER)`. In row-oriented format, data is physically stored like `1,John Smith,29$2,Jane Doe,29$3,Bob Johnson,30$4,Alice Wilson,31`, where information for each row is stored in consecutive space (`,` separates columns, `$` separates rows). In contrast, column-oriented format physically stores data like `1,2,3,4$John Smith,Jane Doe,Bob Johnson,Alice Wilson$29,29,30,31`, where information for each column is stored in consecutive space (`,` separates rows, `$` separates columns).

So why use columnar format? Columnar format is used to handle analytical queries well. Analytical queries typically perform aggregations by accessing multiple rows rather than accessing single rows. `SELECT SUM(age) FROM Student` would be a simple example of an analytical query. Let's assume this is processed with row-oriented format (refer to the previous paragraph's example). The DBMS must read the physically stored data, then find and read the last column `age` for each row, then sum these values. Now consider processing with column-oriented format. The DBMS simply needs to find a specific column and read all consecutive `age` values at once. This difference in approach allows more aggressive use of storage's sequential I/O (since consecutive `age` values can be read) and is advantageous from a CPU caching perspective. Ultimately, there are performance benefits.

Furthermore, column format creates favorable conditions for vectorized execution that many modern systems use.
Vectorized execution means processing data for multiple rows at once.
Modern CPUs support Single Instruction Multiple Data (SIMD) operations, which provide even greater performance benefits for vector processing.
Since columnar format already has data that needs to be processed together in consecutive space (`age` column values stored consecutively like `29,29,30,31`), it's in an optimal state for such SIMD operations.
If it were row-based data, at least 4 or more instructions would be needed through loops, but with SIMD operations, one is sufficient.
Moreover, having data stored in consecutive space also provides benefits from a CPU caching perspective.

Apache Arrow maximizes these advantages of columnar format. Apache Arrow manages user data in columnar format in memory and provides fast data processing performance through SIMD operations and efficient implementation.

### Further Reading

- What does Apache Arrow's columnar format specifically look like? [Link](https://arrow.apache.org/docs/format/Columnar.html#physical-memory-layout)
- How does Apache Arrow specifically implement IPC? How does it manage shared memory and what IPC format does it use for data communication? [Link](https://arrow.apache.org/docs/format/Columnar.html#serialization-and-interprocess-communication-ipc)
- Why do Column Stores perform better than Row Stores for analytical queries? [Link](https://www.cs.umd.edu/~abadi/papers/abadi-sigmod08.pdf)
- How can variable-length types (e.g., VARCHAR) or nested objects (e.g., structures) be managed in columnar format? [variable-length](https://arrow.apache.org/docs/format/Columnar.html#variable-size-list-layout), [structured layout](https://arrow.apache.org/docs/format/Columnar.html#struct-layout)

## Apache DataFusion

Apache DataFusion is an **extensible** **query engine** that uses **Apache Arrow as its in-memory format**.
In other words, it's an engine that processes SQL queries based on columnar format, and it's highly extensible.
Many users embed DataFusion into their processes to use it as a SQL or DataFrame engine.

DataFusion proposes that utilizing high-quality open-source query engines will become a future trend when building new DBMS ([source](https://docs.google.com/presentation/d/1D3GDVas-8y0sA4c8EOgdCvEjVND4s2E7I6zfs67Y4j8/edit#slide=id.g22007bd2b6f_0_343)).
Traditionally, each database system has developed its own query engine (for example, systems like MySQL and PostgreSQL each have their own distinct architectures), but this analysis shows that such approaches require significant costs for construction and maintenance. DataFusion proposes that when creating new DBMS, instead of starting from scratch, you should take well-designed systems like DataFusion in modular form.
Then add features or modify code to match the characteristics of the DBMS you want to develop.

Perhaps for this reason, DataFusion is structured very similarly to traditional DBMS.
In this chapter, let's explore how the DataFusion query engine is structured.

### Query Engine Components

![DataFusion architecture](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-arch.jpg){: width="600" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion Overview ([source](https://docs.google.com/presentation/d/1D3GDVas-8y0sA4c8EOgdCvEjVND4s2E7I6zfs67Y4j8/edit#slide=id.p))* 
{: refdef}

The above diagram represents DataFusion's architecture.
The upper left shows Data Sources, which are the ones that the Storage Engine reads and writes.
The lower left shows that users can query DataFusion using SQL and DataFrames.
In the center of the diagram, Plan Representations consisting of LogicalPlans and ExecutionPlan are placed.
These are the same as the logical plans and physical plans discussed in the DBMS discussion.
On the right are Optimized Execution Operators, which refer to the operators that can be executed contained in ExecutionPlan.

As you can see from the diagram, DataFusion does the same work as a DBMS query engine. It processes queries input from FrontEnds to create logical plans, which are converted to better logical plans through transformation or optimization. DataFusion converts the generated plan to an execution plan that represents how to actually execute it. In this process, it performs similar transformation and optimization. Then it executes the execution plan using operators written based on Arrow.

### Query Optimization

![DataFusion Query Optimization](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-qo.png){: width="600" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion Logical Plan Optimization ([source](https://docs.google.com/presentation/d/1ypylM3-w60kVDW7Q6S99AHzvlBgciTdjsAfqNP85K30/edit#slide=id.p))* 
{: refdef}

DataFusion's logical plan optimization consists of multiple stages as shown in the diagram above. When a logical plan is given as input, optimizer pass 1 creates a logical plan with slightly improved performance, which serves as input to optimizer pass 2 to create another logical plan with slightly improved performance, and this process repeats to create the final logical plan. 
Each optimizer pass is governed by built-in rules, which may or may not be reflected in the resulting logical plan depending on whether rule conditions are met or predicted performance of the optimized plan is better.
Each optimizer pass performs the following tasks:

- Pushdown: Moving Projection, Limit, Filter operators toward the beginning of the query plan
- Simplify-1: Minimizing expression evaluation during query execution
- Simplify-2: Removing unnecessary operators
- Flatten Subqueries: Replacing nested queries (subqueries) with joins
- Optimize Joins: Join operation optimization
- Optimize DISTINCT: DISTINCT operation optimization

The execution plan is generated from the final logical plan created through the above process, and this is also optimized through multiple optimizer passes like the logical plan. The optimizations applied in this process include:

- Enforce Sort/Partitioning: Determining if data needs sorting or partitioning
- Pick Algorithm: Deciding algorithms for join and sort operations based on sorting/partitioning status
- Use Statistics: Checking statistical information and replacing Scan when possible

One of DataFusion's characteristics is extensibility. Accordingly, users can directly add query optimization functionality. Specific details will be left in the further reading section.

### Performance Aspects

Apache DataFusion touts fast query execution performance as an advantage. Primarily, asynchronous I/O, vectorized processing, and multi-core processing through partitioning contribute to query performance.

Asynchronous I/O means processing I/O asynchronously. I/O targets like disks or networks operate much slower than CPUs. Therefore, when a CPU requests I/O, the CPU has nothing to do until the request is processed (i.e., until data is read from HDD or successfully transmitted over network). When the CPU waits without doing other work during this time, it's called Synchronous I/O, and when it does other work meanwhile, it's called Asynchronous I/O. DataFusion adopts the Async I/O approach.

Vectorized processing works well with columnar format as explained in the previous chapter.
DataFusion achieves good performance by being based on columnar Arrow.

![DataFusion Data Partitioning](/assets/images/2025-04-26-apache-arrow-and-datafusion/datafusion-partition.png){: width="700" style="display:block; margin-left:auto; margin-right:auto"}

{:refdef: style="text-align: center;"}
*DataFusion Data Partitioning ([source](https://docs.google.com/presentation/d/1cA2WQJ2qg6tx6y4Wf8FH2WVSm9JQ5UgmBWATHdik0hg/edit#slide=id.g209d99697c0_0_11))* 
{: refdef}

In the DBMS context, partitioning means physically grouping data according to specific criteria. DataFusion splits data into partitions, then performs queries using a data-parallel approach (as shown in the diagram above).

This concludes our exploration of DataFusion.
I have added links that may be useful to readers in the further reading section, so please take a look.

### Further Reading

- How does DataFusion call each operator and generate data? What specific query execution model does it use? [Volcano-style query execution model](https://docs.rs/datafusion/latest/datafusion/#execution)
- How does DataFusion specifically optimize queries? [doc](https://datafusion.apache.org/library-user-guide/query-optimizer.html), [code](https://github.com/apache/datafusion/tree/main/datafusion/optimizer)
- How can you add custom query optimization rules? [Link](https://datafusion.apache.org/library-user-guide/query-optimizer.html#writing-optimization-rules)
- How can you add custom functions and aggregations? [Link](https://datafusion.apache.org/python/user-guide/common-operations/udf-and-udfa.html)

## Conclusion

People who research/develop DBMS seem to be very interested in the Flight DataFusion Arrow Parquet stack (abbreviated as FDAP stack) recently, including DataFusion and Arrow. Particularly in using this stack to create new DBMS. A representative example is InfluxDB, which used its own developed architecture up to version 2, but developed a new system with the FDAP stack for version 3 ([source](https://youtu.be/AGS4GNGDK_4?si=61Kom1xSWZAFlmTa)).

So far, we've discussed the technical characteristics of Apache Arrow and DataFusion in the FDAP stack. These cover the Query Engine (Query Processor and Execution Engine) and parts of the Storage Engine in DBMS. In the [next article]({% post_url 2025-05-18-apache-parquet-and-opendal %}), we'll examine Parquet, which is closely related to the lower-level Storage Engine. We'll also look at OpenDAL, a recent layer that can read various data formats through a unified interface.