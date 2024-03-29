---
title: "Seqlock-Based Atomic Memory Snapshots"
seoDescription: "We discuss seqlock-based atomic snapshots as an alternative to the k-word CAS algorithm when writes are infrequent and single-writer limitation is fine."
datePublished: Sun Aug 06 2023 11:10:02 GMT+0000 (Coordinated Universal Time)
cuid: clkzcdoqo000i0amr1es04tmn
slug: seqlock-based-atomic-memory-snapshots
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/mdgMbYfFlSA/upload/dca4ca8fa58a9751367c20813364de08.jpeg
tags: algorithms, multithreading, concurrency

---

Last time we [discussed](https://puzpuzpuz.dev/a-few-thoughts-on-k-word-cas) k-word CAS algorithm and came to the conclusion that seqlock-based atomic snapshots may be used as an alternative in situations when your writes are infrequent and single-writer limitation is acceptable, but you want the readers to be able to read multiple values atomically. Such use cases may be [met](https://en.wikipedia.org/wiki/Seqlock) in the Linux kernel, but not only. For instance, QuestDB uses a [similar approach](https://github.com/questdb/questdb/blob/0fd8581f6a3ee98a0117cf620d48fa55d7d16c76/core/src/main/java/io/questdb/cairo/TxReader.java#L392-L428) to read the metadata of the latest transaction atomically. So, today we'll consider an implementation in Java and try to measure how efficient it is from the reader's perspective.

To keep things simple, we'll write an atomic tuple class holding 3 `long` values. The structure of the class looks like the following:

```java
public class AtomicLongTuple {

    // Version is used for the seqlock.
    long version;
    // 3 long fields:
    long x;
    long y;
    long z;

    // Var handles for the fields:
    private static final VarHandle VH_VERSION;
    private static final VarHandle VH_X;
    private static final VarHandle VH_Y;
    private static final VarHandle VH_Z;

    // Holder class used by readers and writers.
    public static class TupleHolder {
        public long x;
        public long y;
        public long z;
    }

    // Readers provide a holder to avoid an allocation.
    public void read(TupleHolder holder) {
        // Implementation goes here...
    }

    // Writers provide a function to modify the latest state.
    public void write(Consumer<TupleHolder> writer) {
        // Implementation goes here...
    }
}
```

Here we have the tuple, i.e. 3 `long` fields, plus one more `long` field called `version` . As we'll see now, the version is needed to implement the seqlock. Of course, the above code doesn't include boring details such as padding or `VarHandle` initialization, but you may find the full source code [here](https://github.com/puzpuzpuz/java-concurrency-samples/blob/5bdfc514a859867e90ab36217f952a32835f94ea/src/main/java/io/puzpuzpuz/atomic/AtomicLongTuple.java).

To learn about the seqlock, let's start with the `write()` method:

```java
public void write(Consumer<TupleHolder> writer) {
    for (;;) {
        final long version = (long) VH_VERSION.getAcquire(this);
        if ((version & 1) == 1) {
            // Another write is in progress. Back off and keep spinning.
            LockSupport.parkNanos(1);
            continue;
        }

        // Try to update the version to an odd value (write intent).
        // We don't use compareAndExchangeRelease here to avoid
        // an additional full fence following this operation.
        final long currentVersion = (long) VH_VERSION.compareAndExchange(this, version, version + 1);
        if (currentVersion != version) {
            // Someone else started writing. Back off and try again.
            LockSupport.parkNanos(10);
            continue;
        }

        // Apply the write.
        writerHolder.x = (long) VH_X.getOpaque(this);
        writerHolder.y = (long) VH_Y.getOpaque(this);
        writerHolder.z = (long) VH_Z.getOpaque(this);
        writer.accept(writerHolder);
        VH_X.setOpaque(this, writerHolder.x);
        VH_Y.setOpaque(this, writerHolder.y);
        VH_Z.setOpaque(this, writerHolder.z);

        // Update the version to an even value (write finished).
        VH_VERSION.setRelease(this, version + 2);
        return;
    }
}
```

The above code uses acquire/release memory semantics here, so if you're not familiar with it, refer to [this post](https://puzpuzpuz.dev/using-acquirerelease-semantics-in-java-atomics-for-fun-and-profit). Other than that, the code is quite simple: each writer spins trying to increment the `version` field to an odd value to prevent concurrent writes (think, a [spinlock](https://puzpuzpuz.dev/benchmarking-non-shared-locks-in-java)) and, when it succeeds, applies the given function to the current tuple state. Finally, it increments the version, so that it has an even value telling readers and other writers that there is no ongoing write.

Notice that we're using plain loads and stores to fetch and modify the tuple in the "critical section". These operations don't need to be atomic, all thanks to the atomicity (and ordering guarantees) of the surrounding operations and memory fences.

Now, let's check what readers do:

```java
public void read(TupleHolder holder) {
    for (;;) {
        final long version = (long) VH_VERSION.getAcquire(this);
        if ((version & 1) == 1) {
            // A write is in progress. Back off and keep spinning.
            LockSupport.parkNanos(1);
            continue;
        }

        // Read the tuple.
        holder.x = (long) VH_X.getOpaque(this);
        holder.y = (long) VH_Y.getOpaque(this);
        holder.z = (long) VH_Z.getOpaque(this);

        // We don't want the below load to bubble up, hence the fence.
        VarHandle.loadLoadFence();

        final long currentVersion = (long) VH_VERSION.getAcquire(this);
        if (currentVersion == version) {
            // The version didn't change, so the atomic snapshot succeeded.
            return;
        }
    }
}
```

The reader's code is even simpler than the writer's one. Each reader spins trying to read an even version, then reads the tuple state. Finally, it reads the version once again and compares it with the previously read value. If the value hasn't changed, the reader was able to read the tuple atomically, so the operation finishes. Again, the tuple state operations are non-atomic, but thanks to the surrounding operations and fences that's not needed.

A "classical" seqlock assumes a separate sequence number and a mutex. Using a mutex over a hand-made spinlock is a wise choice since mutexes in modern runtimes and OSes came a long path and they're not as expensive as they used to be. In our case, the `version` field combines the sequence number and the mutex (to be more precise, a spinlock). For educational purposes, that's not really important since the algorithm is the same. Thanks to seqlock, we were able to implement atomic memory snapshots with a few lines of code.

As you may have noticed, the algorithm is not lock-free since if a writer thread gets blocked after it has incremented the version to an odd value, no readers and writers would be able to make progress. Considering this, you may be asking yourself how practical this approach is. It's certainly not versatile and only makes sense in use cases when the writes are infrequent and the total amount of memory to be read stays within a few cache lines (ideally, within a single cache line, i.e. 64 bytes on most modern machines). In other scenarios, it's better to use a good old exclusive or [shared](https://puzpuzpuz.dev/scalable-readers-writer-lock) mutex.

If you're curious about how many wasteful spins a reader has to make before making a successful snapshot, there is a [test](https://github.com/puzpuzpuz/java-concurrency-samples/blob/e180af67323b324e2d69f8cce422e1d9b618b2c6/src/test/java/io/puzpuzpuz/atomic/AtomicLongTupleTest.java#L36) for that. It starts a few reader threads that spin over the `read()` method and a single writer thread that calls `write()` and then emulates a bit of work by calling `Blackhole.consumeCPU(50)` on the `Blackhole` class from JMH. When the test finishes, each reader thread reports the `wasteful_spins / total_spins` ratio which, on my Linux machine, is around 0.7-0.9. So, in the presence of relatively frequent writes, no reader has to do more than a couple of spins on average.

That's it for today. Good luck and see you next time.