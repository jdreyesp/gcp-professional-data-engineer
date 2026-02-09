# Dataflow: Develop pipelines - Windows, Watermarks & triggers

## Windows

Batch pipelines' data is disposed in big batches that are processed for instance, once a day.

If you need to process continuous streams of data, you're dealing with streaming pipelines, that are processing batches under the hood, but artificial batches. 

One of the problems you have to deal with when processing data in streaming is **the lack of order**.

Let's say you have an application and you receive three 8:00 events:

- 1st. at 8:00 sharp. That user was connected to the internet and your application processed the event immediately.
- 2nd. at 9:00. That user was in a subway, and when he reached the surface, the event was sent by the application.
- 3rd. at 18:00. That user was in a plane, in flight mode he created the event, but the application did not send the event to be processed until he reached his destination.

How can you deal with out of order data and how can you split it? **Windows**.

Windows are required when doing aggregations over unbounded data using Beam primitives (GroupByKey, Combiners) (you can also do aggregations using state and timers).

3 window types: fixed, sliding and session
- fixed: consistent non-verlapping intervals.
- sliding: may overlap. Smoother transitions, and values (imagine you work with user clicks => since windows overlap, average is smoother (100,150,50) converts into (100, 110, 105)), but in fixed windows would be (100, 150, 50) (which is less smooth)
- session: session based windows. Its gap is updated when new events arrived for the same session window. If the event > gap => New session window is created for that gap.

**Watermark**: When is the window goign to emit the results?

How can you decide when the window can be closed? 

The watermark is the relation between the `processing time` and the `event time`.

Any message that arrives before the watermark is said to be `early`.
If it arrives at the watermark is said to be `on time`.
If it arrives late is said to be `late data`.

The watermark cannot be calculated, because it depends on the data that arrives. So the data flow estimates it, as the **oldest timestamp waiting to be processed**. This estimation is continuously updated with the new data that arrives. So whatever data comes, it's considered early, on time or late because of the watermark.

Watermark advances based on new incoming data. If watermark > window time, then the window is closed. For data that belongs to a window that it's already closed (since the watermark is >= window end) it's considereed late data. The default behaviour for late data is `drop data`, but triggers can make data to be emitted again.

In Dataflow, the `Data freshness` dashboard represents the amount of time between real time and the output watermark.
`System latency` measures the time that the pipeline fully process a message. This can be useful to determine if the data freshness is not updated because the pipeline takes too much time to process incoming data.

If `System latency` and `Data freshness` monotonically increase indefinitely, it means that the processing of complex objects is too high

If `System latency` is constant, and `Data freshness` monotonically increase indefinitely, it means that there are too many message (like a peak) in the input (because watermak calculation can't cope with the high amount of incoming data). In this scenario, Dataflow will autoscale, increasing the number of workers to process this additional data input.

`Triggers` are used to define when you want to see the results of your windows (sometimes windows can idle since watermark is never updated because of no new incoming data, for instance).

Triggers can be defined from:
- Event time: Emit results after 30 seconds measured by the message timestamps
- Processing time: Emit results based on processing time (e.g. every 30 seconds as measured by the worker's clock).
- Data-driven: E.g. emit window output after we get 25 messages.
- Composite: Combination of any of the triggers above.

By default the trigger `AfterWatermark` is defined. AfterWatermark is an event type trigger.
You can also have `AfterProcessingTime` (so the worker clock is who decides when to trigger).
You can also have the data-driven trigger `AfterCount` (after 25 messages come, then triggers the window).
You can combine them in a `Composite trigger` (e.g. after watermark is updated, it emits data, AND also after 20 seconds, AND also after 5 messages). So you can have any combination of triggers.

So you can emit the results of a window several times... How does Apache Beam controls that? `Accumulation modes`. When you trigger the window several times, you need to decide how to accumulate the results:
- `Accumulate`: Every time you trigger again, all messages for that window are emitted again (plus the new ones).
- `Discard`: Only include the new messages on the new triggering.

Examples:

```python
pcollection | WindowInto(
    SlidingWindows(60, 5), 
    trigger=AfterWatermark(
        early=AfterProcessingTime(delay=30), 
        late=AfterCount(1)
    ),
    accumulation_mode=AccumulationMode.ACCUMULATING,
    allowed_lateness=Duration(seconds=60)
)
# Sliding window of 60 seconds, every 5 seconds
# Relative to the watermark, trigger:
# -- fires 30 seconds after pipeline commences
# -- and for every late record (< allowedLateness)
# the pane should have all the records

pcollection | WindowInto(FixedWindows(60), trigger=Repeatedly(
    AfterAny (
        AfterCount (100),
        AfterProcessingTime (1 * 60))),
        accumulation_mode=AccumulationMode.DISCARDING, allowed_lateness=Duration(seconds=2*24*60*60))
# Fixed window of 60 seconds
# Set up a composite trigger that triggers
# whenever either of these happens:
# -- 100 elements accumulate
# -- every 60 seconds (ignore watermark)
# the trigger should be with only new records
# 2 days
```
