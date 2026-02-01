### Bigquery serving real time data

## BigQuery's various data ingestion methods

###  Batch vs. streaming ingestion

### ETL vs. ELT

### The BigQuery Storage Write API (write-oriented architecture for fast, reliable and efficient real time data processing)

  - Google's recommended streaming API
  - gRPC based bi-directional API
  - Apache Arrow and JSON support
  - Full Java, Go, and Python support
  - More efficient Protocol Buffer support  
  - Unified API for both streaming and batch workloads
  - Offers exactly once messaging
  - Default 3GB throughput
  - Lower cost (up to 50%) without a minimum message size
  - Schema update detection

### BigQuery Data Transfer Service (DTS) (for batch)

  - Not all data is streaming data. Many systems operate on a batch or scheduled basis. For this, BigQuery provides the DTS.
  - DTS is not a streaming service.
  - DTS is a fully managed, automated data loading service for scheduled transfers which is incredibly useful for pull data from other Google applications like Ads and YouTube, or from cloud storage services like Amazon S3 on a recurring schedule.
  - DTS is a "set it and forget it" way to ingest data from supported third-party sources.

### Change Data Capture (CDC)

### Complex transformations: Dataflow and Continuous queries

## Complex transformations: BigQuery Continuous Queries

Periodical automatated queries run on when important information is updated in BigQuery. 

They are SQL statements that execute continuously, processing data as it arrives in your BigQuery tables in real-time. This allows you to perform time-sensitive tasks, such as generating real-time insights, applying machine learning models, and replicating data to other platforms. It’s like having an analyst who is always on, constantly monitoring your data streams and triggering actions the moment something noteworthy occurs.

## Reverse ETL

Reverse ETL is the process of moving data from a storage system, like BigQuery, back into operational systems. This allows you to activate your data by making it available in the tools your business teams use every day like CRMs, marketing automation platforms, and advertising networks.

It can write continuously pushed to `Bigtable`, but the following are also supported destinations:

  - `Pub/Sub`: Publish continuous query output to Pub/Sub topics to trigger downstream applications. 
  - `Spanner`: Export data to Spanner to serve data to application users without exhausting BigQuery quotas. 
  - Other `BigQuery` tables: Write the results of a continuous query into another BigQuery table for further analysis.

## Bigtable

For these operational workloads that demand millions of lookups with single-digit millisecond latency we turn to the final main component of our streaming architecture: Bigtable, Google Cloud's fully managed, petabyte-scale NoSQL database service. This is where your data will land so that it's ready to be served instantly and at scale, driving the real-time applications that make the Galactic Grand Prix possible.

Data structure: Unstructured. Optimized for NoSQL queries but supports GoogleSQL. It is a key-value and wide-column database designed for high throughput, low-latency workloads. It offers a flexible schema that adapts to evolving data needs

Deployment scope: Global. Ready for application serving. It supports topologies from a single zone up to eight regions. Multi-region, multi-primary deployments bring data closer to customers for best latencies.

Latency and throughput: Milliseconds for NoSQL queries over large datasets. It is a low-latency NoSQL database, ideal for latency-sensitive workloads like personalization, clickstream, IoT, and ML model training. It supports millions of reads per second (RPS) and predictable single-digit millisecond latency.

Primary use cases: Real-time application serving, IoT and time series data, Machine learning workloads (online - serving prediction), Data integration

Scalability: Infinitely scalable with no limits on horizontal scalability. Decouples compute from storage. Automatically scales resources to adapt to server traffic and is optimized for high-throughput, low-latency application workloads. Its throughput scales linearly with the number of nodes, which can be adjusted to control Queries Per Second (QPS).

Features:

- Bigtable schema is defined by the application logic, instead of the DB schema. 
- Row key: Index. All data is order lexicographically by this key.
- Bigtable also supports asynchronous secondary indexes. When you write data to the main table, there's an asynchronous process that applies the query of that data into a materialized view, which columns are defined by that secondary index, so that materialized view will have eventual consistency, and it does not impact the write latency of the source table. Your application can then choose to query data from the main table or the materialized view.
- Continuous materialized views: Active ongoing process to query data continuously, making it much faster to read and query



Summary of whole streaming chapter:


Topic

Key concepts

Why it matters

Real-time processing and transformation (ETL)

You learned that to build streaming data pipelines, raw event data is read from streaming sources like Pub/Sub or Managed Service for Apache Kafka. From there, the processing stage begins with Dataflow - a foundational tool for ETL, processing raw data and handling transformations. It uses windowing to aggregate continuous, unbounded data streams, and can even act as an inference engine to make machine learning predictions using transforms like MLTransform and RunInference.

This topic is critical because raw, chaotic stream data (like player decisions or race car movements) cannot be directly used for analytics or applications without preparation.

Strategic application serving

You learned about the Dual Destination Strategy, which helps you choose between BigQuery and Bigtable based on your needs. BigQuery is best for analysis, dashboards, and long-term storage, while Bigtable is for operational workloads and application backends, like live leaderboards. You also discovered the Feedback Loop (Reverse ETL), where insights from BigQuery are sent back to operational systems like Bigtable to complete the analytical cycle.

This topic defines the success of the pipeline by ensuring the processed data lands in the correct destination, meeting the required time to insights.

High-integrity data ingestion and synchronization

You learned that the Storage Write API is Google's recommended streaming mechanism. It's better than older methods because it provides higher throughput and costs less. For data integrity, it uses exactly-once semantics to prevent duplicates, even when there are failures. This same API is also used for BigQuery's native Change Data Capture (CDC), which handles real-time data updates without the need for complex custom solutions.

The intake layer is critical because it dictates data quality, latency, cost, and reliability before processing even begins.

Performance optimization and query design

You learned how to optimize Google Cloud data services. For BigQuery, improve performance and cut costs with partitioning (filtering data before a query) and clustering (organizing data within partitions). Creating views also helps. When writing complex queries, use Common Table Expressions (CTEs) to break down subqueries and always filter early.

For Bigtable, achieve low latency by testing on a multi-node cluster with at least 300 GB of data. A heavy pretest is a key step that helps Bigtable distribute reads and writes evenly, and you know that throughput scales linearly with the number of nodes.

Optimization is very important for meeting service-level agreements (SLAs) for latency and for controlling costs in BigQuery and Bigtable.

