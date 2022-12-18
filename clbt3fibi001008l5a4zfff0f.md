# BuzzwordBusters: What Does Lock-Free, Wait-Free Really Mean?

It seems to be a common belief that code which uses mutexes/locks/synchronized methods is "slow" and, as soon as you replace them with atomics, your code becomes fast and lock-free. Atomic operations don't make your code wait-free, lock-free, or even obstruction-free. This tiny blog post is dedicated to the above definitions.

Wait-freedom means that any thread can make progress in a finite number of steps regardless of external factors, such as other threads blocking. A trivial example of a wait-free data structure is an atomic counter (in x86 it would use a `LOCK XADD` instruction), e.g. Java's `j.u.c.AtomicInteger`.

```java
public class Counter {
    
    private final AtomicInteger cnt = new AtomicInteger();
    
    public int add(int delta) {
        return cnt.addAndGet(delta);
    }    
    
    public int get() {
        return cnt.get();
    }
}
```

Lock-freedom means that the application as a whole can make progress regardless of anything. So, while individual threads may be blocked, at least one of them would be making progress. A trivial example would be the same atomic counter based on a loop with a CAS operation.

```java
public class Counter {

    private static final AtomicIntegerFieldUpdater<Counter> updater =
        AtomicIntegerFieldUpdater.newUpdater(Counter.class, "cnt");
    private volatile int cnt;

    public int add(int delta) {
        int cur;
        do {
            cur = cnt;
        } while (!updater.compareAndSet(this, cur, cur + delta));
        return cur + delta;
    }

    public int get() {
        return cnt;
    }
}
```

Obstruction-freedom means that a thread can make progress only if there is no contention from other threads. This guarantee is the weakest one on the list. It's hard to illustrate this definition with a simple enough example I'm not aware of a trivial example of this one, but you may refer to [this paper](https://ieeexplore.ieee.org/document/1203503).

Is it fair enough to say that wait-free data structures and algorithms are faster than lock-free ones and that lock-freedom means something better than obstruction-freedom and blocking code? Not really. You may limit yourself with wait-freedom if you want to limit the maximum latency of an individual operation, e.g. if you're building a real-time OS. But in most cases, you should consider all possible algorithms and their combinations. For instance, your data structure may be quite fast while it implements wait-free or lock-free reads and blocking writes based on a striped lock (wink-wink Java's `ConcurrentHashMap` and xsync's `Map`/`MapOf`).

If you want to learn more about multithreaded programming and scalable concurrent data structures, I highly recommend Dmitry Vyukov's [old blog](https://www.1024cores.net/). Just go through all posts starting with the [intro one](https://www.1024cores.net/home/lock-free-algorithms/introduction).

Have fun coding and see you next time.