# Dataflow: Develop pipelines - States and timers

Regularly, `ParDo` operations are stateless. But Apache Beam comes with additional possibilities for `ParDo`s: Stateful transformations.

You can have state variables (accumulating state).

State is maintained `per key`, so the input should be pairs of key,values.

For streaming pipelines, the state is maintained `per window`.

The state can be read and mutated during the processing of each one of the elements. 

The state is local to each one of the transformers, i.e. two different keys processed in two different workers won't be able to share a common state, but all elements in the same key can share a common state.

So for a `ParDo` example that maps squares into circles, but at the same time calculates the total area of the incoming squares... that calculation is basically being accumulated in a state variable. From within the `processElement()` method, that state can be mutated.

Dataflow takes care of making sure that the mutation of the state is consistent and safe.

There are these types of state variables:

- Value: Read/write any value (but always the whole value)
- Bag: Cheap append (several elements). No ordering when reading the objects that were added.
- Combining: For any kind of aggregation that is associative and commutative, it is better to use the combining state.
- Map: And if you are going to maintain a set of key values, a dictionary or map, use map state. Map state is more efficient than other state variables for retrieving specific keys.
- Set: Finally, the set state, available in the patching programming model, but not supported in data flow.

In case your Dataflow pipeline is integrated with an exeternal system, processing an API call for instance for each element can be very problematic, since millions of calls will be done from all workers at the same time. To overcome this, state variables can accumulate results and buffer them so that tehy can be sent later on as batch calls to the API. 

Buffering example in Java:

```java
new DoFn<Event, EnrichedEvent>() {
    private static final int MAX_BUFFER_SIZE = 500;
    @StateId("buffer") private final StateSpec<BagState<Event>> bufferedEvents = StateSpecs.bag();
    @StateId( "count") private final StateSpec<ValueState<Integer>> countState = StateSpecs.value();

    @ProcessElement
    public void process(ProcessContext context,
        @StateId( "buffer") BagState<Event> bufferState,
        @StateId("count") ValueState<Integer> countState) {
        
            int count = firstNonNull(countState.read(), 0) ;
            count = count + 1; countState.write(count); // increment count and add element to buffer
            bufferState.add(context.element())
            
            if (count >= MAX_BUFFER_SIZE) { // when buffer size limit is reached a request is sent to the ext service
                for (EnrichedEvent enrichedEvent : enrichEvents(bufferState.read())) {
                    context.output(enrichedEvent);
                }
                bufferState. clear(); 
                countState.clear();
            }
        }
}
```

What happens for the last message that we receive, what if the buffer does not reach the max buffer size? The `DoFn` would keep that state indefinitely, and the `DoFn` would never finish. That's solved with `Timers`.

`Timers` are used in combination with state variables to ensure that the state is clear at regular intervals of time.

Event timers: Use them when you want to do the callback based on the watermark and the timestamps of your data. Event timers will be influenced by the rate of progress of the input data.
Processing timers: It will expire expires regardless of the progress of your data. The time will trigger at regular intervals.

```java
@OnTimer ("expiry")
public void onExpiry (OnTimerContext context, @StateId("buffer") BagState<Event> bufferState) {
    if (!bufferState.isEmpty(). read()) {
        for (EnrichedEvent enrichedEvent: enrichEvents (bufferState. read())) {
           context. output (enrichedEvent);
        }
        bufferState. clear ();
    }
}
```

state and timers usecases:

In any situation where you need fine control on how the aggregation of the elements is done, state and timers allow you to implement a precise and complex logic.

In general, any workflow that should be applied per key can be expressed as state and timers.