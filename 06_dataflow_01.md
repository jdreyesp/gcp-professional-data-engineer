# Dataflow - module 1 - Introduction

## The Beam vision

The Beam vision is to provide a comprehensive portability framework for data processing pipelines, one that allows you to write your pipeline once in the programming language of your choice and run it with minimal effort on the execution engine of your choice.

With Beam, you also have the flexibility to move your data processing pipeline from your own premise environment to Dataflow on Google Cloud or any other clouds. There is no vendor lock-in.

The portability framework is a language-agnostic way of representing and executing Beam pipelines. It introduces well-defined, language neutral data structures and protocols between the SDKs and the runners. This interoperability layer is called Portability API and enables you to use the language of your choice with the runner of your choice, thus ensuring that SDKs and runners can work with each other uniformly.

Moreover, because docker containerization is used, you can customize the execution environment running on the worker nodes of the back end service.

## Portability 

Portability brings several additional benefits.

With portability, every runner can work with every supported language.

Containerization allows us a configurable, hermetic worker environment.

You can have multi-language pipelines and cross-language transforms because of the language-agnostic representation of pipelines and the isolated environment of each operation.

To use the portability features mentioned earlier, you must use the Dataflow Runner v2. It supports custom containers, multi-language pipelines and cross-language transforms.

## Containerization

Containarization:Because of containerization, you can benefit from ahead-of-time installation. You can include arbitrary dependencies, and even further customization is possible. To use this feature, you need to have the Apache Beam SDK version 2.25.0 or later installed.

Cross-language and portability: Thanks to this feature, you're no longer tied to a single programming language in a pipeline. For instance, with a Python streaming pipeline example, you can call a `ReadFromKafka` transform. This is a transform developed using the Beam Java SDK. Under the hood, to make Java transforms available to a dataflow Python pipeline, The Apache Beam Python SDK starts up a local Java service on your computer to create and inject the appropriate Java pipeline fragments into your Python pipeline. Then the SDK downloads in stages the necessary Java dependencies needed to execute these transforms, and at run time, the dataflow workers will execute the Python and Java code simultaneously to run your pipeline.

## Why Dataflow is so loved

- First, it's because it is fully managed and autoconfigured.
- Second, Dataflow optimizes the graph execution by fusing operations efficiently and by not waiting for previous steps to finish before starting a new one unless there is a dependency involved.
- Third, autoscaling happens step by step in the middle of a pipeline job. As a job needs more resources, it receives them automatically.
- All of this happens while maintaining strong streaming semantics Aggregations like sums and counts are correct, even if the input source sends duplicate records. Dataflow can also handle later-arriving records with intelligent watermarking.

## Dataflow shuffle service

The Dataflow Shuffle Service is a managed service that handles data redistribution (shuffling) between pipeline stages. It moves data efficiently across worker VMs during operations like grouping, joining, and aggregations.
The Problem It Solves
In distributed pipelines, operations like GroupByKey, CoGroupByKey, and Combine require:
Collecting data with the same key from multiple workers
Redistributing it so all values for a key end up on the same worker
This is the "shuffle" phase
Without a dedicated shuffle service, workers would:
Send data directly to each other (many point-to-point connections)
Use local disk for intermediate storage (limited capacity)
Create network bottlenecks and inefficiencies
How It Works
Centralized service: A managed service (separate from your workers) handles shuffle operations
Efficient storage: Uses Google Cloud Storage (GCS) as intermediate storage for shuffled data
Network optimization: Optimizes data transfer patterns to reduce network overhead
Decoupled from workers: Workers don't need to store large amounts of intermediate data locally
Flow:
Worker 1 → Shuffle Service (GCS) → Worker 2Worker 3 → Shuffle Service (GCS) → Worker 2Worker 4 → Shuffle Service (GCS) → Worker 2
All data for a key gets routed through the shuffle service to the correct worker.

## Dataflow streaming engine

Just like shuffle component in batch, the streaming engine offloads the window state storage from the persistent disks attached to worker VMs to a back-end service. It also implements an efficient shuffle for streaming cases.

With the dataflow streaming engine, you will have a reduction in consumed CPU, memory, and persistent disk storage resources on the worker VMs.

Streaming engine works best with smaller worker machine types like n1-standard-2, and does not require persistent disks beyond a smaller worker boot disk.

## Flexible Resource Scheduling (FlexRS)

FlexRS helps you reduce the cost of your batch processing pipelines because you can use advanced scheduling techniques in the Dataflow Shuffle Service and leverage a mix of preemptible and normal virtual machines.

When you submit a FlexRS job, the Dataflow service places the job into a queue and submits it for execution within six hours from job creation. This makes FlexRS suitable for workloads that are not time-critical, such as daily or weekly jobs that can be completed within a certain time window.

As soon as you submit your FlexRS job, Dataflow records a job ID and performs an early validation run to verify execution parameters, configurations, quota and permissions.

In case of failure, the error is reported immediately, and you don't have to wait for a delayed execution.

Standard Dataflow:
- Uses regular (on-demand) VMs
- Jobs start immediately when resources are available
- Higher cost, predictable execution

FlexRS:
- Uses a combination of preemptible VMs (up to 80% cheaper) and normal VMs
- Jobs can wait in a queue for capacity
- If a VM is preempted, Dataflow automatically retries the work
- Lower cost, potentially longer execution time

## IAM

When the pipeline is submitted, it is sent to two places. 

- The SDK uploads your code to Cloud storage and sends it to the Dataflow service.
- The Dataflow service validates and optimizes the pipeline, it creates the Compute Engine virtual machines in your projects to run your code, it deploys the code to the VMs, and it starts to gather monitoring information for display. 

When all that is done, the VMs will start running your code.

**3 credentials determine whether a Dataflow job can be launched**:

- `User role`: When you `submit a code`, whether you are allowed to submit it is determined by the IAM role set to your account. On Google Cloud, your account is represented by your email address. It can have these possible roles:
  - `Dataflow viewer role`: It allows users who have the role to only view Dataflow jobs either in the UI or by using the command line interface.
  - `Dataflow developer role`: For a job to run on Dataflow, the user must be able to submit the job to Dataflow, stage files to cloud storage, and view the available Compute Engine quota. If a user only has the Dataflow developer role, they can view and cancel jobs that are currently running, but they cannot create jobs because the role does not have permissions to stage the files and view the Compute Engine quota.
  (NOTE: You can use the Dataflow developer role as a building block to compose custom roles. For example, if you also want to be able to create pipelines, you can create a role that has the permissions from the Dataflow developer role plus the permissions required to stage files to a bucket and to view the Compute Engine quota.)
  - `Dataflow admin role`: Use this role to provide a user or group with the minimum set of permissions that allow both creating and managing Dataflow jobs. It allows a user or group to interact with Dataflow and stage files in an existing Cloud storage bucket and view the Compute Engine quota.
- `Dataflow service account`: Dataflow uses the Dataflow service account to interact between your project and Dataflow. For example, to check project quota, to create worker instances on your behalf, and to manage the job during job execution. When you `run` your pipeline on Dataflow, it uses the service account *service@dataflow-service-producer-prod*. This account is automatically created when the Dataflow API is enabled.
- `Controller service account`: The controller service account is assigned to the Compute Engine VMs to run your Dataflow pipeline. 
By default, workers use your project's Compute Engine default service account as the controller service account (the default `<project-number>-compute @developer.gservices.com` that is automatically created when we enable the Compute Engine API in our GCP project). 
However, for production workloads, we recommend that you create a new service account with only the roles and permissions that you need. At a minimum, your service account must have the `Dataflow worker` role and can be used by adding the service account email flag when launching a Dataflow pipeline.
When using your own service account, you might also need to add additional roles to access different Google Cloud resources. For example, if your job reads from BigQuery, your service account must also have a role like the BigQuery Data Viewer role.

## Quotas

- `CPU quota`: CPU quota is the total number of virtual CPUs across all of your VM instances in a region or zone. Any Google Cloud product that creates a Compute Engine VM, such as Dataproc, GKE, or AI Notebooks, consumes this quota. CPU quota can be viewed in the UI on the IAM Quota page.

In Dataflow, if the VM size selected is n1-standard-1, meaning 1 CPU core per VM, the CPU usage will be 100. If the VM size selected is n1-standard-8, that would mean 800 CPUs are needed. If the limit is 600, the job will display an error because the CPU limit has been exceeded.

- `In use IP addressses`: The in-use IP address quota limits the number of VMs that can be launched with an **external** IP address for each region in your project.

Jobs that access APIs and services outside Google Cloud require internet access. However, if your job does not need to access any external APIs or services, you can launch the Dataflow job using internal IPs only, which saves money and conserves the In-use IP address quota.

When you launch a Dataflow job, the more restrictive quota takes precedence. (if only 575 In-use IP addresses remain, and 600 CPUs remain, 575 applies)

- `Persistent disk`: You can choose between two different types of Persistent Disks when running Dataflow jobs. You can launch jobs with either legacy Hard Disk Drives or modern Solid State Drives. Each disk type has a limit per region that can be used.

Use Pd-standard for Hard Disk Drives and pd-ssd for Solid State Drives.

- Batch pipelines: When you launch a batch pipeline, the ratio of VMs to PDs is 1:1 (For each VM, only one persistent disk (PD) is attached).

For jobs running shuffle on worker VMs, the default size of each persistent disk is 250 GB.

If the Batch job is running using Shuffle Service, the default PD size is 25 GB.

- Streaming pipelines: Streaming pipelines, however, are deployed with a fixed pool of Persistent Disks. Each worker must have at least 1 persistent disk attached to it, while the maximum is 15 persistent disks per worker instance.

As with Batch jobs, Streaming jobs can be run either on the worker VMs or on the Dataflow backend. When you run a job using the Dataflow backend, the feature that is used is Dataflow's Streaming Engine (Streaming Engine moves pipeline execution out of the worker VMs and into the Dataflow service backend). 

For jobs launched to execute in the worker VMs, the default persistent disk size is 400 GB.

Jobs launched using Streaming Engine have a persistent disk size of 30 GB.



It is important to note that the amount of disk allocated in a streaming pipeline is equal to the `max_num_workers` flag.

For example, if you launch a job with 3 workers initially and set the maximum number of workers to 25, 25 disks will count against your quota, not 3.

To set the maximum number of workers that a pipeline can use, use the --max_num_workers flag. This cannot be above 1000. When you launch a streaming job that does not use Streaming Engine, the flag --max_num_workers is required.

For streaming jobs that do use Streaming Engine, the --max_num_workers flag is optional. The default is 100.


## Security 

