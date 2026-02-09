# Dataflow: Develop pipelines - Best practices

## Schemas

Schemas allows the DataFlow service to make optimizations behind the scenes as it is aware of the type and structure of data being processed. For example, the DataFlow service optimizes the encoder and decoder required for serialization and deserialization of data as it moves from one phase to another.

Schema example:

```java
@DefaultSchema(JavaBeanSchema.class)
public class Purchase {
    public String getUserId();
    public long getItemId();
    public ShippingAddress getShippingAddress();
    public long getCostCents();
    public List<Transaction> getTransactions();

    @SchemaCreate
    public Purchase(String userId, long itemId, ShippingAddress shippingAddress, long costCents, List<Transaction> transactions) {
        //...
    }
}
```

```python
class Purchase (typing. NamedTuple):
    user_id: str # The id of the user who made the purchase.
    item_id: int # The identifier of the item that was purchased.
    shipping_address: ShippingAddress # The shipping address, a nested type. cost_cents: int # The cost of the item
    transactions: typing.Sequence[Transaction]
```

## Handling un-processable data

Handling un-processable data from a `DoFn` can be handled via a DLQ hosted in BigQuery or GCS.

```
final TupleTag successTag;
final TupleTag deadLetterTag;

PCollection input = /* ... */;

PCollectionTuple outputTuple = input.apply(ParDo.of(new DoFn() {
    @Override
    void processElement (ProcessContext ctxt) {
        try {
            c. output (process (c.element)) ;
        } catch (MyException ex) {
            // Optional Logging at debug level
            c. sideOutPut(deadLetterTag, c.element) ;
        }
    }
})).writeOutPutTags(successTag, TupleTagList.of(deadLetterTag));

// Write dead letter elements to separate sink
outputTuple.get(deadLetterTag). apply(BigQuery write(...));

// Process the successful element differently.
PCollection success = outputTuple.get(successTag);
```

## Error handling

Always use try-catch block around activities like parsing data.

In the exception block, send the erroneous records to a separate sink, instad of just logging the issue.

Use tuple tags to access multiple outputs from the PCollection.

## AutoValue code generator (Java only)

In cases where Apache Beam schemas are not possible in some operations, where POJOs are needed, for example, when dealing with key value objects or handling the object state, use `AutoValue` (instead of manually creating those POJOs).

This ensures that the necessary overrides are covered and lets you avoid the potential errors introduced by hand-coding.

AutoValue is heavily used within the Apache Beam code base, so familiarity with it is useful if you want to develop Apache Beam pipelines on dataflow using Java SDK.

AutoValue can also be used in concert with Apache Beam schemas if you add on @DefaultSchemas annotation.

## JSON data handling

- `JSONToRow` is a good operation for converting JSON strings to POJOs.
- To convert JSON strings to POJOs using `AutoValue`, register a schema for using `@DefaultSchema(AutoValueSchema.class)` annotation, then use the Convert utility.
- Use Deadletter pattern for processing unsuccessful messages

Example:
```java
PCollection<String> json = ...

PCollection<MyUserType> col = json
    .apply("Parse JSON to Beam Rows", JsonToRow.withSchema(expectedSchema))
    .apply("Convert to a user type with a compatible schema registered", Convert.to(MyUserType.class))
```

## Utilize DoFn lifecycle

While working on big data use cases, it is easy to overwhelm an external service endpoint if you make a single call for each element flowing through the system, especially if you haven't applied any reducing functions.

If you recall, the lifecycle of a DoFn looks like:

@Setup -> @StartBundle -> @ProcessElementOnTimer -> @FinishBundle -> @Teardown
                            |-> read state objects   |-> write state objects

You can use StartBundle and FinishBundle to instantiate (and terminate) your external services or dependencies. 

**Don't forget that StartBundle and FinishBundle can be called multiple times for a single bundle.**, so it's important to reset variables accordingly

## Pipeline optimizations

1. FIlter data early in the pipeline whenever possible
2. Move any steps that reduce data volume up in your pipeline
3. Apply data transformations serially to let Dataflow optimize DAG. When transformations are applied serially, hey can be merged together in single stage, enabling them to be processed in the same worker nodes and reducing costly IO network operations.
4. If your pipeline interacts with external systems, look out for back pressure and external systems. May be a key value store like BigTable or HBase used for lookups in a pipeline or a sink your pipeline writes to. It is recommended that you ensure the appropriate capacity of external systems to avoid back pressure issues.
5. Enable autoscaling to downscale if workers are underutilized.