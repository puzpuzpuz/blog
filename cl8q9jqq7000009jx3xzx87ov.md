## Fast and Simple SPSC Queue

Single producer single consumer (SPSC) queues form the simplest type of concurrent queues. We have a single thread producing the items, as well as a single thread consuming them concurrently - what can be simpler than that? Nevertheless, such queues may be met in complex software projects, such as Linux kernel. Use cases include sending network packets between NICs and OS drivers and receiving I/O completion events in [io_uring](https://kernel.dk/io_uring.pdf), the newest asynchronous I/O API available in Linux. An SPSC queue may be unbounded meaning that the total number of items that can be pushed into the queue is unlimited or bounded which in practice means that it's built on top of a [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer). Today, we're discussing a bounded SPSC queue implemented in Java. The beauty of this data structure is its simplicity combined with a good level of performance on modern hardware.

We start with the skeleton of the data structure, i.e. its fields and interface:
```java
public class SpscBoundedQueue<E> {

    private final Object[] data;
    private final PaddedAtomicInteger producerIdx = new PaddedAtomicInteger();
    private final PaddedAtomicInteger producerCachedIdx = new PaddedAtomicInteger();
    private final PaddedAtomicInteger consumerIdx = new PaddedAtomicInteger();
    private final PaddedAtomicInteger consumerCachedIdx = new PaddedAtomicInteger();

    public SpscBoundedQueue(int size) {
        this.data = new Object[size + 1];
    }

    public boolean offer(E e) {
        // The code will follow...
    }

    public E poll() {
        // The code will follow...
    }

    static class PaddedAtomicInteger extends AtomicInteger {
        @SuppressWarnings("unused")
        private int i1, i2, i3, i4, i5, i6, i7, i8,
                i9, i10, i11, i12, i13, i14, i15;
    }
}
```

Here, we have an array of queue elements plus a number of index fields where consumer and producer each get a pair of `PaddedAtomicInteger`. The `PaddedAtomicInteger` is basically the standard `j.u.c.a.AtomicInteger` class with some padding added to prevent [false sharing](https://en.wikipedia.org/wiki/False_sharing). Alternatively, we could keep the memory layout flat with all indexes declared as primitive fields right in the `SpscBoundedQueue` class, but this would make the code much less readable.

You may also notice that the `offer()` and `poll()` methods are blocking. Again, that's to keep the code compact and readable. Adding optimistic, as well as batch flavors of the methods is simple enough and left as an exercise for curious readers.

The array of queue items is used as a ring buffer of arbitrary size, i.e. there is no power of two restriction for the size like in some ring buffer implementations. The `producerIdx` and `consumerIdx` fields are used to synchronize producer's and consumer's accesses to the array. Both producer and consumer check each other's index to understand if they can insert or read the next item and, if the check succeeds, perform the action and update their own index. Two other fields are used to cache the index seen during the latest check. We'll discuss why such caching improves the end performance in a moment.

Let's see how it all works for the producer:
```java
public boolean offer(E e) {
    // Read producer's own index.
    final int idx = producerIdx.getOpaque();
    int nextIdx = idx + 1;
    if (nextIdx == data.length) {
        nextIdx = 0;
    }
    // Read the last seen consumer's index.
    int cachedIdx = consumerCachedIdx.getPlain();
    if (nextIdx == cachedIdx) {
        // If we have reached the known index, we need to read the current value.
        cachedIdx = consumerIdx.getAcquire();
        // Make sure to update the cached value.
        consumerCachedIdx.setPlain(cachedIdx);
        if (nextIdx == cachedIdx) {
            // The queue is full.
            return false;
        }
    }
    // There is an empty slot, so we can insert the item.
    data[idx] = e;
    // Make sure to update our own index.
    // We use release semantics while the consumer has an acquire edge.
    producerIdx.setRelease(nextIdx);
    return true;
}
```

The above code uses [acquire/release semantics](https://puzpuzpuz.dev/using-acquirerelease-semantics-in-java-atomics-for-fun-and-profit) to keep the emitted instructions as lightweight as possible from the cache coherency traffic perspective. Other than that, the code does pretty much as what we discussed before.

As it was already mentioned, the manipulations with the `consumerCachedIdx` field are important for the end performance. All reads and writes on this field are thread-local, i.e. only producer threads accesses this field, so we don't need to use costly atomic operations. This reduces cache coherency traffic dramatically and lets the CPU core on its own thread-local data in those cases when there multiple empty slots are available in the queue.

Consumer's part of the picture may be seen in the full source code available [here](https://github.com/puzpuzpuz/java-concurrency-samples/blob/6eb2c14e5cc7476a268606c94abb722c2e6f1e81/src/main/java/io/puzpuzpuz/queue/SpscBoundedQueue.java).

Finally, we're going compare our queue with the good old `j.u.c.ArrayBlockingQueue` and a couple of SPSC queue implementations from [JCTools](https://github.com/JCTools/JCTools) library. If you're not familiar with JCTools and never used it, I advice you to put it on your radar.

The benchmark we'll be running is available [here](https://github.com/puzpuzpuz/java-concurrency-samples/blob/6eb2c14e5cc7476a268606c94abb722c2e6f1e81/src/test/java/io/puzpuzpuz/queue/SpscQueueBenchmark.java). When run, it starts a couple of threads to play a ping-pong game. Each operation, a.k.a. a ping-pong round, assumes sending/receiving a single item over the SPSC queue combined with a bit of work done for each successful attempt.

Here is a reduced JMH benchmark output on my laptop running Ubuntu 20.04 and OpenJDK 17.0.4 64-bit:
```
Benchmark                                            (type)   Mode  Cnt          Score          Error  Units
SpscQueueBenchmark.group                         SPSC_QUEUE  thrpt    3  107503612.612 ± 16230253.288  ops/s
SpscQueueBenchmark.group               ARRAY_BLOCKING_QUEUE  thrpt    3    7158948.722 ±  8635350.468  ops/s
SpscQueueBenchmark.group                      JCTOOLS_QUEUE  thrpt    3  120533694.168 ±  4686758.722  ops/s
SpscQueueBenchmark.group               JCTOOLS_ATOMIC_QUEUE  thrpt    3  101704017.278 ± 18252611.281  ops/s
```

As expected, JCTools' queues and our own one are significantly faster than the `ArrayBlockingQueue` queue. Also, surprisingly, our SPSC queue keeps on par with the JCTools' queues which is not something I was expecting, to be honest. Does it mean that you should go for an in-house implementation instead of JCTools? Not really. If you can afford yourself 3rd-party dependencies, go for JCTools. JCTools' data structures are certainly more efficient, as well as much better tested and benchmarked than our toy queue. So, you'd have to spend quite some time reaching the same level of stability for a DIY queue.

Needless to say that this algorithm is not something new. You may see it in this great [blog post](https://rigtorp.se/ringbuffer/) by Erik Rigtorp, as well as recognize it in the `SPSequence` and `SCSequence` classes in QuestDB's [source code](https://github.com/questdb/questdb). Yet, I hope that this data structure would be a nice addition to your engineering toolkit. See you next time.