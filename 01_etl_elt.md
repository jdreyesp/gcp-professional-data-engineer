# ELT / ETL

## General tools

Dataprep
Cloud Data Fusion
DataProc
Dataflow

            Service	Type	Code Required	Streaming	Batch	Best For
Dataprep	Visual/no-code	No	            No	        Yes	    Data preparation
Data Fusion	Visual/low-code	Minimal	        Yes	        Yes	    Enterprise ETL
Dataproc	Code-based	    Yes (Spark)	    Yes	        Yes	    Spark/Hadoop workloads
Dataflow	Code-based	    Yes (Beam)	    Yes	        Yes	    Unified streaming/batch

### Transformation only

            Service	Type	Code Required	Streaming	Batch	Best For
Dataform	Code-based	    Yes (SQL)	    No	        Yes	    SQL-based data modeling/ELT

## Automation and workflow orchestration

Cloud scheduler - yaml definition
Cloud composer - Apache Airflow (python)
Clour Run functions (event triggers (HTTP, pubsub, gcs, firestore, eventarc)) -> serverless execution
Eventarc - Event sources (GCP ones, third-party ones or custom ones via pub/sub) connected to event targets (Cloud run functions, http endpoitns, workflows)

            Service	                Trigger Type	    Serverless	Coding Effort	Programming Languages
Cloud Scheduler	                    Schedule, Manual	Yes	        Low	            YAML (with Workflows)
Cloud Composer	                    Schedule, Manual	No	        Medium	        Python
Cloud Run Functions	                Event	            Yes	        High	        Python, Java, Go, Node.js, Ruby, PHP, .NET core
Eventarc	                        Event	            Yes	        High	        Any

