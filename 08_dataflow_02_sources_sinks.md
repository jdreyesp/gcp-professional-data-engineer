# Dataflow: Develop pipelines - Sources and sinks

Input: Sources
Output: Sinks

- Sources generally appear at the beginning of a pipeline (but it's not always the case)
- Sinks are PTransforms that performs a write to the specified destination
    - A `PTransform` is an operation that takes an input and provides an output
    - A `PDone` is a common output for a sink, which signals that the branch of the pipe is done.

Types of sources:
- Bounded source: Dataflow splits the work of reading into smaller chunks, known as `bundles`. It can provide estimates of progress to the service and number of bytes to be processed (since there is a known start and end). Dataflow can track if the units of work (bundles) can be broken down into smaller chungs for dynamic work rebalancing, and carry out the `split` operation if needed.

- Unbounded source: Unbounded amount of input (streaming). A `checkpoint` allows for the ability to bookmark where the data has been read in the source. `Watermarks` from sources can provide the point in time estimates for a piece of data. Some unbounded sources, can use `Record Ids` from it for deduplicating the data. Dataflow will keep track of the message Ids for 10 minutes and automatically discard the record if duplicated.

Types of sinks:
- Data sinks: Writes data to end systems

There's a lot of opensource IO's in Apache Beam.

## TextIO & FileIO

```python
pcoll = (pipeline
    | 'Create' >> Create([file_name])
    | 'ReadAll' >> ReadAllFromText()) # Read method

pcoll2 = pipeline | 'Read' >> ReadFromText(file_name) # Read method (not the initial action even if it's a source, as we described before)
)
```
FileIO uses pattern matching for grabbing a range of files, e.g:

```python
with beam.Pipeline() as p:
    readable_files = (
        p
        | fileio.MatchFiles('hdfs://path/to/*.txt') # <-- Match file pattern
        | fileio.ReadMatches()
        | beam.Reshuffle())
    
    files_and_contents = (
        readable_files
        | beam.Map(lambda x: (x.metadata.path, # <-- Access file metadata
                              x.read_utf8()))
    )
```

Java example of processing files as they arrive:

```java
p.apply(
    FileIO.match()
    .filePattern("...")
    .continuously(
        Duration.standardSeconds(30),
        Watch.Grow.afterTimeSinceNewOutput(Duration.standardHours(1))
    )
    // so this will continuously monitor files, every 30 seconds for 1hour
)
```

Another example using Pub/Sub:

```python
with beam.Pipeline() as p:
    readable_files = (
        p
        | beam.io.ReadFromPubSub(...)
        # parse pubsub message and yield filename
    )

    files_and_contents = (
        readable_files
        | ReadAllFromText()) # <-- used parsed filename to read
```

related to FileIO, `ContextualTextIO` is able to return things such as ordinal position or read multi-line CSV records.

### TextIO writing

Simple example:

```python
transformed_data
| 'write' >> WriteToText(known_args.output, coder=JsonCoder())
```

Writing with dynamic destinations:

```java
PCollection<BankTransaction> transactions = ...;

transactions.apply(FileIO.<TransactionType, Transaction>writeDynamic() //dynamic dest.
.by(Transaction::getTypeName)
.via(tx -> tx.getTypeName().toFields(tx),
    type -> new CSVSink(type.getFieldNames()))
    .to(".../path/to")
    .withNaming(type -> defaultNaming(type + "-transactions", ".csv"))) //write dynamic filename
```

## BigQueryIO

It runs queries against BigQuery and map results. Example:

```java
PCollection<Double> maxTemperatures =
    p.apply(
        BigQueryIO.read(
            (SchemaAndRecord elem) -> (Double)
                elem.getRecord (
                .get("max_temperature"))
            .fromQuery (
                "SELECT max_temperature FROM
                'clouddataflow-readonly.samples.weather_stations'")
            .usingStandardSql()
            .withCoder(DoubleCoder.of()));
```

When using standard SQL, Dataflow will submit the query, and first retrieve metadata. BQ will then export the results to a temp location in GCS, and then Dataflow will read the contents from there.

There are multiple ways to specify schema, but using the built-in schema functionality is quick and efficient way to specify it.

## Pubsub IO

```java
pipeline.apply("Read Pubsub Messages",
PubsubIO.readStrings().fromTopic(options.getInputTopic())) // this will create the subscription automatically when the Dataflow is deployed, and destroyed upon termination of the job
.apply(
    Window.into(
        FixedWindows.of(
            Duration.standardMinutes(options.getWindowSize())
        )
    )
)
```

Capturing failures is possible by defining a DLQ:

```java
//...
appliedUdf.get(KafkaPubsubConstants.UDF_DEADLETTER_OUT)
.apply(...)
//...
.apply("writeFailureMessages",
PubsubIO.writeStrings().to(options.getOuptputDeadLetterTopic()));
```

## KafkaIO

Unbounded source for streaming.
You're able to use checkpoints with Kafka to bookmark your read so you can resume where you left off.

`withTopics(...)` allows you to subscribe to multiple topics.

KafkaIO is written in Java, but Beam uses Cross-language transforms (you can use it in Python code).

## BigTableIO

Scalable, noSql tabase service by google.

```java
ByteKeyRange keyRange = ...;
p.apply("read", 
BigTableIO.read()
.withProjectId(projectId)
.withInstanceId(instanceId)
.withTableId("table")
.withKeyRange(keyRange)) //<- useful for prefix scan on the index to quickly arrive at the desired prefix
```

There are times where you will want to continue on a pipeline after a sink has completed. BigtableIO has this ability by triggering the `Wait.on` function in Beam by sending a signal to the Wait function. This enables you to continue with an additional transformation after a write has completed.

## AvroIO

Avro is a file format that is self-describing.

AvroIO allows you to read and write to that file type. Avro provides the schema and the data so the files can be self-describing.

You can use the built-in functions from AvroIO to retrieve the schema into your Beam pipeline.

You can also use wild cards to read multiple files as seen in this Java example:

```java
PCollection<AvroAutoGenClass> records = p.apply( // read avro schema
    AvroIO.read(AvroAutoGenClass.class)
          .from( "gs:..*.avro")) ;

Schema schema = new Schema.Parser() // read avro schema
.parse (new File(" schema. avsc"));

PCollection<GenericRecord> records =
p.apply(AvroIO.readGenericRecords(schema)
.from("gs:...-*.avro"));
```

## Splittable DoFn

Splittable Do Functions are a generalization of a Do Function that gives it core capabilities of a source, splittability, and the ability to report back metrics such as progress (Progress and other metrics then allow you to know how far along a bundle is and how far it has to go). This enables the ability for the work to be split into multiple bundles.

`InitialRestriction`: Represents the "what" — what work needs to be done for this element. Describes the work to be done.
Examples: byte range [0, 1000], key range ["a", "z"], row range [0, 10000]

`RestrictionTracker`: Tracks progress through the restriction. Claims work units (prevents duplicate processing). Key methods: `tryClaim(position)` -> Claims the next work unit, `getProgress()`: Reports current progress (0.0 -> 1.0), `splitAtFraction(fraction)`: Splits the restriction at a given fraction.