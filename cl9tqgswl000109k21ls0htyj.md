# Thread-Local State in Go, Huh?

We all know that there is no such thing as thread-local state in Go. Yet, there is a trick that would help you to retain the thread identity at least on the hot path. This trick would be helpful if you're trying to implement a [striped counter](https://github.com/puzpuzpuz/xsync#counter) (wink-wink, `j.u.c.a.LongAdder` from Java), or a [BRAVO lock](https://github.com/puzpuzpuz/xsync#rbmutex), or any kind of a data structure with striped state.

This brief post is based on [the talk](https://github.com/puzpuzpuz/talks/blob/c1839354447cf9092d23f90986fd128f9f3f6563/2021-ru-is-time-to-resync/slides.pdf) I gave a while ago. I assume that you're familiar with the concept of [state striping](https://www.baeldung.com/java-longadder-and-longaccumulator#dynamic-striping) in concurrent counters since it's important for understanding the end application.

First of all, while Golang assigns identifiers to goroutines, it doesn't expose them. That's by [the design](https://golang.org/doc/faq#no_goroutine_id):
> Goroutines do not have names; they are just anonymous
workers. They expose no unique identifier, name, or data
structure to the programmer. Some people are surprised by
this, expecting the go statement to return some item that can
be used to access and control the goroutine later.
>
> The fundamental reason goroutines are anonymous is so that
the full Go language is available when programming
concurrent code. By contrast, the usage patterns that develop
when threads and goroutines are named can restrict what a
library using them can do.

The same applies to the worker threads used by the Golang scheduler to run your goroutines. But the whole idea of a striped counter depends on being able to identify the current thread, so that subsequent calls are (most of the time) run on the same CPU core and, hence, avoid contention.

There are two straightforward approaches to the problem:
1. CPUID x86 instruction - not portable to other architectures and also requires some assembly or FFI calls to be used.
2. gettid(2) Linux-only system call - not portable to other OSes.

Both of these options are non-versatile and, ideally, we want to have a Go-native and cross-platform solution. Luckily there is one.

I'm talking of [`sync.Pool`](https://pkg.go.dev/sync#Pool). If you're familiar with its [source code](https://github.com/golang/go/blob/e09bbaec69a8ff960110e13eabb3bef5331ecb0c/src/sync/pool.go), you already know that it uses thread-local pools under the hood. If we allocate a struct and place it in the pool, the next time we request it one the same thread (but not necessarily same goroutine) we should get the same struct.

Let's take a look at a fragment of the `xsync.Counter`'s code:
```go
// pool for P tokens
var ptokenPool sync.Pool

// ptoken is used to point at the current OS thread (P)
// on which the goroutine is run; exact identity of the thread,
// as well as P migration tolerance, is not important since
// it's used to as a best effort mechanism for assigning
// concurrent operations (goroutines) to different stripes of
// the counter.
type ptoken struct {
	idx uint32
}

// Counter is a striped int64 counter.
type Counter struct {
	stripes []cstripe
	mask    uint32
}

type cstripe struct {
	c int64
	// The padding prevent false sharing.
	pad [cacheLineSize - 8]byte
}

func NewCounter() *Counter {
	// Consider the number of CPU cores
	// when deciding on the number of stripes.
	nstripes := nextPowOf2(parallelism())
	c := Counter{
		stripes: make([]cstripe, nstripes),
		mask:    nstripes - 1,
	}
	return &c
}

// Value returns the current counter value.
func (c *Counter) Value() int64 {
	v := int64(0)
	for i := 0; i < len(c.stripes); i++ {
		stripe := &c.stripes[i]
		v += atomic.LoadInt64(&stripe.c)
	}
	return v
}

// Add adds the delta to the counter.
func (c *Counter) Add(delta int64) {
	// Pick up a token from the pool. If Add was called recently
	// on the same thread, we'll probably get the same ptoken.
	t, ok := ptokenPool.Get().(*ptoken)
	if !ok {
		// Allocate a new token and pick up a random stripe index.
		t = new(ptoken)
		t.idx = fastrand() & c.mask
	}
	for {
		stripe := &c.stripes[t.idx]
		cnt := atomic.LoadInt64(&stripe.c)
		if atomic.CompareAndSwapInt64(&stripe.c, cnt, cnt+delta) {
			// We were able to update the stripe, so all done.
			break
		}
		// CAS failed, so there is some contention over the stripe.
		// Give a try with another randomly selected stripe.
		t.idx = fastrand() & c.mask
	}
	// Return ptoken back to the pool, so that another goroutine
	// running on the same thread can use it.
	ptokenPool.Put(t)
}
```

Here, in the `Add` method, we using the `ptoken` structs to hold (not-so) thread-local state. Once we obtain a `ptoken`, we try to change the corresponding stripe in a CAS-based loop. This allows the goroutines to self-organize: they detect contention via a failed CAS operation and then change the stripe. The goal is to avoid contention due to unlucky thread-to-stripe distribution.

You may ask if piggybacking on a `sync.Pool`'s implementation detail is worth hassle. My answer would be "no, unless you really know what you're doing". Say, single-threaded performance of a primitive atomic `int64` would be better. There is also an overhead in the `Value()` method since it needs to read values from all stripes. So, this trick is certainly from the "don't try that at home" category. But if you aim for scalability of your write operations, it's certainly worth it:
```bash
$ go test -benchmem -run=^$ -bench "Counter|Atomic"
goos: linux
goarch: amd64
pkg: github.com/puzpuzpuz/xsync/v2
cpu: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
BenchmarkCounter-8       	409773502	         2.909 ns/op	       0 B/op	       0 allocs/op
BenchmarkAtomicInt64-8   	92472007	        14.09 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/puzpuzpuz/xsync/v2	3.024s
```

If you run the same benchmark on a machine with more cores, the `int64`'s result would only get worse due to contention.

Both `Map` and `MapOf`, concurrent hash maps from [`xsync` library](https://github.com/puzpuzpuz/xsync), use a variation of a striped counter internally to track the current map size. Naturally, they do a counter increment or decrement on each write operation, but read the counter value rarely when a resize happens.

One more example of application of this trick is [`RBMutex`](https://github.com/puzpuzpuz/xsync#rbmutex), a reader biased reader/writer mutual exclusion lock that implements BRAVO algorithm. I'm leaving learning the internals of this one to the curious reader.

As promised, the post is a short one, so that's it for today. Have fun coding and see you next time.