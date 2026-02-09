# Monitoring

- Job run, job info ... useful for seeing running jobs, job information (cpus, memory used, etc.) as well as region, project, etc.

- Job graph

DataFlow optimizes your pipeline for both streaming and batch pipelines. Part of the optimization includes fusing multiple steps in your pipeline into single steps. If we press on any step, we can see how DataFlow splits each stage into a number of optimized stages. Some stages will be shared between different steps. We can click on stages and see what's the shared stage status.

Another metric available at each step and substep is the `wall time`.
This shows the total amount of time by the assigned workers to run each step. This can be a useful metric to look at when you want to see where your workers are spending the most amount of time.

- Custom metrics

Counter: Increment and decrement
Distribution: COUNT, MIN, MAX, MEAN
Gauge: Latest value of the variable to track

On the DataFlow UI, we can also see the custom metrics associated with a job.

- Job metrics

For batch jobs, there are some job metrics that are useful, like autoscaling and CPU utilization. If workers' CPU is more or less the same, it proves that your pipeline is healthy. An uneven CPU workload represents an unhealthy pipeline, and the root cause could be derived from a badly distributed key on `GroupBy` operations, for instance.

For streaming jobs, these are useful metrics:

- Data freshness: The data freshness graph shows the difference between real time and the output watermark. The output watermark is a timestamp where any time step prior to the watermark is nearly guaranteed to have been processed. For example, if the current time is 9:26 a.m., and the data freshness graphs value at that time is six minutes, that means that all elements with a timestamp of 9:20 a.m. or earlier have arrived and have been processed by the pipeline. (so the higher, the less 'fresh' the data is)

- System latency: The system latency graph shows how long it takes elements to go through the pipeline. If the pipeline is blocked at any stage, the latency will increase. For example, imagine our pipeline reads from Pub/Sub, does some beam transformation on the elements, then syncs them into Spanner. Suddenly, Spanner goes down for five minutes. When this happens, Pub/Sub won't receive confirmation from Dataflow that an element has been sunk into Spanner. This confirmation is needed for Pub/Sub to delete that element. As there is no confirmation, the system latency and data freshness graphs will both rise to five minutes. (so the higher, the least ideal since more time happens for an element to go through the pipeline).

- Input metrics
    - Request per sec (for inputs, like pubsub). Depending on the input you have configured in your pipeline (you could have even multiple inputs), you can select theirs requests per secondon the `Request per sec` graph.
    - Response error per sec by error type: If errors occur frequently and repeatedly, see what they are and cross reference them to the specific error code documentation on Pub/Sub error codes.

- Metrics explorer: Explore all metrics and create custom dashboards