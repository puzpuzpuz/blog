## Scalable Readers-Writer Lock

Locks, or mutexes (mutual exclusions), are probably the most basic concurrency primitives. It's hard to find a developer who won't be able to explain a mutex, at least on the fundamental level. Yet, mutexes are more than that. They may be:
* OS-level (think, a pthread mutex) or user-land (think, a spinlock),
* expose pessimistic (blocking) or optimistic (non-blocking) locking API,
* provide fairness in lock acquisition or keep things unfair,
* support reentrant calls, or prefer to be non-reentrant,
* have a notion of asymmetry in locking (say, with a shared lock available for readers) or stick to symmetric, exclusive locking,
* strictly require unlocking on the same thread (a pthread mutex, once again) or prefer not to bother with unlocker's identity (`sync.Mutex` in Golang).

Today we focus on asymmetric, readers-writer locks which are [familiar](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReadWriteLock.html) to most Java developers. Such locks allow concurrent readers to proceed with executing their critical section, while writers are guaranteed to acquire exclusive ownership of the lock. These locks are used in scenarios where the vast majority of calls come from readers and writers acquire the lock rather infrequently.

Our ultimate goal is to come up with a lock implementation that would scale read operations linearly in terms of CPU core count and compare it with alternatives such as  [`ReentrantReadWriteLock`](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html) class from the standard library.

## Prior art

Readers-writer (R/W) locks are not something new. A wide-spread R/W lock implementation uses an atomic counter for the reader part and looks something like the [following class](https://github.com/puzpuzpuz/questdb/blob/f817ed19b205be383d9556b64c8e1ac96a5f377d/core/src/main/java/io/questdb/std/SimpleReadWriteLock.java) from QuestDB code base.

Locking for a reader in this class looks like this:
```java
@Override
public void lock() {
    // start a lock attempt
    while (nReaders.incrementAndGet() >= MAX_READERS) {
        // there is a writer owning the lock, so clean up, sleep and go for another spin
        nReaders.decrementAndGet();
        LockSupport.parkNanos(10);
    }
}
```

*Note.* If you run your code on Windows, you may [face](https://hazelcast.com/blog/locksupport-parknanos-under-the-hood-and-the-curious-case-of-parking-part-ii-windows/) latency issues with `LockSupport.parkNanos()`. So, make sure to do some benchmarking before using any of the locks we cover today.

Here `nReaders` is an `AtomicInteger` used as a medium between readers and the writer. Each reader increments the counter atomically and checks the result value. If it's smaller than the threshold, a reader lock is acquired successfully. If not, the reader has to busy spin (in fact, it could sleep and wait for a notification from the writer, but that would slightly increase the latency). The writer, on the other hand, does the following to acquire a lock:
```java
@Override
public void lock() {
    // trimmed code that acquires the internal writer lock:
    // ...

    // increment the readers counter by the threshold
    int n = nReaders.addAndGet(MAX_READERS);
    // wait until there are no readers holding a lock
    while (n != MAX_READERS) {
        n = nReaders.get();
    }
}
```

It is important to stress that this lock is non-reentrant. As it was previously mentioned, such readers-writer lock design is quite popular and you may find it in, say, Go standard library's [sync.RWMutex](https://github.com/golang/go/blob/b357b05b70d2b8c4988ac2a27f2af176e7a09e1b/src/sync/rwmutex.go#L56-L69) struct. The main problem with this approach is that its reader part doesn't scale. This means that if the time spent in the critical section by each reader is rather low, adding more threads (and cores) to the program may not lead to improved performance.

Let's demonstrate this. Our test stand is a laptop with i7-1185G7 CPU with 4/8 cores running Ubuntu 20.04 and OpenJDK 17.0.1. The JMH microbenchmark we're going to use may be found [here](https://github.com/puzpuzpuz/questdb/blob/f817ed19b205be383d9556b64c8e1ac96a5f377d/benchmarks/src/main/java/org/questdb/ReadWriteLockBenchmark.java).

```
Benchmark                            (type)  Mode  Cnt    Score   Error  Units
ReadWriteLockBenchmark.testBaseline     N/A  avgt   10   21.506 ± 0.784  ns/op
ReadWriteLockBenchmark.testLock2     SIMPLE  avgt   10  101.476 ± 2.403  ns/op
ReadWriteLockBenchmark.testLock4     SIMPLE  avgt   10  205.656 ± 1.970  ns/op
ReadWriteLockBenchmark.testLock8     SIMPLE  avgt   10  296.461 ± 0.735  ns/op
```

Here the `testBaseline` benchmark stands for the baseline, i.e. the work done in the critical section in the main benchmark. As for, `testLockN` it means reader lock benchmark run on N threads.

You may have already noticed almost linear degradation in the average operation time when we increase the number of threads. That's because the hot path in the microbenchmark boils down to an atomic increment instruction available in modern CPUs, e.g. `LOCK XADD` on x86. The aforementioned instruction implies exclusive access to the counter (the corresponding cache line, to be more precise) acquired by the CPU core executing the caller thread. Hence, the increment is still restricted with a single core and, to add on top of that, in the face of contention the synchronization cost paid by each core increases significantly. Refer to this comprehensive [blog post](https://travisdowns.github.io/blog/2020/07/06/concurrency-costs.html) by Travis Downs to learn more about the cost of concurrency primitives on modern HW. 

If we add `LinuxPerfProfiler` JMH profiler and re-run the benchmark to get [`perf stat`](https://man7.org/linux/man-pages/man1/perf-stat.1.html) output for the benchmark run on 2 threads, we'll see the following:
```
Perf stats:
--------------------------------------------------

        199 010,83 msec task-clock                #    1,521 CPUs utilized          
             3 887      context-switches          #    0,020 K/sec                  
               223      cpu-migrations            #    0,001 K/sec                  
               255      page-faults               #    0,001 K/sec                  
   614 120 444 917      cycles                    #    3,086 GHz                      (41,68%)
   418 345 242 445      instructions              #    0,68  insn per cycle           (50,02%)
    21 283 383 788      branches                  #  106,946 M/sec                    (58,35%)
         2 027 813      branch-misses             #    0,01% of all branches          (66,69%)
    57 388 541 249      L1-dcache-loads           #  288,369 M/sec                    (66,69%)
     2 080 486 452      L1-dcache-load-misses     #    3,63% of all L1-dcache accesses  (66,68%)
         2 462 215      LLC-loads                 #    0,012 M/sec                    (66,68%)
           312 884      LLC-load-misses           #   12,71% of all LL-cache accesses  (66,69%)
   <not supported>      L1-icache-loads                                             
        20 972 526      L1-icache-load-misses                                         (33,32%)
    57 430 817 101      dTLB-loads                #  288,581 M/sec                    (33,33%)
            63 230      dTLB-load-misses          #    0,00% of all dTLB cache accesses  (33,34%)
   <not supported>      iTLB-loads                                                  
           116 372      iTLB-load-misses                                              (33,33%)
   <not supported>      L1-dcache-prefetches                                        
   <not supported>      L1-dcache-prefetch-misses                                   

     130,817291871 seconds time elapsed

     260,986586000 seconds user
       0,210040000 seconds sys
```

Now, if we run the benchmark on 8 threads, the result would be:
```
Perf stats:
--------------------------------------------------

        784 673,96 msec task-clock                #    5,996 CPUs utilized          
           340 290      context-switches          #    0,434 K/sec                  
               212      cpu-migrations            #    0,000 K/sec                  
               253      page-faults               #    0,000 K/sec                  
 2 418 749 827 832      cycles                    #    3,082 GHz                      (41,66%)
   394 531 732 276      instructions              #    0,16  insn per cycle           (50,01%)
    20 804 674 516      branches                  #   26,514 M/sec                    (58,35%)
        11 107 908      branch-misses             #    0,05% of all branches          (66,67%)
    54 747 118 838      L1-dcache-loads           #   69,771 M/sec                    (66,68%)
     2 413 557 669      L1-dcache-load-misses     #    4,41% of all L1-dcache accesses  (66,68%)
       582 485 788      LLC-loads                 #    0,742 M/sec                    (66,67%)
           484 122      LLC-load-misses           #    0,08% of all LL-cache accesses  (66,66%)
   <not supported>      L1-icache-loads                                             
       157 690 988      L1-icache-load-misses                                         (33,32%)
    54 789 336 429      dTLB-loads                #   69,824 M/sec                    (33,33%)
           504 440      dTLB-load-misses          #    0,00% of all dTLB cache accesses  (33,33%)
   <not supported>      iTLB-loads                                                  
         3 304 265      iTLB-load-misses                                              (33,34%)
   <not supported>      L1-dcache-prefetches                                        
   <not supported>      L1-dcache-prefetch-misses                                   

     130,864386377 seconds time elapsed

    1025,676317000 seconds user
       2,552276000 seconds sys
```

The previously mentioned problem with contention over a single atomic counter primary manifests itself in significant degradation of instruction per cycle (IPC) metric: it goes from 0,68 insn/cycle for 2 threads, which is already quite low, to pitiful 0,16 insn/cycle for 8 threads. Other metrics also degrade proportionally. That's because each CPU core's backend spends a lot of time synchronizing the counter's cache line with other cores.

To be precise, low IPC doesn't necessarily mean contention caused by atomic instructions. It may be caused by other reasons, such as random memory accesses on a data structure that doesn't fit into memory. If you'd like to detect this particular scenario, you should profile for specific PMU (Performance Monitoring Unit) events as Travis Downs [pointed out](https://twitter.com/trav_downs/status/1466472385834590211?s=20).

To nail down the single atomic counter-based lock topic, let's try to compare it with the standard `j.u.c.l.ReentrantReadWriteLock` class. We're going to use the same benchmark running on 8 threads:
```
Benchmark                        (type)  Mode  Cnt     Score    Error  Units
ReadWriteLockBenchmark.testLock     JUC  avgt   10  2081.664 ± 15.117  ns/op
ReadWriteLockBenchmark.testLock  SIMPLE  avgt   10   289.155 ±  3.339  ns/op
```

Here `JUC` type stands for the `ReentrantReadWriteLock` class and `SIMPLE` means the single atomic counter lock. The difference is significant, almost 7x. Moreover, if we would activate the GC profiler in the benchmark, we would see that the standard class allocates around 200 MB/sec (not the end of the world, but could be avoided), while the atomic counter lock does not allocate at all. So, if you have a concrete use case and you really know what you're doing, using a custom lock might be a good idea.

Long story short, a single atomic counter-based lock might do a better job than the standard Java library, but it has an important flaw. The thing is that it assumes contention between readers while it's not necessary at all. Readers need to share memory (think, synchronize) with the writer only, not with each other. And, of course, there are algorithms that try to address the reader scalability problem. Let's quickly discuss the alternatives.

The first alternative I'm aware of is Dmitry Vyukov's distributed reader-writer [mutex](https://www.1024cores.net/home/lock-free-algorithms/reader-writer-problem/distributed-reader-writer-mutex). It's written in C++, but it should be straightforward to port it to Java. The main idea is to shard the reader counter to an array of atomic counters. The size of the array is set in runtime to match the number of available cores. Each reader is associated with a slot in the array based on the id of the core running the reader thread. The id is obtained via the [`getcpu()`](https://man7.org/linux/man-pages/man2/getcpu.2.html) system call available on Linux, so porting it to other OSes is problematic. Another problem is that threads may migrate between cores unless you go with thread affinity. To address this problem, D.Vyukov's lock provides the core id as the return value of the reader's `lock()` method. This value has to be provided later when the reader calls `unlock()`. So, while this lock has the potential, it's certainly not general-purpose.

Another worthwhile scalable readers-writer lock I know of is called BRAVO lock. The idea is quite close to D.Vyukov's class, yet the sharded counters array is fixed-size, and the reader's slot is determined based on the thread id. The second part is not a hard requirement and it's possible to implement the BRAVO algorithm in, say, [Golang](https://github.com/puzpuzpuz/xsync/blob/efd8a81aa9261ce9d19dc9c02a9a87e8f34d8e93/rbmutex.go) which doesn't expose goroutine or thread id by design. Implementation in Java is available [here](https://github.com/puzpuzpuz/questdb/blob/f817ed19b205be383d9556b64c8e1ac96a5f377d/core/src/main/java/io/questdb/std/BiasedReadWriteLock.java). We're going to benchmark it against the single atomic counter lock now. If you're interested in learning more about BRAVO lock, refer to the [original paper](https://arxiv.org/pdf/1810.01553.pdf).

Here is how BRAVO lock does in the benchmark run on 8 threads:
```
Benchmark                          (type)  Mode  Cnt    Score   Error  Units
ReadWriteLockBenchmark.testLock    SIMPLE  avgt   10  296.300 ± 1.824  ns/op
ReadWriteLockBenchmark.testLock    BIASED  avgt   10   53.811 ± 1.548  ns/op
```

The BIASED type from the above JMH output is the BRAVO lock while the SIMPLE type is the same atomic counter lock we previously discussed. As expected, BRAVO lock solves the scalability issue for readers, but it also has some problematic parts. First, the size of the array has to be large enough to avoid reader contention. The Java class we benchmarked sets the array size to 4096, which means 16KB of memory. The array size should be ideally based on the hardware that runs your code. Second, even if the array is properly sized, readers may still go to the same slot. In fact, adjacent slots are enough as long as they occupy the same cache line. That's because the slot assignment applies a hash function to the thread id and, due to that, does not guarantee the absence of hash code collisions.

So, both alternatives have certain cons and leave enough space for improvements. That's exactly what we came for.

## Meet TLBiasedReadWriteLock

Yes, naming is not my strong side. `TLBiasedReadWriteLock` stands for thread-local reader biased lock. As the name suggests, [the class](https://github.com/puzpuzpuz/questdb/blob/f817ed19b205be383d9556b64c8e1ac96a5f377d/core/src/main/java/io/questdb/std/TLBiasedReadWriteLock.java) uses a thread-local counter for each reader, while the writer lock is based on a spinlock.

Simplified reader's lock method looks like the following:
```java
@Override
public void lock() {
    // initialize or fetch a thread-local counter for the reader
    PaddedAtomicLong readerCounter = tlReaderCounter.get();
    for (;;) {
        // check if the writer lock is available and we can start the attempt
        if (!wLock.get()) {
            // increment the reader counter
            readerCounter.incrementAndGet();
            // check that no writer acquired the lock
            if (!wLock.get()) {
                // attempt is successful, we're done
                break;
            }
            // attempt failed, go for another spin
            readerCounter.decrementAndGet();
        }
        LockSupport.parkNanos(10);
    }
}
```

In this code, `PaddedAtomicLong` is nothing more than, well, a padded `AtomicLong` used as a reader counter. The padding is added to prevent false sharing, i.e. the situation when different readers are unfortunate to have their counters adjacent on the heap memory so that they end up in the same CPU cache line. When a reader accesses the thread-local counter for the first time, the counter gets added to an array of weak references. These weak references are checked by writers when they want to acquire the lock.

You may wonder about scalability of the above code in terms of readers. Yes, there are no writes to shared memory, but we read the writer lock value multiple times. Will this code scale? The answer is yes, reads (loads) of shared memory [scale](https://www.1024cores.net/home/lock-free-algorithms/first-things-first) and it's fine to use them anywhere in your code.

Next, writer's lock method looks like this:

```java
@Override
public void lock() {
    // first, acquire writer lock
    lock0();
    // next, wait for the readers
    Iterator<WeakReference<PaddedAtomicLong>> iterator = readerCounters.iterator();
    while (iterator.hasNext()) {
        // fetch reader counter's weak reference
        WeakReference<PaddedAtomicLong> ref = iterator.next();
        PaddedAtomicLong counter = ref.get();
        if (counter == null) {
            // clean up the counter since the reader thread stopped
            iterator.remove();
            continue;
        }
        while (counter.get() != 0) {
            // the reader still holds the lock, so sleep and go for another spin
            LockSupport.parkNanos(10);
        }
    }
}
```

As promised, this lock guarantees zero contention for readers since their counters are thread-local. The counters are allocated dynamically, so in scenarios when there are a few threads accessing the lock its memory footprint should be lower than BRAVO lock's one. This lock should do its best when accessed on a fixed-size thread pool.

If we benchmark this lock (TLBIASED type) against BRAVO (BIASED type) on 8 threads, we get this:
```
Benchmark                          (type)  Mode  Cnt    Score   Error  Units
ReadWriteLockBenchmark.testLock    BIASED  avgt   10   53.811 ± 1.548  ns/op
ReadWriteLockBenchmark.testLock  TLBIASED  avgt   10   54.682 ± 1.204  ns/op
```

As you would expect, two locks are on par in terms of reader lock scalability and end performance. Expectedly, if we would size BRAVO lock's array less aggressively or run the benchmark on a lot more cores/threads, BRAVO lock would perform worse.

## Final battle

Before we wrap up, let's compare all observed lock implementations in a slightly more realistic benchmark. To make it happen we increase the time spent in the critical section by 4x so that it takes around 140 nanoseconds instead of 22 nanoseconds used in the reader-only benchmark. Next, we change the benchmark so that the writer lock is occasionally acquired. The ratio between reader and writer lock calls we're going to use will be 1,000:1, 10,000:1, or 100,000:1. The benchmark code may be found [here](https://github.com/puzpuzpuz/questdb/blob/faaf0eb5deb948bc98f95172a9177f2f9386ff51/benchmarks/src/main/java/org/questdb/ReadWriteLockBenchmark.java#L85-L96).

That's what we get when the benchmark is run on 8 threads:
```
Benchmark                                 (readWriteRatio)    (type)  Mode  Cnt     Score    Error  Units
ReadWriteLockBenchmark.testReadWriteLock              1000       JUC  avgt   10  1966.910 ± 25.643  ns/op
ReadWriteLockBenchmark.testReadWriteLock              1000    SIMPLE  avgt   10   668.782 ±  0.879  ns/op
ReadWriteLockBenchmark.testReadWriteLock              1000    BIASED  avgt   10   458.470 ±  3.169  ns/op
ReadWriteLockBenchmark.testReadWriteLock              1000  TLBIASED  avgt   10   598.463 ±  2.135  ns/op
ReadWriteLockBenchmark.testReadWriteLock             10000       JUC  avgt   10  1828.119 ± 41.570  ns/op
ReadWriteLockBenchmark.testReadWriteLock             10000    SIMPLE  avgt   10   593.159 ±  1.913  ns/op
ReadWriteLockBenchmark.testReadWriteLock             10000    BIASED  avgt   10   219.809 ±  3.359  ns/op
ReadWriteLockBenchmark.testReadWriteLock             10000  TLBIASED  avgt   10   226.159 ±  0.808  ns/op
ReadWriteLockBenchmark.testReadWriteLock            100000       JUC  avgt   10  1973.585 ± 14.310  ns/op
ReadWriteLockBenchmark.testReadWriteLock            100000    SIMPLE  avgt   10   584.412 ±  5.078  ns/op
ReadWriteLockBenchmark.testReadWriteLock            100000    BIASED  avgt   10   173.194 ±  1.360  ns/op
ReadWriteLockBenchmark.testReadWriteLock            100000  TLBIASED  avgt   10   174.324 ±  3.236  ns/op
```

Good old `ReentrantReadWriteLock` (JUC) is equally slow regardless of the read/write lock call ratio. The single atomic counter lock (SIMPLE) also doesn't improve much when there are fewer writer calls. For the 1,000:1 ratio, the single counter lock's performance is very close to the performance of BRAVO lock (BIASED) and thread-local reader biased lock (TLBIASED).

What's interesting is that the BRAVO lock (BIASED) is a bit cheaper than the thread-local biased lock for the 1,000:1 read-write ratio, but when the number of reader calls grows the difference disappears. That's because of a more expensive write lock call in the thread-local biased lock. Again, the strong point of the thread-local lock is the guaranteed absence of contention for readers. Since the BRAVO lock is put into the best possible conditions in terms of the internal array sizing and the number of threads and CPU cores, we didn't observe any scalability issues with this lock in the conducted benchmarks.

## What's in it for me?

Before we go any further, I'd suggest using a custom lock implementation only in niche use cases and, what's even more important, only if you know what you're doing. In all other situations, the standard library should be the default way to go.

If you're certain that there is a majority of reader locks in your code, they have rather short critical sections, and the `ReentrantReadWriteLock` class shows up as the bottleneck in profiler reports, any of the observed alternatives have the chance to improve your application's performance.

Hopefully, you've learned something new today. Good luck and see you next time.