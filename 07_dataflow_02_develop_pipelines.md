# Dataflow: Develop pipelines

3 ways to launch a Dataflow pipeline:

- Launching a template using the Create Job Wizard in Cloud Console
- Authoring a pipeline using the Apache Beam SDK and launching from your development environment <-- This module
- Writing a SQL statement and launching it in the Dataflow SQL UI <-- This module

## Apache Beam (`B`atch + str`eam`)

Unifying batch programming and streaming processing is a big innovation in data engineering. 

### Main concepts

The four main concepts are:

- Ptransforms: The actions are contained in here. It handles input, transformation and output of the data. The data in a PCollection is passed along the graph from one PTransform to another.
- PCollections: Immutable data abstraction where data is held. Immutable data simplifies distributed processing, since no need for managing the shared resource.
- Pipelines: Identifies the data to be processed and the actions to be taken on the data. 
- Pipeline runners: They are analagous to container hosts, such as K8s. The integral pipeline can be run on a local computer, in a VM, in a data center or in a service in the cloud, such as `Dataflow`. The only difference is scale and access to platform specific services (e.g. GCS).

### Transforms

Applies operations at scale.

- `ParDo` (parallel do): It lets you apply a function to each one of the elements of a PCollection
- `GroupByKey` / `Combine`: With GroupByKey, you put all the elements with the same key together in the same worker. If your group is very large or the data is very skewed, you have a so-called hotkey and you're going to apply a commutative and associative operation, you can use Combine instead. Combine will make the transformation in a hierarchy of several steps. For large groups, this will have much better performance than GroupByKey.
- `CoGroupByKey`: It will join 2 PCollections by a common key. You can create a left or right, outer join, inner join and so on using CoGroupByKey.
- `Flatten`: Flatten also receives two or more input P collections and fuses them together. But please do not confuse flattened with joins or with GroupByKey. If two P collections contain exactly the same type, they can be fused together in just one P collection using the Flatten transform. However, with joins or with CoGroupByKey, you have two P collections, but typically with different value types that share a common key.
- `Partition`: It's the opposite of `Flatten`. It divides your PCollection into several output P collections by applying a function that assigns a groupID to each element in the input P collection.

### DoFn

It's not simply a map function, but it offers very powerful possibilities, like map, flatMap, filter, WithKeys (applies a fn to keys), keys (extract keys from (key,value)), values.

Elements in a PCollection are `processing bundles` (a worker in Dataflow receives a bunch of `bundles` to process). Each bundle is passed to a `DoFn` object. Many DoFns can be running at the same time within the same process in the worker (1 process can handle multiple bundles!). 

The division of the collection into bundles is arbitrary and selected by the runner. This allows the runner to find the middleground between persistent results and having to retry everything if there's a failure. For example:
    - A streaming runner may prefer to process and commit small bundles
    - A batch runner may prefer to process larger bundles.

When processing keys and values a single bundle may contain several different keys.

DoFn can be overriden in your code to control how your code interacts with each data bundle: `setup(self)`, `start_bundle(self)`, `process(self, element)` (main method), `finish_bundle(self)`, `teardown(self)`.

1. When a worker starts, it creates an instance of your DoFn. After it creates the instance, it calles the `@Setup` method (it's called only once per worker): It's used for setting up DB connections, etc.
2. Every time you receive a new data bundle, your runner calls the `@StartBundle` method of your DoFn. This is a good place to track your Data bundle if you need to (e.g. for instance variables, or metrics purposes).
3. After I start the bundle for every element, the runner will call the `@ProcessElement` (this is where the transform takes place). For that transform, your process method may `read state objects` or receive side inputs. From the process method you can also update state. If you define timers, this could be called more than once in a bundle, depending on the value of the timer (`@OnTimer`).
4. Once the DoFn Transform method processes the last element of the bundle, the runner calls the method `@FinishBundle`. This is a good place to do batch actions, for instance if you're updating an external system if your DoFn is a sink. This method could `write state objects`.
5. Finally when all data bundles are processed, and the worker is not needed anymore, the runner calls the `@TearDown` method. If you started any connections in the `@Setup` method, this is the place where you teardown those connections.

As a generic rule, always mutate state using `state variables`, rather than class members (which could be recycled by the runner, or reused into different workers for redundancy). And remember, a bundle can contain several keys, so store the state in maps based on that key.

