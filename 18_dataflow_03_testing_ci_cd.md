# Testing & CI/CD

## Overview

Dataflow pipelines need a comprehensive testing strategy.

So we should be implementing unit tests, integration tests, and end-to-end tests to ensure that our pipeline behaves as we expect.

A haphazard rollout can result in corrupted data being written to the sink or disruptions to your downstream applications.

Finally, data engineers should strive to validate changes made to pipeline logic, and have a rollback plan if there is a bad release.

While all these considerations are similar to general application development, there are some key differences to point out:

- Data pipelines often aggregate data, and this makes them stateful, in that they must accumulate the result of some aggregation over time. This means that if you need to update your pipeline, you need to consider any state that may exist in the pipeline you're updating.

- When you change your pipeline, you'll need to account for existing state, as well as any changes to the pipeline logic and topology. 

- If your pipeline makes non-idempotent side effects to external systems, you will have to account for those effects after a rollback.

Unit tests:

All pipelines revolve around transforms, and the lowest level we typically deal with in Beam is the DoFn. Since these are essentially functions, we validate their behavior with unit tests that operate on input datasets. They produce output datasets that we validate with assertions.

Similarly, we can provide test inputs to the entire pipeline, which might contain our DoFns as well as other PTransforms and DoFn subclasses. We also assert that the results of the entire pipeline are what we expect.

Integration tests:

For system integration tests, we incorporate a small amount of test data using the actual I/Os. This should be a small amount of data, since our goal is to ensure the interaction with the IOs produces the expected results.

End-to-end tests:

Finally, end-to-end tests use a full testing dataset, which is more representative of the data our pipeline will see in production.

## Runners

Whatever tooling you're using in your CI/CD testing environment, you'll make use of the `Direct Runner`, which runs on your local machine, and your `Production runners`, which run on the cloud service of your choice, like Dataflow.

The Direct Runner will be used for local development, unit tests, and small integration tests with your data sources.

You'll use your production runner when it's time to do larger integration tests, when you want to test performance, and when you want to test pipeline deployment and rollback.

In the development part of the lifecycle, we write our code, executing unit tests locally using the `Direct runner` and executing integration tests using the `Dataflow runner`. As we develop and test, we're committing to source repositories along the way. These commits and pushes trigger the continuous integration system to compile and test our code in an automated manner, using `Cloud Build` or a similar CI system.

Once the builds complete successfully, artifacts are deployed, first to a `preproduction environment` where end-to-end tests are run. If these succeed, we deploy to our `production environment`.

## Unit testing

Unit tests are fundamental to all software development, and beam is no exception.

We use unit tests in Beam to assert behavior of one small testable piece of your production pipeline. These small portions are usually either individual `DoFns` or `PTransforms`.

These tests should complete quickly, and they should run locally with no dependencies on external systems (FIRST principle). To get started with unit testing in Beam Java pipelines, we need a few dependencies: Beam uses JUnit 4 for unit testing, and hamcrest is also use for assertion expresiveness.

Test pipeline is a class included in the beam SDK specifically for testing transforms. So when writing tests use TestPipeline in place of Pipeline where you would create a Pipeline object:

```java
@Rule
public final transient TestPipeline p = TestPipeline.create();
```

```python
with TestPipeline as p:
    #...
```

Unlike Pipeline.create, TestPipeline.create handles setting pipeline options internally.

`PAssert` is another class provided as part of Beam that lets you check the output of your transforms. These assertions can be checked no matter what kind of pipeline runner is used. PAssert becomes part of your pipeline alongside the transforms. PAssert works on both local and production runners:

```java
@Test
@Category(NeedsRunner.class)
public void myPipelineTest() throws Exception {
    final PCollection<String> pcol = p.apply(...)
    PAssert.that(pcol).containsInAnyOrder(...);
    p.run();
}
```

```python
from apache_beam.testing.util import assert_that
from apache_beam.testing.util import equal_to

output = ...

# Check whether a PCollection contains some elements in any order
assert_that(
    output,
    equal_to(["elem1", "elem3", "elem2"])
)
```

Unit tests lets you provide known input to your `DoFns` and composite transforms, then compare the output of those transforms with a verified set of expected outputs.

The Apache Beam SDK provides a JUnit rule called `TestPipeline` for unit testing individual transforms like your DoFns subclasses, composite transforms, like your PTransform subclasses, and entire pipelines.

Beam provides a `Create.timestamped(TimestampedValue.of("a", new Instant(0L)))` method which can be used to create timestamped elements in a testing PCollection. You can manipulate the timestamp directly as we do in this example by adding the window duration to the timestamp of the last element.

Then here:
```java
@Test
@Category(NeedsRunner.class)
public void testWindowedData() {

    PCollection<String> input = 
        p.apply(Create.timestamped(
            TimestampedValue.of("a", new Instant(0L))
            TimestampedValue.of("a", new Instant(0L))
            TimestampedValue.of("b", new Instant(0L))
            TimestampedValue.of("c", new Instant(0L))
            TimestampedValue.of("c", new Instant(0L).plus(WINDOW_DURATION))
        ).withCoder(StringUtf8Coder.of()));

    PCollection<KV<String, Long>> windowedCount = input.apply(Window.into(FixedWindow.of(WINDOW_DURATION)).apply(Count.perElement()));

    PAssert.that(windowedCount)
        .containsInAnyOrder(
            //output from first window
            KV.of("a", 2L),
            KV.of("b", 1L),
            KV.of("c", 1L),
            // output from second window
            KV.of("c", 1L)
        );

    p.run();
}
```

you can see that we apply fixed windows of window duration and perform a count on the windowed elements. We can then assert that the resulting PCollection contains the windowed calculations we expect.

Note that windowing takes place in both batch and streaming pipelines. Testing how your window transforms behave is useful in both types of pipelines.

### `TestStream` (for streaming pipelines)

Test stream is a testing input that generates an unbounded `PCollection` of elements advancing the watermark and processing time as elements are emitted. After all of the specified elements are emitted, test stream stops producing output.

Each call to a TestStream.builder method will only be reflected in the state of the pipeline after each method before it has completed and no more progress can be made by the pipeline. The pipeline runner must ensure that no more progress can be made in the pipeline before advancing the state of the test stream.

Example:

```java
@Test
@Category(NeedsRunner. class)
public void testDroppedLateData() {
    TestStream<String> input = 
        TestStream.create(StringUtf8Coder.of())
                .addElements(
                        TimestampedValue.of("a", new Instant(OL)), 
                        TimestampedValue.of("a", new Instant (OL)), 
                        TimestampedValue.of ("b", new Instant (OL)),
                        TimestampedValue.of("c", new Instant (OL).plus(Duration.standardMinutes(3))))
                .advanceWatermarkTo(new Instant(OL).plus(WINDOW_DURATION).plus(Duration.standardMinutes(1))) // You can manipulate the timestamps in the TestStream by adjusting the timestamp object and manipulate the position of the watermark using an instant object.
                .addElements(TimestampedValue.of("c", new Instant (OL)))
                .advanceWatermarkToInfinity(); // Advancing the watermark to infinity closes all windows so that you can perform your windowed calculation and assert on the result.

    Collection<KV<String, Long>> windowedCount = ...

    PAssert.that(windowedCount).containsInAnyOrder(
        KV.of("a", 2L),
        KV.of("b", 1L).
        KV.of("c", 1L));

    p.run();
}
```

Test stream is supported by the direct runner and the Dataflow runner. Use both to carry out your streaming pipeline tests.

## Integration testing

Let's say that an actual pipeline reads from two data sources and writes to BigQuery. Integration tests create a smaller amount of data, and assert that the output of the transforms are what we expect.

For large integration tests, we work with data on closer to a production scale. To do this, we can clone data from a production project to a non production project.

For bigquery we can copy the production data into non-production by using the `storage transfer service`, for instance.

For Pub/Sub we can copy the data production data into non-production data by creating a new testing subscription.

etc.

In integration tests, we typically test the entire pipeline without sources and sinks.

## Artifact building

How we should package Apache Beam with the rest of our pipeline artifacts:

Apache Beam uses semantic versioning. Version numbers use the form major dot minor dot incremental and are incremented as follows:

- Major versions are incremented for incompatible API changes
- Minor versions are incremented for new functionality added in a backward compatible manner
- Finally, the incremental version is incremented for forward compatible bug fixes.

Dependencies:

Core Java SDK:

```xml
<groupId>org.apache.beam</groupId>
<artifactId>beam-sdks-java-core</artifactId>
```

Dataflow runner:

```xml
<groupId>org.apache.beam</groupId>
<artifactId>beam-runners-google-cloud-dataflow-java</artifactId>
```

IOs:

```xml
<groupId>org.apache.beam</groupId>
<artifactId>beam-sdks-java-io-cloud-platform</artifactId>
```

We recommend that you use beam 2.26 and higher Versions from 2.226 use the Google Cloud libraries pom.

## Deployment

We’ll look at three distinct stages of the pipeline lifecycle: `deployment`, `in-flight actions`, and `termination`.

### Pipeline lifecycle - Deployment

There are two ways to deploy your Dataflow job:

- We can use a direct launch, in which we run the pipeline directly from the development environment. For Java pipelines this means running the pipeline from Gradle or Maven. For Python, this means running the python script directly. This method can be used for both batch and streaming pipelines.

- We can also use templates. Templates allow us to launch a pipeline without having access to a developer environment. Templates are built and deployed as an artifact on Cloud Storage, and can be used for both batch and streaming pipelines.

If you're using an external scheduler, like Airflow, you'll be able to use Airflow's built-in support for Dataflow, which calls a template when invoked.

**You might notice that Dataflow SQL is not on this list. Dataflow SQL is a special case of a templated deployment—it’s actually a user interface built on top of a flex template.**

--

When we're deploying our pipeline for the first time, we need to submit the pipeline to the Dataflow service with a unique name. This name will be used by the monitoring tools when you look at the monitoring page in the console.

### Pipeline lifecycle - In-flight actions

These actions are only available to `streaming` pipelines, since batch jobs can simply be relaunched.

As a streaming pipeline processes data, it accumulates state.

It's useful to have ways to preserve the state so that we can manage changes to our pipeline without the risk of permanent data loss.

We can use snapshots for this.

With snapshots, we can save the state of an executing pipeline before launching a new pipeline.

This way, if we need to roll our pipeline back to a previous version, we can do that without losing the data processed by the version being rolled back.

Since streaming pipelines are long-running applications, we’re likely to need to modify our pipeline from time to time.

Now that we’ve saved the state of the running pipeline with a snapshot, we can safely update it.

When you update a job on a Dataflow service, you replace the existing job with a new job that runs your updated pipeline code. The Dataflow service retains the job name, but runs the replacement job with an updated job ID.

If for any reason you are not happy with how the replacement job is running, you can roll back to the prior version by creating a job from a snapshot.

#### Snapshots

Dataflow snapshots are useful for several scenarios. Dataflow Snapshots provides a copy of intermediate state of your pipeline at the moment the snapshot is taken. You can use that snapshot to validate a pipeline update, or use it as checkpoint for you to roll back your pipeline in the event of an unhealthy release.

You can also use Snapshots for backups and recovery use cases.

Lastly, Snapshots create a safe path for migrating pipelines to Streaming Engine.

If you want to reap the benefits of smoother autoscaling and superior performance, you can take a snapshot and create a job from that snapshot. The new job will run on Streaming Engine. The flip side of this is that jobs created from Snapshots cannot be run with Streaming Engine disabled.

If you are using Pub/Sub, creating a snapshot with source will allow you to create a coordinated snapshot between your unread messages and accumulated state. This makes it easier to roll back your pipeline to a known point in time.

Depending on how much state is buffered, it could take a matter of minutes. We recommend planning to take snapshots during periods of the day when latency can be tolerated, such as non-business hours.

You can also create snapshots using the CLI or API.

To create a job from a snapshot:

- We have to enable Streaming Engine with the --enableStreamingEngine flag.
- Secondly, we pass in the Snapshot ID into the createFromSnapshot parameter.

If you are creating a job from the snapshot for a modified graph, the new graph must be compatible with the prior job.

Now that we’ve snapshotted our pipeline, we’re ready to update our pipeline.

#### Update

There are various reasons why you might want to update your Dataflow job: 

- One is to enhance or otherwise improve your pipeline code.
- Another is to fix bugs in your pipeline code.
- You might also want to update your pipeline to handle changes in the data format.
- Finally, you might want to change your pipeline to account for version and other changes in your data source.

To update your pipeline, you’ll need to do a couple of things:

- First, you need to pass the "update" and "jobName" options when you submit the new pipeline. You’ll have to set jobName to the name of the existing pipeline, or else the old job will not be replaced. This tells Dataflow that you're updating the job, rather than deploying a new pipeline.
- Second, if you added, removed, or changed any transform names, you'll need to tell Dataflow about these changes by providing a transformNameMapping. The replacement job will preserve any intermediate state data from the prior job.

Note, however, that changing the windowing or triggering strategies will not affect data that's already buffered or already being processed by the pipeline.

"In-flight" data will still be processed by the transforms in your new pipeline. Additional transforms that you add in your replacement pipeline code may or may not take effect, depending on where the records are buffered.

Updates can also be triggered via the API. 
This can enable continuous deployment contingent on other tests passing. When you update your job, the Dataflow service performs a compatibility check between your currently running job and your potential replacement job. The compatibility check ensures that things like intermediate state information and buffered data can be transferred from your prior job to your replacement job.

Let’s review the most common compatibility breaks:

- Modifying your pipeline without providing a transform mapping will fail the compatibility check.
- Adding or removing side inputs will also cause the check to fail.
- Changing coders. The Dataflow service isn’t able to serialize or deserialize records if your updated job uses different data encoding.
- Running your job with a new zone and a new region will also cause your compatibility check to fail. Your replacement job must run in the same zone in which you ran your prior job.
- Lastly, removing stateful operations. Dataflow fuses multiple steps together for efficiency, but if you’ve removed a state-dependent operation from within a fused step, the check will fail.

If your pipeline requires any of these changes, we recommend draining your pipeline, then launching a new job with the updated code.

### Pipeline lifecycle - Termination

Now that we’ve discussed actions you can take on your streaming pipeline, we’ll discuss two ways that you can terminate your pipeline.

- First, we start with drain. Selecting drain will tell the Dataflow service to stop consuming messages from your source and finish processing all buffered data. After the last record is processed, the Dataflow workers are torn down. This action is only applicable to streaming pipelines.

- Secondly, we can cancel the job. Using the Cancel option ceases all data processing and drops any intermediate, unprocessed data. We can cancel both batch and streaming jobs.

Drain: 
√ No data is lost. 
X All windows are closed. Closing all the windows in this way will result in incomplete aggregations, since draining the pipeline will not wait for open windows to be closed before stopping pulling from the source system.

Pro-tip: Use Beam PaneInfo object to identify & filter incomplete windows.

Cancel:
√ Dataflow will immediately begin shutting down the resources associated with your job. Easy for non-mission critical workloads.
X Data is lost. The pipeline will stop pulling and processing data immediately, which means you may lose any data that was still being processed when the pipeline was canceled. If your use case can tolerate data loss, then cancelling your job will fit your purpose.