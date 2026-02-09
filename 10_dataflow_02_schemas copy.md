# Dataflow: Develop pipelines - Schemas

A `PCollection` must consist of elements of the same type (for instance JSON objects, PlainText, Avro, protobuf).

To be able to enable distributed processing, Beam needs to be able to encode each individual element, for example, as a byte stream.

You can encode elements like:
- Byte arrays: `DoFn<byte[],...>`
- Proto, Avro, Bigquery objects: `DoFn<Transaction, ...>.withCoder`
- Custom serialization: `DoFn<Transaction, ...>.withCoder` (custom encoder/decoder).

Schemas describe a type in terms of fields and values.

```
Transactions
bank: STRING
transactionId: STRING
purchaseAmountCnts: LONG
```

Fields can have string names or be numerically indexed. There is a known list of primitive types a filed can have, like int, logn and string. Some felds can be marked as optional. Schemas can be nested arbitrarily, and can contain repeated or map fields as well.

In order to take advantaget of schemas, your PCollection must have a schema attached to it. Often the source itself will attach a schema to the PCollection. For example, when using `AvroIO` to read Avro files, the source can automatically infer a Beam schema from the average schema and attach that to the Beam PCollection:

```java
PCollection<Purchase> purchases = p.apply(
    PubSubIO.readAvros(Purchase.class).fromTopic(purchaseTopic));
```

However, not all sources produce schemas.

