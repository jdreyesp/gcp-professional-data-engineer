# Performance

We might sometimes underestimate simple considerations that are critical to a pipeline’s performance.

## Pipeline design

### Filtering data early might be considered one such option.

It is recommended to place transformations that reduce the volume of data as high up on the graph as possible. This includes placing them above window operations, even though the Window transform itself does nothing more than tag elements in preparation for the next aggregation step in the DAG. e.g. (Read - **Filter** - Window - GBK).

### Choose coders that provide good performance

For example, in the Java SDK, do not use `SerializableCoder`. Choose a more efficient coder, for example `ProtoCoder` or `Dataflow Schemas`.

Pro Tip: Encoding and decoding are a large source of overhead. Thus, if you have a large blob but only need part of it, you could selectively decode just that part. For example, `'com.google.protobuf.FieldMask'` in Protobufs enables reading specific bits of information without deserializing whole blob.

### Window considerations

If your pipeline has large windows aggregating large volumes of data, you can create smaller Window + Combine patterns before the main sliding window to reduce the volume of elements to be processed when the window slides.

### Fusion

Runners may support fusion as part of graph optimization. Such optimizations can include **fusing multiple steps or transforms in your pipeline's execution graph into a single phase**.

Before we discuss graph optimization further let’s first understand what a `fanout` transformation is: In a `fanout` transformation a single element can output hundreds or thousands of times as many elements. (1 input element -> N output elements) (e.g. explode operations, split operations, cartesian product, cross join, generating time series from a single event, tokenizing, etc.)

When fusion occurs, the fanout and downstream transforms run on the same worker. If one input produces thousands of outputs, all of them must be kept in memory on that worker before downstream processing, which can cause OOM or performance issues.

Preventing fusion lets Dataflow distribute the fanout outputs across multiple workers. Without fusion, the fanout can run on one worker, and downstream transforms can process the outputs in parallel on other workers.

The primary example where fusion is **not** desirable is a large fanout transform. To prevent Fusion you can insert a `Reshuffle` after your first `ParDo` (The Dataflow service never fuses `ParDo` operations across an aggregation).

Alternatively you can pass your intermediate PCollection as a side input to another ParDo (The Dataflow service always materializes side inputs, so that will prevent fusion).

### Logging

- Strike a balance between excessive logging and no logging at all.
- Avoid logging at info level against PCollection element granularity.
- Log.error should be also carefully considered: Using a Dead Letter pattern followed by a count per window (e.g. 5 minutes) for reporting data errors could be a better approach.

## Data shape

### Data skew

Operations like `groupByKey` merge multiple `PCollections` into one. During a GroupByKey or combine operation, keys will be shuffled to workers. All values related to one key will be sent to the same machine throughout the process. This can be a problem when the data you are processing is skewed.

For example, columns used as keys that are `@nullable` often end up being hot keys.

To mitigate the hot key issue, we can use one of the following three techniques:

- The first one is to use the helper API `withFanout(int)`. This allows for the **definition of intermediate workers** before the final combine step.
- Another similar API is `withHotKeyFanout(Sfn)`: It is available for Combine.perkey and allows for a function to determine intermediate steps.
- Using Dataflow Shuffle service for batch or streaming also alleviates this issue. The Dataflow shuffle or streaming service offloads the shuffle operation to a backend service. This means that shuffle operation is not constrained by resources available on a single worker machine.

The Dataflow service makes it easier to detect and surface hot keys. To do so, set `hotKeyLoggingEnabled` flag to true. Enabling this flag will print the specific key that is your bottleneck, which can help Dataflow developers to implement custom logic for that specific key. Without the flag, Dataflow will print if they think they've detected a hot key, but cannot reveal what that key is.

Key space used in your pipeline also has an impact on its performance. For example, **the maximum amount of parallelism is determined by the number of keys** (more machines will not be able to do any more work if key space is limited):

- Too few keys is bad for performance. Limited keyspace will make it hard to share workload, and per-key ordering will kill performance.
- Too many keys can be bad too as overhead starts to creep in. If the key space is very large, consider using hashes separating keys out internally.

This is especially useful if keys carry date/time information. In this case you can "re-use" processing keys from the past that are no longer active, essentially for free.

*Student note*: Explanation with good words, since this previous explanation is **SO VAGUE**:

The Problem Without Date/Time Keys:

If keys don't include date/time:

- You might have an ever-growing set of unique keys, so key space will have a lot of keys (Bad for performance!!)
- Example: user_123, user_456, user_789... (thousands of users)
- All these keys remain active indefinitely (because new data is using the same keys, so they never expire!)
- Overhead grows with the number of unique keys (key tracking, hash partition management, routing metadata, and per-key state if using stateful operations).

The Benefit With Date/Time Keys:

With date/time keys like "2024-01-15":

- Old keys naturally become inactive (no more data arrives for that date, ever!, so the key expires)
- The hash partition for "2024-01-15" becomes free, so it can host a new key!
- New date/time keys can reuse that same hash partition
- The active key space stays bounded (e.g., only last 30 days active)

Why This Matters

The overhead is about:

- Bounded key space: Date/time keys naturally limit how many keys are active at once
- Hash partition reuse: Same partitions can handle new date/time keys
- Reduced tracking overhead: System doesn't need to track thousands of permanently active keys

*Student note:* Hope this explanation is way better than the slides one...

Pro tip! If windows are distinct, the window can be added as part of the key to shard work across more workers. Adding the window to the key improves the ability of the system to parallelize processing since those keys can now be processed in parallel on different machines since they are now recognized as unrelated.

### Sources, Sinks & external systems

In the Dataflow service, more sources and sinks abstract the user from the need to deal with Read stage parallelism. Sometimes, this hides underlying issues that impact a pipeline's performance.

For example, if you're reading `gzip` files via `TextIO`, `gzip` files **can't be read in parallel**. A single thread will deal with each file (3 problems: `First` one is that only one machine can do the read I/O operation, `secondly`, after the read stage, all fused stages will need to run on the same worker that read the data, and `last but not least`, in any shuffle stage, a single machine will need to push all the data from the file to all other machines.). This single host network becomes the bottleneck. => Solution: Switch to uncompressed files while using TextIO, or switch to compressed Avro format.

Sinks: Beam runners are designed to be able to rapidly chew through parallel work They can spin up many threads across many machines to achieve this goal. This can easily swamp an external system (backpressure). This is an issue for both batch and streaming pipelines => To alleviate this issue, make use of a batch mechanism in the call to external system and use a mechanism, like `GroupIntoBatches`, `transforms`, or `@StartBundle` and `@FinishBundle`.

Colocation:

While working in Cloud, it's sometimes easy to forget the impact of the simple choices we make while developing applications. Colocation is one such aspect. Using services and resources from same region usually means relatively lower latency for interservice communication. This lower latency may result in significant performance gains, especially when the pipeline involves significant interaction with actual analysis services like BigQuery, Bigtable, or any other service outside of Dataflow.

### Shuffle & Streaming Engine

Dataflow Shuffle is the base operation behind Dataflow transforms such as GroupByKey, CoGroupByKey, and Combine. The Dataflow Shuffle operation partitions and groups data by key in a scalable, efficient, fault-tolerant manner.

Currently, Dataflow uses a shuffle implementation that runs entirely on worker virtual machines and consumes worker CPU, memory, and persistent disk storage.

The service-based Dataflow Shuffle feature, available for `batch pipelines only`, moves the shuffle operation out of the worker VMs and into the Dataflow service backend.

The `service-based Dataflow Shuffle` has the following benefits: 

- Faster execution time of batch pipelines for the majority of pipeline job types.
- A reduction in consumed CPU, memory, and persistent disk storage resources on the worker VMs.
- Better autoscaling, since VMs no longer hold any shuffle data and can therefore be scaled down earlier.
- Better fault tolerance.

`Dataflow Shuffle` and the `Streaming Engine feature` offloads the window state storage operation from the persistent disks (`PDs`) attached to workers, to a backend service. It also implements an efficient shuffle for streaming cases. (the `Dataflow Shuffle service` is applicable to **batch pipelines**, while the `Streaming Engine service` is built for **streaming pipelines**). No code changes are required to get the benefits of these features.

Many scalability and autoscaling issues can be resolved by enabling Shuffle and Streaming Engine for your batch and streaming pipelines, respectively.