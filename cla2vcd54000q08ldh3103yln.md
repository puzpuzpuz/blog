# Concurrent Maps in Go vs Java: Yet Another Meaningless Benchmark

Today we're comparing Java's [`j.u.c.ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html) and Go's [`xsync.MapOf`](https://github.com/puzpuzpuz/xsync#map) in a totally non-scientific, unfair benchmark. While most of such language performance comparisons are generally useless and harmful, the purpose of this exercise is a comparison of the algorithms behind both data structures. I'm driven by curiosity here, so don't take this post seriously. The results may be completely different on another HW and in different scenarios, so don't forget that any benchmark has to be taken with a grain of salt.

On the one hand we have `ConcurrentHashMap`, also known as CHM. It's a brilliant concurrent hash table: no writes to shared memory in read-only operations, hybrid linked list / tree used for buckets depending on the number of stored nodes, volatile value fields in nodes allowing to update the value without having to allocate a new node, and so on. CHM has been improved across many versions of Java, for many years, so reaching its level of performance isn't an easy task.

On the other hand, `MapOf` borrows ideas from multiple sources: buckets are organized in unrolled linked lists of cache line size thanks to [Cache-Line Hash Table](https://github.com/LPD-EPFL/CLHT) (CLHT), hash codes are stored in the buckets to avoid extra hash function calls and pointer chasing, entries themselves are immutable structures, so no extra synchronization is necessary for reads and also reduced GC pressure.

Both maps have a bunch of handy atomic operations available to the developer. But enough of this boring stuff, let's do the benchmarking.

Our test stand is nothing more, but a laptop machine with an i7-1185G7 CPU (8 HT cores) and Ubuntu 22.04 running 64-bit builds of Go 1.19.3 and OpenJDK 17.0.4.

The CHM benchmark can be found [here](https://github.com/puzpuzpuz/java-concurrency-samples/blob/8fbd5032327551fb339ed6d7d208df55436a1952/src/test/java/io/puzpuzpuz/map/ConcurrentMapBenchmark.java), while the Go benchmark is in the [xsync repo](https://github.com/puzpuzpuz/xsync/blob/a140d88f8cdc4ebfddf75d89428079d8d1f3ad6f/mapof_test.go#L1045-L1061).

The benchmarks start with a pre-warmed map holding 1,000 64-bit integer key-value pairs. The "99% reads" scenario assumes that each thread spends 99% of its calls on `get` (`Load`) operations and 0.5% on both `put` (`Store`) and `remove` (`Delete`) operations. Each operation is called for a randomly selected key, so there are no hot spots. The "75% reads" scenario is more write-heavy as write operations get 12.5% of total number of operations each.

The benchmarks were run with different number of threads/cores to be used, staring with 1 core and ending with all 8 available cores. That's to see how well both maps scale on multi-core machines. Finally, the below results are based on average values collected after 10 runs of each benchmark.

Let's start with the write-heavy scenario. CHM has an advantage here since it allows updating values in-place, so it might show better results.

![Results for 75% reads scenario](https://cdn.hashnode.com/res/hashnode/image/upload/v1667588120561/-fPxV0rj8.png align="left")

As expected, CHM is slightly better on all core counts except for 4 cores. Both maps scale well enough.

Let's see results for the read-heavy scenario which might be more common in many real world applications.

![Results for 99% reads scenario](https://cdn.hashnode.com/res/hashnode/image/upload/v1667588318604/rtOSa0JJh.png align="left")

Again, both maps scale well with a slight advantage of `MapOf`. I'm pretty happy with the results. There are certainly rough edges, but `MapOf` has proven its worthiness.

Have fun coding and see you next time.