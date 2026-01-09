# LakeHouse components

GCS: An open table format that adds structure and performance to data residing in GCS. It organizes data files into queryable tables, offering critical features like schema evolution (adapting to changing data requirements), hidden partitioning (automatic data organization for efficient querying), time travel (querying past data versions for auditing), and atomic transactions (reliable concurrent operations).

Apache Iceberg: An open table format that adds structure and performance to data residing in GCS. It organizes data files into queryable tables, offering critical features like schema evolution (adapting to changing data requirements), hidden partitioning (automatic data organization for efficient querying), time travel (querying past data versions for auditing), and atomic transactions (reliable concurrent operations).

BigQuery: Google's fully managed, serverless enterprise-grade data warehouse, serving as the central analytics and query engine. Its architecture separates storage from compute, enabling independent scaling and fast querying via the Dremel engine. Through BigLake technology, BigQuery can directly query open-format data (like Iceberg tables) in GCS or other clouds, applying BigQuery's powerful optimizations such as partitioning and clustering by leveraging Iceberg's metadata.

BigLake: BigLake centralizes governance, extending BigQuery's fine-grained security (e.g., row-level and column-level security) to data stored in Cloud Storage. BigQuery can also query its own native optimized storage for "hot" datasets and external operational databases like AlloyDB for PostgreSQL via federated queries, seamlessly linking real-time and historical data without movement.

Dataplex: A unified data fabric and centralized metadata hub for discovering, managing, monitoring, and governing data across the entire lakehouse, ensuring data quality and consistency.

Sensitive Data Protection: Automates the discovery, classification, and protection of sensitive data (e.g., Personally Identifiable Information (PII) like email addresses, phone numbers, or credit card numbers) using techniques like masking, tokenization, or redaction.

BigQuery ML and Vertex AI: Enable advanced analytics and machine learning directly on lakehouse data. BigQuery ML allows data analysts to build and deploy ML models using familiar SQL queries, while Vertex AI provides a comprehensive platform for more complex custom models and MLOps, seamlessly integrated with BigQuery data.
