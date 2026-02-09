# Dataflow: Develop pipelines - Dataflow and Beam SQL

SQL is a domain-specific language used in programming and designed for managing data held in a relational database management system. Or, more relevant here, for stream processing in a relational data stream management system. The data is accessed through relational algebra, where, for example, projections are used to pick a subset of the columns to query, filter to apply certain conditions to the rows being returned, and finally apply aggregation via the group by clause. It includes also syntax to operate on nested structures.

We could use Java SDK or Scio (developed by Spotify based on scala features) to write the joins and mappings, but SQL is a better approach, since we are providing the ability to translate the business logic written in SQL back to Apache Beam primitives, to be executed in our Dataflow serverless service in a scalable and overall concise way.

This is typical user journey on running queries over a dataset:

`BigQueryUI`: A data analyst will typically start interacting with historical data in the BigQuery UI, running SQL statements on historical data to test their hypothesis about what happened.
`Dataflow UI`: After testing on historical data, they will ideally want to test the same business logic in the form of SQL statements over real-time data this time.
`Beam SQL`: Lastly, once the data analyst is happy with the logic, they would be able to pass those SQL statements to the data engineer, who will be able to implement them with little change in the form of SQLTransforms inside the Beam Java SDK.

Please note that Beam SQL and Dataflow SQL are effectively identical, but while Beam SQL offers a programmatic interface, Dataflow SQL also offers a UI interface.

## BeamSQL

It allows a Beam user to query bounded and unbounded PCollections with SQL statements.

Works with batch and stream inputs.

Your SQL query is embedded using `SQLTransforms`, an encapsulated segment of a Beam pipeline similar to `PTransforms`. You can freely mix SQLTransforms and other PTransforms if needed.

It also supports UDFs in Java.

BeamSQL Supports multiple dialects, like `Beam Calcite SQL` and `Google ZetaSQL`.

`Apache Calcite SQL`:
Provides compatibility with other OSS SQL dialects (e.g. Flink SQL), more mature implementatio (default in `Apache Beam`).

It supports windowing when working with streaming (or unbounded) data.

`ZetaSQL`

Provides BigQuery capabilities

## DataflowSQL

It's a Beam ZetaSQL SqlTransform in a Dataflow template!

Write Dataflow SQL queries in the Bigquery UI or gcloud CLI.

Uses `ZetaSQL` (not Calcite SQL), the same dialect as Bigquery Standard SQL.

Dataflow SQL UI: BIgquery UI

## Windowing in SQL

You can implement streaming windows in SQL so that you can run them as streaming queries:

- Tumbling windows (fixed windows)

```sql
SELECT productId,
TUMBLE_START("INTERVAL 10 SECOND") as period_start, COUNT(transactionId)
AS num_purchases
FROM pubsub. topic.`instant-insights`.`retaildemo-online-purchases-json`
AS pr
GROUP BY productId, TUMBLE(pr.event_timestamp, "INTERVAL 10 SECOND")
```

- Hopping windows (sliding windows)

```sql
SELECT productId,
HOP_START ("INTERVAL 10 SECOND", -- window length
           "INTERVAL 30 SECOND") as period_start, -- when new window begins.
HOP_END ("INTERVAL 10 SECOND",
         "INTERVAL 30 SECOND") as period_end,
COUNT (transactionId) AS num_purchases
FROM pubsub.topic.`instant-insights`.`retaildemo-online-purchases-json`
AS pr
GROUP BY productId,
HOP (pr.event_timestamp,
"INTERVAL 10 SECOND",
"INTERVAL 30 SECOND")
```

- Session windows

```sql
SELECT userId,
SESSION_START("INTERVAL 10 MINUTE") AS interval_start, 
SESSION_END("INTERVAL 10 MINUTE") AS interval_end,
COUNT (transactionId) AS num_transactions
FROM pubsub.topic.`instant-insights`.`retaildemo-online-purchases-json`
AS pr
GROUP BY userId,
SESSION(pr.event_timestamp, "INTERVAL 10 MINUTE")
```

## Bean Dataframes

A more Pythonic expressive API compatible with Pandas DataFrames.

The big difference between `beam data frames` and `Pandas` is that operations are deferred by the Beam API to support the beam parallel processing model.

You can think of beam data frames as a domain specific language for beam pipelines similar to beam SQL data frames is a DSL built into the Beam Python SDK.

Differences between Beam dataframes and Pandas dataframes:

- Beam DataFrames are unordered, so shifting data is not possible
- Operations are deferred, so an operation may not be available for control flow or interactive visualization. For example, you can compute a sum, but you can't branch on the result.
- Result columns must be computable without access to the data. For example, you can't use transpose.

## Notebooks

Notebooks are useful for developing purposes, when you want to see intermediate results instead of submitting a job and waiting for dataflow to fail / process it. It mixes Beam Python SDK with Jupyter Lab interface.

InteractiveRunner is an Apache Beam runner (it's a Python API class in the Apache Beam Python SDK) used in notebooks for interactive development and testing of Dataflow pipelines. Purpose of it is to see intermediate results without submitting a full Dataflow job, iterate and debug pipelines interactively, and avoiding waiting for a full job to complete or fail

First challenge when reading intermediate resutls is: If we're processing an unbounded / streaming source of information, how do we tell Beam to stop so that we can query the dataset? There are a couple of options:

- `ib.options.recording_duration`: Sets the amount of time the InteractiveRunner records data from an unbounded source
- `ib.options.recording_size_limit`: Sets the amount of data the InteractiveRunner records (in bytes) from an unbounded source.

Other operations using InteractiveRunner:

- `ib.show(output_collection, include_window_info=True)`: Materializes the resulting `output_collection` PCollection in a table
- `ib.collect(output_collection, include_window_info=True)`: Loads the output in a Pandas Dataframe
- `ib.show(output_collection, include_window_info=True, visualize_data=True)`: Visualizes the data in the notebook.

