# Troubleshooting

The general troubleshooting workflow involves two steps: first, checking for errors in the job, and second, looking for anomalies in the Job Metrics tab.

It is important to note that not all problematic jobs will be in a failed state, it is possible that certain problematic jobs are still in the running state.

For all Dataflow jobs, the CPU utilization graph is a good indicator of the parallelism in a job and can also indicate if a job is CPU-bound.

Types of failures:

- Failure building the pipeline
- Failure starting the pipeline on Dataflow
- Failure during pipeline execution
- Performance problems

## Failure building the pipeline

These types of errors occur while Apache Beam is building your Dataflow pipeline, and validating the beam aspects as well as the input/output specifications of your pipeline.

These errors can typically be reproduced using the direct runner and can be tested against using unit tests.

e.g.:

```text
prior to GroupBykey.
java.lang.IllegalStateException: GroupByKey cannot be applied to non-bounded Collection in the GlobalWindow without a trigger. Use a Window.into or Window.triggering transform

at org.apache.beam.sdk.transforms.GroupByKey.applicableTo (GroupByKey.java: 153)

at org.apache.beam.sdk.transforms.GroupByKey.expand (GroupByKey.java: 185)

at org.apache.beam.sdk .transforms. GroupByKey.expand (GroupByKey.java: 107)

at org.apache.beam.sdk.Pipeline.applyInternal (Pipeline.java:537)

at org.apache.beam.sdk.Pipeline.applyTransform (Pipeline.java:471)

at org.apache.beam.sdk.values.PCollection.apply (Collection. java:357)
at ...
```

## Failure starting the pipeline on Dataflow

Once the data flow service has received your pipelines graph, the service will attempt to validate your job.

This validation includes the following:

- Making sure the service can access your jobs associated cloud storage buckets for file staging and temporary output.
- Checking the required permissions in your Google Cloud project, making sure the service can access input and output sources such as files.

If your job fails the validation process, you'll see an error message in the data flow monitoring interface, as well as in your console or terminal window if you are using blocking execution.

**These errors cannot be reproduced with the direct runner**. They require the Dataflow runner and potentially the Dataflow service.

To iterate quickly and protect against regression, build a small test that runs your pipeline or a fragment of it.

## Failure during pipeline execution

These errors generally mean that the DoFns in your pipeline code have generated unhandled exceptions, which result in failed tasks in your Dataflow job.

Exceptions in your user code are reported in the Dataflow monitoring interface. You can investigate these exceptions using the general troubleshooting workflow described above.

Consider guarding against errors in your code by adding exception handlers.

For example, if you'd like to drop elements that fail some custom input validation done in a `ParDo`, handle the exception within your DoFn and drop the elements or return it separately.

You can also track failing elements in a few different ways:

- You can log the failing elements and check the output using cloud logging.
- You can check the Dataflow worker and worker startup logs for warnings or errors related to work item failures.
- And finally, you can have your ParDo write the failing elements to an additional output for later inspection.

**Important note**: It is important to note that batch and streaming pipelines have different behaviors and handle exceptions differently.
- In batch pipelines, the Dataflow service retries failed tasks up to four times.
- In streaming pipelines, your failed job may stall indefinitely. You will need to use other signals to troubleshoot your job. High data freshness, job logs, cloud monitoring, metrics for pipeline progress, and error accounts.

## Performance problems

Multiple factors such as pipeline design, data shape, interactions with sources, sinks, and external systems can affect the performance of a pipeline.

Use the user interface to identify expensive steps.

- Wall time for a step provides the total approximate time spent across all threads in all workers on the following actions: initializing the step, processing data, shuffling data, and ending the step.
- The input element count is the approximate number of elements that the step received, and the estimated size provides the total volume of data that was received.
- Similarly, the output element count is the approximate number of elements produced by the step, and the estimated size provides the total volume of data that was produced.
