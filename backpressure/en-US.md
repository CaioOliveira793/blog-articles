# Backpressure — How it works and types of implementations

In the context of distributed systems, it’s common to create programs that react to a data input in which the producer can issue more than the consumer can handle, opening up possibilities for failure conditions such as data loss, or unavailability in the consumer systems.

To avoid errors due to greater production speed, a resistance opposite to the data flow is required. In other words, some form of backpressure must be applied.

To approach an implementation for specific situations, some strategies are more suitable for certain applications.

### Buffering

A widely used strategy is the queuing of items between producer and consumer, so that in peak load cases it’s possible to hold orders and process some time later.

Although queues being one of the most used solutions, some implications may be encountered.

In scenarios where the production rate is higher than consumption for long periods, the queue is likely to be at full capacity most of the time, preventing the producer from publishing new items, bubbling up unavailability errors for the user.

However, employing the use of unbounded queues can bring more complications, for example the lack of memory caused by an uncontrolled growth, expiration of the processing time of requests, etc.

Although buffering is the strategy that involves more complexity, with a correct implementation and queue bound adjustments, it is possible to result in better throughput and consistency in consumption performance.

### Control

Controlling the producer’s emission in cases where it’s possible, may be the simplest solution, with no need for buffering or data dropping.

Interfaces that use the pulling mechanism are naturally able to provide backpressure, e.g. requesting new data only when ready to process. An example of this method that has great use are IO Streams, present in many languages/runtimes.

### Dropping

In contexts where data loss is tolerable, appling of techniques that make dropping becomes viable, reducing the load by removing items to be processed.

With use in real-time update demands that do not use outdated data, such as user interfaces, a debounce technique has a number of applications, from collecting data of a search input during typing, to the rendering issued by a screen resize event.

## Summary

From a concept found in other areas, is brilliant how it’s possible to implement and make solutions that enable constant levels of performance in load peaks, reduction of resource usage and/or better efficiency when manipulating data.

Despite having some complex implementations and not being an intuitively desired behavior, backpressure makes possible to deal with large problems in a scalable way, being a desirable technique to have in the toolbox.
