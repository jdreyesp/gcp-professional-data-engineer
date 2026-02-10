# Reliability

Batch jobs are simple. If a batch job does not launch or if it fails during execution, you can always rerun the job. Source data is not lost and partial data written to sinks can be rewritten, if it was written at all.

Streaming jobs, on the other hand, are more complex. Streaming jobs are continuously processing data and behave like a long-lived application. Thus, reliability is of the utmost importance. Most of the reliability best practices in this specific module are for streaming pipelines.

We can classify the pipeline's failures in two broad categories, failures related to user code and data shapes, and failures caused by outages (service, zonal, and/or regional).

First, we start with a reminder that for batch jobs, tasks with failing items are retried up to `four times`. For streaming pipelines, failing work items will be retried indefinitely.

Erroneous records may cause your pipeline to get stuck or fail outright. => As described in previous modules, we highly recommend implementing a dead-letter queue, and error logging to prevent these failure modes.

## Monitoring

To maximize the reliability of your workloads, it is essential to implement a robust monitoring and alerting strategy. Dataflow provides a web based monitoring interface that can be used to view and manage jobs. You can create metrics based alerts with a couple of clicks.

In addition, data flows integration with cloud monitoring provides extensive flexibility for pipeline monitoring. You can collect custom metrics that point to health conditions that are relevant for your use case, like the number of erroneous records that have been detected. The possibilities are endless with Dataflow's monitoring integration.

For batch workloads, you might be interested in the overall runtime of your job. If the job runs on a recurring schedule, you might want to ensure that the job completes successfully within a given period of time.

For streaming pipelines, you want steady and sustained data processing. Dataflow provides standard metrics like `data freshness` and `system latency` that make it easy to track whether your pipeline is falling behind. You can create an alert with a couple of clicks from the Dataflow monitoring UI that will be triggered if this selected metrics fall behind the specified threshold.

## Geolocation

When a user submits a job to a regional endpoint without explicitly specifying end zone (let's say it does `--region $REGION` only), Dataflow service routes the job to a zone in the specified region based on resource availability. If you explicitly specify a zone, you will not get this benefit. If a job submission fails due to a zone issue, retrying without explicitly specifying a zone will usually fix the issue. This is a helpful technique in the event of a zonal outage.

Note that you cannot change the location of a job after you got started. If it is a streaming job, you will have to `drain` or `cancel` the pipeline first before launching it again.

When thinking about the locations of your Dataflow job, there are three elements to be aware: Your `sources`, your `processing`, and your `sinks`. **You should always locate your resources in the same region**.

Services like Google Cloud Storage, BigQuery, and Pub/Sub provide geo-redundant options that make your data seamlessly accessible in multiple regions.

Dataflow processing can only occur in one region. But in the event of a regional outage, using multiregional sources and sinks allows you to move your data processing to a different region without suffering from performance penalty.

You should try to avoid any configurations that have critical cross-region dependency. If you have a pipeline that has a critical dependency on services from multiple regions, your pipeline is likely to be affected by a failure in any of those regions.

For example, a pipeline that is reading from my Cloud Storage bucket in us-central1 and writing for BigQuery table in us-east4 could go down if either one of those two regions are down.

## Disaster recovery

Data is your most prized asset, which is why it is essential to have a disaster recovery strategy in place for your production systems.

We'll see some disaster recovery methods. These methods only apply to streaming pipelines.

- One way is to take snapshots of your data source. This capability is supported in many popular relational databases and data warehouses.

- But what if you are using a messaging application? Pub/sub snapshots is the solution. You can implement a disaster recovery strategy with two features: 
  - Pub/Sub Snapshots, which allows you to capture the message acknowledgement state of a subscription AND
  - Pub/Sub Seek, which allows you to alter the acknowledgement state of messages in bulk.

### Pub/sub snapshots + seek strategy

If you are using this strategy, you will have to reprocess messages in the event of a pipeline failure. This means you will have to consider how to reconcile this in your data sync and duplicate some records that have been written twice.

How to use pub/sub snapshots?? Few steps in order needs to be followed:

1. Make a snapshot of your subscription: `gcloud pubsub snapshot create my-snapshot --subscription=my-sub`

2. Stop and drain your Dataflow job: `gcloud dataflow jobs drain [job-id]`

3. Once the job has stopped, you can use Pubsub seek functionality to revert the acknowledgement of messages in your subscription: `gcloud pubsub subscriptions seek my-sub --snapshot=my-snapshot`

4. Finally, you're ready to resubmit your pipeline. You can launch your pipeline using any of the ways that you use to deploy your Dataflow job either directly from your development environment or by using the command line tool to launch a template. Example with simple command for a templated job that has been launched with a command line interface: `gcloud dataflow jobs run my-job-name --gcs_location=my_gcs_bucket`

An important caveat to consider is that Pub/Sub messages have a maximum data retention of seven days. This means that after seven days, a Pub/Sub Snapshot no longer has any use for your stream processing. If you choose to use Pub/Sub Snapshots for your disaster recovery, we recommend that you take Snapshots `weekly` at a minimum to ensure that you do not lose any data in the event of a pipeline failure.

When you use Pub/Sub Seek to restart your data pipeline from a Pub/Sub snapshot, messages will be reprocessed. This creates a few challenges:

- First, you might observe duplicate records in your sync. The amount of duplication depends on how many messages were processed between the time of when the snapshot was taken, and the time the pipeline was terminated.
- In addition to that, data that has been read by your pipeline, but yet to be processed and written to sink will need to be processed over again. Remember that Dataflow acknowledges a message from Pub/Sub when it has read the message, not when the record has been written to the sink. This presents a challenge for pipelines with complex transformation logic. For example, if your pipeline is processing millions of messages per second and goes through multiple processing steps, having to reprocess the data represents a significant amount of lost compute.
- Lastly, if your pipeline is implemented exactly-once processing, windowing logic will be interrupted when you drain and restart your pipeline. Since you have to lose the buffered state when you drain your pipeline, you must conduct a tedious reconciliation exercise if exactly-once processing is a requirement for your use case.

## Dataflow snapshots strategy

Dataflow Snapshots can also be used for disaster recovery scenarios. Since Dataflow Snapshots saves streaming pipeline state, we can restart the pipeline without reprocessing in-flight data. This saves you money whenever you have to restart your pipeline.

Moreover, you can restore your pipeline much faster than using the Pub/Sub Snapshots and Seek strategy. This ensures that you have minimal downtime.

Dataflow Snapshots can be created with a corresponding Pub/Sub source Snapshot. This helps you coordinate the Snapshot of your pipeline with your source. In other words, you can pick up your processing where you left off when you restart the pipeline. This saves you the hassle of having to manage Pub/Sub Snapshots.

1. We can do Dataflow snapshots directly in the UI with the Create Snapshot button in the menu bar. You can also create a Snapshot using the command line interface.

2. Next, we need to stop and drain your Dataflow pipeline. This is also possible in both the UI and using the command line interface.

3. Lastly, we create a new job from the snapshot. This is accomplished by passing in the snapshot ID into a parameter when you deploy your job from your deployment environment.

Since Dataflow Snapshots, like its Pub/Sub counterpart, has a maximum retention of seven days, we recommend scheduling a coordinated Dataflow and Pub/Sub snapshot at least once a week. This means that if your pipeline goes down, you have a point in time in the past seven days from which you can restart processing, ensuring you can almost always avoid any data loss scenario. You can use Cloud Composer or Cloud Scheduler to schedule this weekly snapshot.

Snapshots are located in the region of their region job. When you create a job from a snapshot, you must launch the job in the same region. If a zone goes down, you can relaunch the job from a snapshot in a different zone in the same region.

However, Dataflow Snapshots cannot help migrate to a different region in the event of a regional outage. The best action to take in that event is to wait for the region to come back online or to relaunch the job in a new region without the snapshot. If you've taken a snapshot, though, you can ensure that your data is not lost.

## High availability

High availability is a hard requirement for some use cases.

If you are processing financial transactions or identifying cybersecurity threats in an event stream, there are very real external risks if your pipeline goes down.

When considering high availability, you need to take three factors into consideration:

- First: downtime. How much downtime can your operation tolerate without breaking business continuity? Many organizations define recovery time objectives, or `RTO` to articulate this upper link.
- Second: data loss. How much of your data can your application afford to lose in the event of an outage? IT managers will often use the term recovery point objectives or `RPOs` to describe this requirement.
- Third: cost. Running in a highly available configuration doesn't come for free, and it is important to consider how much your business is willing to pay to ensure that their data pipelines reach sufficient reliability standards.

Now that we've discussed the consideration, let's look at a couple of possible configurations on dataflow:

- Redudant configuration (√ Downtime. X Data loss. √ Cost)

You can choose to make redundant sources that are available in multiple regions. In this example architecture, you can maintain two independent subscriptions in two different regions that are reading from the same topic. If a regional outage occurs, you can start a replacement pipeline in the second region and have the pipeline consume data from the backup subscription. If a region goes offline, you can start a new pipeline in a different region immediately to continue processing.

Your application might drop data in the process as the intermediate data in the original pipeline will be dropped. However, you can replay the backup subscription to an appropriate time to keep data loss at a minimum if you're coordinating pub/sub snapshots between the two subscriptions.

Using a multi-regional sync can also ensure that your new pipeline will be able to write to the sync without degrading latency. Downstream applications must know how to switch to the running pipelines output. 

Since only the source data is duplicated, it is more cost efficient than other alternative high availability configurations.

- Redundant pipelines (√ Downtime. √ Data loss. X Cost)

If your application cannot tolerate data loss, run duplicate web pipelines in parallel in two different regions.

Your pipelines will consume the same data from two different subscriptions, process data using workers in different regions, and write to multi-regional sinks in each location.

This architectures provides geographical redundancy and fault tolerance.

Dataflow workers can only work in one zone per job. By running parallel pipelines in separate Google Cloud regions, you can insulate your jobs from failures that affect a single region.

Using multi-regional storage locations for your data syncs is not a requirement, but provides you one extra degree of fault tolerance.

Applications that feed from the process data sets must have a way to switch to the running pipelines output.

```md
[Global]                                [Regional]                  [Multi-region]
Pub/Sub                                 Dataflow                    BigQuery

Topic -> Subscription A (us-central1) -> PipelineA (us-central1)  -> Table (US)
      -> Subscription B (us-east1)    -> PipelineB (europe-west1) -> Table (EU)
```

This architecture basically offers you zero downtime, even where we'll have multiple instances of your pipeline running.

Similarly, as your data is being processed in multiple regions, data loss is extremely unlikely.

However, since you are duplicating resources across the entire stack, this approach is the most expensive high availability configuration.