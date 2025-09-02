---
layout: post
title:  "Vectorization and Options for DuckDB Table Function"
date:   2024-09-15 23:38:28 +0900
categories: duckdb
---

In [the previous post]({% post_url 2024-04-09-How-to-make-a-DuckDB-extension-for-a-table-function %}), we explored how to create a DuckDB extension for a table function with simple examples. 
In this post, we will cover 1) result vectorization and 2) table function options for usability and performance improvements.

**Note:** This post is based on my analysis of DuckDB source code and extension templates. 
It may contain inaccuracies. 
If you find any, please kindly let me know. The content is based on DuckDB v1.0.1.

## Vectorized Execution and Result Splitting

DuckDB adopts vectorized execution for query processing. 
An operator in a query plan uses a vector as the unit of data, rather than a tuple, so vectors are produced and consumed.

In DuckDB, there are several vector types, and the flat vector is the basic one, similar to a standard C/C++ array (a sequence of values).

By default, an empty flat vector with a size of `STANDARD_VECTOR_SIZE` is passed to the table function, which then fills the vector. 
If the data your table function produces exceeds `STANDARD_VECTOR_SIZE`, you need to split the data and return it incrementally.

*Code 1* shows how to handle result splitting in the `GenSequenceFunction`. 
The function is designed to be called multiple times. 
In the example, `gstate.total` and `gstate.cur` represent the total number of tuples to produce and the number of tuples produced so far, respectively. 
By using these, we calculate `local_remains`, which indicates how many tuples to produce in the current function call. 
The table function then fills the vector using `local_remains`.

```cpp
static void GenSequenceFunction(ClientContext &context, TableFunctionInput &data_p, DataChunk &output) 
{
  auto &gstate = data_p.global_state->Cast<GSReadGlobalState>();
  
  // get a flat vector
  auto vec = FlatVector::GetData<uint64_t>(output.data[0]);  // assume only single column

  // calculate the number of tuples to produce
  auto total_remains = gstate.total - gstate.cur;
  auto local_remains = std::min((uint64_t)STANDARD_VECTOR_SIZE, total_remains);

  // generate sequence
  for (uint64_t idx = 0; idx < local_remains; idx++) {
    uint64_t val = gstate.cur + idx;
    vec[idx] = val;
  }

  // update global state for further function calls
  gstate.cur += local_remains;

  // set cardinality
  output.SetCardinality(local_remains);
}
```
*Code 1. Example of a table function producing vectors up to `STANDARD_VECTOR_SIZE`*


## Table Function Options

A table function can have additional options that provide DuckDB with more information about the function. 
Some notable options include:
- `table_scan_progress`: Shows how much of the data has been produced (in percentage).
- `cardinality`: Specifies how many tuples the table function will produce.
- `filter_pushdown`: Indicates whether the table function can handle predicate pushdown (i.e., process the `WHERE` clause).
- `projection_pushdown`: Indicates whether the table function can handle projection pushdown (i.e., process the `SELECT` clause).
- `filter_prune`: Allows the table function to produce rows without columns in the `WHERE` clause.

Let's explore how to use `table_scan_progress` and `cardinality`, and how they affect query execution in DuckDB. 
I will cover `filter_pushdown`, `projection_pushdown`, and `filter_prune` in the next post.

Note that the following snippet is a simplified `TableFunction` class (which you load in the `Load()` function) with type definitions. 
You can find the full set of options for the `TableFunction` class [here](https://github.com/duckdb/duckdb/blob/v1.0.0/src/include/duckdb/function/table_function.hpp).

```cpp
typedef double (*table_function_progress_t)(ClientContext &context, const FunctionData *bind_data, const GlobalTableFunctionState *global_state);

typedef unique_ptr<NodeStatistics> (*table_function_cardinality_t)(ClientContext &context, const FunctionData *bind_data);

class TableFunction {
  table_function_progress_t table_scan_progress;
  table_function_cardinality_t cardinality;
  
  bool projection_pushdown;
  bool filter_pushdown;
  bool filter_prune;
}
```
*Code 2. Simplified `TableFunction` class and relevant types*

### `table_scan_progress`

The `table_scan_progress` function returns the percentage (between `0` and `1`) of data produced by the table function. 
A function definition that accepts `ClientContext`, `FunctionData*` (bind data), and const `GlobalTableFunctionState*` (global state), and returns a double, is required (see *Code 3* for an example).

You may need to store information such as the number of tuples produced or the progress value in the global state, so it can be retrieved by the `table_scan_progress` function. This function is called frequently, and its result is displayed in DuckDB’s progress bar.

*Figure 1* shows an example of the DuckDB progress bar for the function in *Code 3*.

```cpp
double ReadArrayProgress(ClientContext &context, const FunctionData *bind_data, const GlobalTableFunctionState *global_state)
{
  auto gstate = global_state->Cast<ArrayReadGlobalState>();

  // gstate.cur: the number of cells produced so far
  // data.num_cells: the number of cells to process
  auto progress = (double)gstate.cur / data.num_cells;    
  return progress;
}

TableFunction ArrayExtension::GetTableFunction()
{
  TableFunction function = TableFunction("read_array", {LogicalType::VARCHAR, LogicalType::LIST(LogicalType::INTEGER)}, ReadArrayFunction, ReadArrayBind, ReadArrayGlobalStateInit, ReadArrayLocalStateInit);

  // set table_scan_progress 
  function.table_scan_progress = ReadArrayProgress;
  return function;
}
```
*Code 3. Example of `table_scan_progress` function and its setup*

```
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D SELECT * FROM read_array("__temparr_0", [0, 0]);
 50% ▕██████████████████████████████                              ▏ 
```
*Figure 1. Example of the DuckDB progress bar*


### `cardinality`

The `cardinality` function provides information about the expected and maximum number of tuples the table function will produce, affecting how DuckDB plans query execution.

You pass this information via [the `NodeStatictics` class](
https://github.com/duckdb/duckdb/blob/v1.0.0/src/include/duckdb/storage/statistics/node_statistics.hpp). As shown in *Code 4*, enable `has_estimated_cardinality` and `has_max_cardinality`, then assign appropriate values to `estimated_cardinality` and `max_cardinality`.

DuckDB uses `NodeStatistics` for query planning, which can be seen using the `EXPLAIN` statement, as shown in *Figure 2*. The `EC` value in each node represents the estimated cardinality.

```cpp
unique_ptr<NodeStatistics> ReadArrayCardinality(ClientContext &context, const FunctionData *bind_data) {
  auto stat = make_uniq<NodeStatistics>();
  auto &array_data = bind_data->Cast<ArrayReadData>();

  // enable both estimated cardinality and max cardinality
  stat->has_estimated_cardinality = true;
  stat->has_max_cardinality = true; 

  // calculate cardinality
  stat->estimated_cardinality = 1;
  for (uint32_t idx = 0; idx < array_data.dim_len; idx++) {
    // This example reads an array
    // Estimated number of tuples is the product of the dimensions, e.g., M * N for M x N matrix
    stat->estimated_cardinality *= array_data.array_size[idx];
  }
  // max_cardinality is the same in this example
  stat->max_cardinality = stat->estimated_cardinality;
  return std::move(stat);
}

TableFunction ArrayExtension::GetTableFunction() {
  ...
  // set cardinality 
  function.cardinality = ReadArrayCardinality;
  ...
} 
```
*Code 4. Example of setting cardinality*

```
D EXPLAIN SELECT val FROM read_array("__temparr_0", [0, 0]) WHERE x = 10 AND y = 10;

┌─────────────────────────────┐
│┌───────────────────────────┐│
││       Physical Plan       ││
│└───────────────────────────┘│
└─────────────────────────────┘
┌───────────────────────────┐
│         PROJECTION        │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│            val            │
└─────────────┬─────────────┘                             
┌─────────────┴─────────────┐
│           FILTER          │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│  ((y = 10) AND (x = 10))  │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│       EC: 400000000       │
└─────────────┬─────────────┘                             
┌─────────────┴─────────────┐
│        READ_ARRAY         │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│             x             │
│             y             │
│            val            │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│       EC: 2000000000      │
└───────────────────────────┘    
```
*Figure 2. Example output of the DuckDB EXPLAIN statement with Code 4*


## Summary

In this post, we explored vectorized data processing and table function options for progress tracking and cardinality estimation. 
When a table function generates data exceeding `STANDARD_VECTOR_SIZE`, it needs to split the data incrementally. 
The `table_scan_progress` function allows progress tracking, while `cardinality` provides DuckDB with important planning information.

In the next post, we will discuss additional options: `projection_pushdown`, `filter_pushdown`, and `filter_prune`. 
These options enable the table function to reduce the amount of data processed, potentially improving query execution speed.