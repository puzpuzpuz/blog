## Using Acquire/Release Semantics in Java Atomics for Fun and Profit

In case you've missed it, recent JDK versions include new memory semantics for atomic operations available in [`VarHandle`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/VarHandle.html) and `Atomic*` classes. These new semantics are equivalent to C/C++'s [`std::memory_order`](https://en.cppreference.com/w/cpp/atomic/memory_order). The only confusing naming convention is that `*Opaque` methods in Java map to the `memory_order_relaxed` memory order in C/C++. Other than that, the idea is the same - these semantics allow developers to use a weaker memory model than the default sequential consistency model, i.e. full memory barrier which is used in the old atomic methods. This can potentially improve the performance at the cost of more complex and, thus, less maintainable code.

Anyhow, we're not going to go through the basics of memory semantics. If you're not familiar with them, I'd recommend watching [this talk](https://youtu.be/ZQFzMfHIxng) by Fedor Pikus where he does a great job at explaining C++'s `std::atomic`. As usual, today we'll be doing a weird and questionable experiment. We'll use acquire/release semantics to build a lossy (a.k.a. not-so-atomic) counter on top of `j.u.c.a.AtomicLong`.

Imagine that you need a rough order of magnitude counter in your application. Say, you want to measure the total number of operations performed mostly on a single thread, and in the case of concurrent execution, you're fine with losing some of the concurrent updates as long as the counter is incremented by at least one of the threads. Apart from the good old `AtomicLong#addAndGet()` method which would keep the counter truly atomic at the cost of performance penalty under contention, there are some other well-known ways to achieve what we want here. To name a few, one way is the `j.u.c.a.LongAdder` class which implements a sharded atomic counter. Its downsides are the higher read cost and the memory footprint. Another approach might be to accumulate the number of operations in a thread-local counter and periodically flush them via the `AtomicLong#addAndGet()` call. That's a certainly viable way to build an eventually consistent atomic counter, but today we'll consider a simpler approach that comes at the cost of concurrent increments loss.

You could say that the above example sounds artificial and you would be not far away from being absolutely correct. Nevertheless, the use case is good enough for today's experiment.

So, if we use acquire/release operations to build a lossy counter, we should get something like the following:
```java
public class LossyCounter extends AtomicLong {
    
    public long addAndGetLossy(long delta) {
        long value = getAcquire();
        long newValue = value + delta;
        setRelease(newValue);
        return newValue;
    }
}
```

We're going to benchmark this counter with other approaches, including atomic increments. While this wouldn't be an apple-to-apple comparison in terms of the counter operation guarantees, our goal is to get some understanding of the performance implications for different semantics and types of atomic operations.

The JMH benchmark we're going to use may be found [here](https://github.com/puzpuzpuz/java-concurrency-samples/blob/f5aaf0898408927918a16b649ddc8df54879957e/src/test/java/io/puzpuzpuz/atomic/LossyCounterBenchmark.java). Our test stand is a laptop with i7-1185G7 x86-64 CPU with 4/8 cores running Ubuntu 20.04 and OpenJDK 17.0.1.

Let's first run the benchmark on a single thread:
```
Benchmark                                      Mode  Cnt   Score   Error  Units
LossyCounterBenchmark.testAtomicCas            avgt   10  11.740 ± 0.040  ns/op
LossyCounterBenchmark.testAtomicIncrement      avgt   10   6.509 ± 0.027  ns/op
LossyCounterBenchmark.testBaseline             avgt   10   3.454 ± 0.004  ns/op
LossyCounterBenchmark.testLossyAcquireRelease  avgt   10   3.777 ± 0.158  ns/op
LossyCounterBenchmark.testLossyDefault         avgt   10   8.788 ± 0.016  ns/op
```

The testAtomicCas result here stands for a `compareAndSet` loop which is an awful way to do atomic increments on a counter. Not a big surprise that it showed the worst result. Then, testAtomicIncrement stands for the `addAndGet` operation, the default way to build an atomic counter. The baseline is nothing more than random number generation which is done as a part of all other benchmarks. Finally, testLossyAcquireRelease is our lossy counter while testLossyDefault stands for the same counter, but with the default operation semantics.

You may notice that our lossy counter adds almost nothing on top of the baseline and that's expected. The thing is that acquire/release semantics are no-op on x86 when it comes to ordinary loads (`get`) and stores (`set`). Read this [blog post](https://research.swtch.com/hwmm) from Russ Cox if you want to learn more about HW memory models.

Let's run the benchmark on 8 threads now:
```
Benchmark                                      Mode  Cnt     Score    Error  Units
LossyCounterBenchmark.testAtomicCas            avgt   10  1163.104 ± 62.158  ns/op
LossyCounterBenchmark.testAtomicIncrement      avgt   10   139.210 ±  0.571  ns/op
LossyCounterBenchmark.testBaseline             avgt   10     6.549 ±  0.029  ns/op
LossyCounterBenchmark.testLossyAcquireRelease  avgt   10    20.058 ±  0.186  ns/op
LossyCounterBenchmark.testLossyDefault         avgt   10   253.887 ±  3.668  ns/op
```

As expected, the CAS-based counter is a terrible idea. The `addAndGet` (`LOCK XADD` on x86) atomic counter does a much better job. Of course, a `LongAdder`, being used to build an atomic counter, would do even better under contention, but we're not interested in atomic counters now.

Interestingly, the testLossyDefault counter is almost 2x slower than the atomic one. That should be explained by the price of two full memory barriers executed on each increment operation in that lossy counter. Finally, the acquire/release lossy counter is the doubtless winner of our unfair competition.

The above benchmark and the lossy counter approach should be taken with a grain of salt. My only intention was to demonstrate that weaker memory semantics may yield better performance of your code, at least in a niche use case. However, the performance advantage may be insignificant in your concrete application, yet it will certainly come at the cost of more complex and, thus, less maintainable code. So, be mindful when using the new memory semantics.

Next time we're going to build atomic memory snapshots based on a seqlock and discuss whether it's a good idea to do so. See you!