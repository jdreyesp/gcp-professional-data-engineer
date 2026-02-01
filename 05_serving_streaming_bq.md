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