---
layout: post
title:  "How to Make a DuckDB Extension for a Table Function?"
date:   2024-04-09 23:00:28 +0900
categories: duckdb extension dbms
---

DuckDB is one of the most popular database management systems, aimed at embedded data analytics. It is not only lightweight but also incredibly fast, making it an appealing choice for developers and data engineers. One way to leverage its capabilities, due to its extendable nature, is by creating an extension to fetch data from other applications.

Unfortunately, there are few resources available on how to make a DuckDB extension, which makes it quite challenging to build one ourselves.

In this post, I will explain how to create a DuckDB extension for a table function, which produces rows for DuckDB. By implementing this, you can generate or read data from external sources for use in DuckDB.

NOTE: This post is based on my analysis of DuckDB source codes and extension templates. It may contain inaccuracies. If you find any, please kindly let me know.

## Table of Contents
- What is a Table Function?
- How to Create an Extension and a Table Function?
- How to Define a Table Function?
- The Flow of Function Calling
- Summary

## What is a Table Function?

According to the DuckDB documentation, a table function is one that can be called from the `FROM` clause of a query. For example, the CSV function `read_csv()` in the `FROM` clause below is a table function that reads a CSV file from the file system.

```SQL
SELECT * FROM read_csv('flights.csv');
```

## How to Create an Extension and a Table Function?

DuckDB's [extension template](https://github.com/duckdb/extension-template) serves as an excellent starting point for understanding extension creation. It is beneficial to consult this alongside the blog post, starting with `/src/quack_extension.cpp`.

To create a DuckDB extension, you need to define an extension class that extends the [`Extension`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/main/extension.hpp) class. This class has two important methods: `Name()` and `Load(DuckDB &)`, each of which returns the name of your extension and  defines what your extension provides, respectively. When you submit a query to load an extension, like `LOAD foobar`, DuckDB calls the `Load(DuckDB &)` function to load the extension.

```cpp
class Extension {
public:
    DUCKDB_API virtual ~Extension();

    DUCKDB_API virtual void Load(DuckDB &db) = 0;
    DUCKDB_API virtual std::string Name() = 0;
};
```
*Code 1. The `Extension` class*

The `Load(DuckDB &)` method is responsible for registering functions to let DuckDB know what functions can be used. You need to define a table function inside this method and then register the function by calling the `ExtensionUtil::RegisterFunction()` method.

```cpp
void ArrayExtension::Load(DuckDB &db) {
    auto table_function = ArrayExtension::GetTableFunction();
    ExtensionUtil::RegisterFunction(*db.instance, table_function);
}
```
*Code 2. An example of the `Load(DuckDB &)` function*

Please note that if you plan to distribute your extension, you need to define C functions such as `DUCKDB_EXTENSION_API void EXTENSIONNAME_init(duckdb::DatabaseInstance &db)` and `DUCKDB_EXTENSION_API const char *EXTENSIONNAME_version()`. These functions are invoked when you attempt to load an extension from the DuckDB shell. For more information, please refer to [the extension template](https://github.com/duckdb/extension-template/blob/main/src/quack_extension.cpp#L56).

To define a table function, you need to create several functions and then add them to your table function definition. The snippet below shows how DuckDB defines the `read_csv()` function. `GetFunction()` initializes a table function by setting up some options before returning the function. Let's examine these steps in the next section.

```cpp
TableFunction ReadCSVTableFunction::GetFunction() {
    TableFunction read_csv("read_csv", {LogicalType::VARCHAR}, ReadCSVFunction, ReadCSVBind, ReadCSVInitGlobal, ReadCSVInitLocal);
    read_csv.table_scan_progress = CSVReaderProgress;
    read_csv.pushdown_complex_filter = CSVComplexFilterPushdown;
    read_csv.serialize = CSVReaderSerialize;
    read_csv.deserialize = CSVReaderDeserialize;
    read_csv.get_batch_index = CSVReaderGetBatchIndex;
    read_csv.cardinality = CSVReaderCardinality;
    read_csv.projection_pushdown = true;
    ReadCSVAddNamedParameters(read_csv);
    return read_csv;
}
```
*Code 3. A snippet of [`ReadCSVTableFunction::GetFunction()`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/function/table/read_csv.cpp#L315)*


## How to Define a Table Function?

The `TableFunction` class requires several components: 1) the name of the table function, 2) a vector of data types for arguments, 3) a pointer to a main calling function, 4) a bind function, 5) a function to initialize a global state, and 6) a function to initialize a local state. Below is the declaration of `TableFunction`.

```cpp
DUCKDB_API
TableFunction(string name, vector<LogicalType> arguments, table_function_t function, table_function_bind_t bind = nullptr, table_function_init_global_t init_global = nullptr, table_function_init_local_t init_local = nullptr);
```
*Code 4. [The declaration of the `TableFunction` initialization](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/function/table_function.hpp#L217)*

The most crucial function is the third argument, `function`, which practically produces rows. Given DuckDB's consideration for parallelism, this function may be invoked multiple times by multiple threads when your table function is called in DuckDB. The subsequent arguments `bind`, `init_global`, and `init_local` are for initialization purposes.

Before diving into the `function` argument, let's examine the `bind` function to understand the overall flow.

### A Bind Function

In the database field, the term `bind` often refers to prerequisite work, such as mapping a given table name in an SQL statement to its internal representation. In this context, the bind function prepares the table function for execution by processing input arguments, defining the output schema of the table, and creating `FunctionData` containing information for your table function.

Let's walk through an example I'm currently working on. I'm building a table function that reads a 2D array (similar to HDF5), producing a list of coordinate pairs (COO) for a table. The `ReadArrayBind` function below is a bind function.

```cpp
unique_ptr<FunctionData> ReadArrayBind(ClientContext &context, TableFunctionBindInput &input, vector<LogicalType> &return_types, vector<string> &names) {
   auto bind_data = make_unique<ArrayReadData>();   // ArrayReadData is extended from FunctionData

   bind_data->arrayname = StringValue::Get(input.inputs[0]);
   bind_data->somearg1 = UIntegerValue::Get(input.inputs[1]);
   bind_data->somearg2 = UIntegerValue::Get(input.inputs[2]);

   return_types.push_back(LogicalType::UINTEGER);
   return_types.push_back(LogicalType::UINTEGER);
   return_types.push_back(LogicalType::DOUBLE);

   names.emplace_back("x");
   names.emplace_back("y");
   names.emplace_back("val");

   return std::move(bind_data);
}
```
*Code 5. An example of a bind function*

As shown in Code 5, the `ReadArrayBind` takes `input`, `return_types`, and `names` as arguments, returning `ArrayReadData`, which is extended from `FunctionData`. The `input` argument provides the inputs for a table function, such as `read_array("array_name", somearg1, somearg2)`, which would result in `input.inputs[]` containing `{"array_name", somearg1, somearg2}`. The `return_types` and `names` define the output schema. Since the `read_array()` table function produces a COO table, the schema would be `(x UINTEGER, y UINTEGER, val DOUBLE)`. Therefore, two `LogicalType::UINTEGER` and one `LogicalType::DOUBLE` are pushed to the `return_types`, and column names `"x"`, `"y"`, and `"val"` to the `names`. For reference to `LogicalType`, see [DuckDB Data Types](https://duckdb.org/docs/sql/data_types/overview.html) and [types.hpp in the DuckDB Repository](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/common/types.hpp#L239).

A bind function must be the form of [`table_function_bind_t`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/function/table_function.hpp#L179). It returns an instance of the `FunctionData` class, which stores information necessary for running your table function. This information depends on your application and will be used during state initialization and data production. It is recommended to include input data for the table function and some pre-processed data for future use in the `FunctionData`.

The `FunctionData` should be considered as read-only data, as advised by [DuckDB](https://duckdb.org/docs/api/c/table_functions#duckdb_init_get_bind_data).

### States

Your table function can produce all the required data at once, but for various reasons, including memory management, you may want to produce data in pieces and pass it to DuckDB. In such cases, the table function needs to maintain a state.

There are two types of states you need to define for a table function: a global state and a local state. Given that multiple threads can process your table function, the global state is shared among these threads, while the local state is specific to a single thread.

The information maintained in these states varies depending on your application. The global state, accessible from multiple threads, is suitable for managing shared resources, data producers, or a workload pool. Conversely, the local state, specific to a single thread, is ideal for tracking progress for a given piece of workload. For instance, the global state of DuckDB's CSV reader maintains progress information for reading a CSV file and generates work units for worker threads upon request. These threads then process the CSV file using information stored in their local states. For more details, refer to [DuckDB's CSV reader](https://github.com/duckdb/duckdb/blob/v0.10.0/src/function/table/read_csv.cpp).

When initializing a table function, the `init_global` and `init_local` arguments of `TableFunction` are necessary for initializing the global and local states, respectively. To tailor states to your application, define your global and local state classes, extending from `GlobalTableFunctionState` and `LocalTableFunctionState`. Then, create functions for `init_global` and `init_local` that return your defined states and include these functions in your `TableFunction` initialization. Each function has to be the forms of [`table_function_init_global_t`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/function/table_function.hpp#L182) and [`table_function_init_local_t`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/function/table_function.hpp#L184), respectively.

It is essential to include a `MaxThreads()` method in your global state class, as its return value determines the number of threads DuckDB will use.

Accessing `FunctionData` returned from your bind function might be necessary. You can achieve this as shown in the following snippet (from my project).

```cpp
unique_ptr<GlobalTableFunctionState> ReadArrayGlobalStateInit(ClientContext &context, TableFunctionInitInput &input) {
    auto &data = input.bind_data->Cast<FunctionData>();

    // Now you can use your FunctionData!
    // You may want to cast it to your custom FunctionData class
    ...
}
```
*Code 6. Accessing `FunctionData` from `table_function_init_global_t`*

### A Main Table Function

The core of `TableFunction` involves defining the main table function of type [`table_function_t`](https://github.com/duckdb/duckdb/blob/v0.10.0/src/include/duckdb/function/table_function.hpp#L189), which practically produces outputs (i.e., rows in a table). You can read data from external sources or generate data in this function.

This function should set output data and cardinality (i.e., the number of rows) through the `output` argument. Since DuckDB employs [a vectorized execution engine](https://dl.acm.org/doi/abs/10.1145/3299869.3320212?casa_token=2b8uJS8qCCgAAAAA:Du3zaZsNNh9gdaw1lVqpZjmiGKXX9WCPo6eu24Rh2jLBiwyOmE0bgOy4Ijp0Yb8xaCeZMkvlPwER), data must be organized in a vector form.

Consider the main table function in Code 7 as an example. From the `// 3. get vectors` comment onwards, vectors for each column can be accessed via `output.data`. After applying `FlatVector::GetData<>()`, these vectors can be used as pointers (e.g., `uint32_t*` for `xs`). Data can then be added to these vectors to produce rows, as shown in the `// 4. put data` section. Finally, by calling `output.SetCardinality(size)`, you inform DuckDB of the number of tuples added.

The main table function would be called repeatedly until it indicates no more output is available. This can be done by setting `output.SetCardinality(0)`, after which DuckDB will not call the function again (see `// 2. tell DuckDB that we have consumed all data` comment).

```cpp
static void ReadArrayFunction(ClientContext &context, TableFunctionInput &data_p, DataChunk &output) {
    // 1. getting FunctionData, global state, and local state
    auto &data = data_p.bind_data->Cast<ArrayReadData>();
    auto &gstate = data_p.global_state->Cast<ArrayReadGlobalState>();
    auto &lstate = data_p.local_state->Cast<ArrayReadLocalState>();

    // 2. tell DuckDB that we have consumed all data
    if (checkIfFinished())
    {
        output.SetCardinality(0);
        return;
    }

    // otherwise, read the current tile
    // .... something we read array data from an external file

    // 3. get vectors
    auto xs = FlatVector::GetData<uint32_t>(output.data[0]);
    auto ys = FlatVector::GetData<uint32_t>(output.data[1]);
    auto vals = FlatVector::GetData<double>(output.data[2]);

    // 4. put data
    uint64_t size = getSize();
    for (uint64_t idx = 0; idx < size; idx++) {
        auto coords = convertTo2dCoords(idx);  // type of uint32_t*
        double val = mybuffer[idx];

        xs[idx] = coords[0];
        ys[idx] = coords[1];
        vals[idx] = val;
    }

    // 5. set cardinality
    output.SetCardinality(size);
}
```
*Code 7. An example of a main table function*

Note that DuckDB's CSV reader calls `output.Verify()` upon setting the output. This function appears to be used for debugging purposes, as it is not compiled when a debug flag is on.

## The Flow of Function Calling

When a query with a table function is invoked in DuckDB, the following steps occur in order:
1. The `bind` function is called.
2. The `init_global` function is called.
3. DuckDB spawns threads after checking `GlobalTableFunctionState.MaxThreads()`.
4. Each thread calls the `init_local` function.
5. Each thread calls the main table function until no more data is produced.
6. DuckDB returns the query results.

## Summary

In this post, we have explored how to create a DuckDB extension for a table function. A table function requires a `bind` function, a main table function, and functions for initializing the global and local states. The `FunctionData`, `GlobalTableFunctionState`, and `LocalTableFunctionState` are used across these functions to hold information and track the state of the table function. With this setup, we can implement a table function and build an extension to use it in DuckDB.

A table function can have additional options to give hints or performance advantages to DuckDB, such as estimated cardinality, predicate pushdown, projection pushdown, etc. I will discuss this topic in the next post.

Although a table function can read data, it cannot write data to an external source. To achieve this functionality, a `CopyFunction`, which is invoked with a query like `COPY TO ...`, or a `StorageExtension`, which supports statements such as `SELECT`, `INSERT`, `UPDATE`, etc., after calling `ATTACH`, needs to be created. For more information, please consult the DuckDB source code.

