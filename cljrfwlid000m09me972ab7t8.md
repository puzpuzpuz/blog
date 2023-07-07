---
title: "A Few Thoughts on K-Word CAS"
seoDescription: "A few months ago I went through Efficient Multi-word Compare and Swap paper, so here are a few thoughts on the algorithm."
datePublished: Thu Jul 06 2023 17:46:52 GMT+0000 (Coordinated Universal Time)
cuid: cljrfwlid000m09me972ab7t8
slug: a-few-thoughts-on-k-word-cas
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/pxVOztBa6mY/upload/55fbf363651dcae3f3b9ec78734e4f2e.jpeg
tags: algorithms, multithreading, concurrency

---

A few months ago I went through [Efficient Multi-word Compare and Swap](https://arxiv.org/abs/2008.02527) paper, so here are a few thoughts on the algorithm. Long story short, I have mixed feelings about this k-word CAS algorithm. It focuses on nice properties of the [CAS operation](https://en.wikipedia.org/wiki/Compare-and-swap), such as lock-freedom and linearizability, while overlooking atomic k-word reads.

The core idea of the algorithm is that writers do a loop of CAS operations on each stored word. They try to swap the value (or a pointer) with a pointer to the so-called operation descriptor structure. The structure includes old and new values, as well as the operation status. Once a writer successfully completes all CASes, it does one more CAS on the status field marking the operation as complete. So, the algorithm requires k+1 single-word CAS operations per k-word CAS.

Indeed, this k-word CAS algorithm is lock-free and linearizable, but if you also want to be able to read all k words atomically, the algorithm won't be of any help. Of course, you can do a no-op k-word CAS to do an atomic read, but such read may be costly. One more downside is that being able to swap a primitive value with a pointer means that the algorithm is not meant to be used in any language with a GC. In theory, it can be still used in languages with non-moving GC, such as Golang, but with unsafe things like pointer tagging. Also, if you're fine with less plain memory layout, then, say, in Java the algorithm may be modified to use an array of `AtomicReference<Object>` instead of an array of primitive type (usually, `long[]`).

If lock-freedom is not a must, a [seqlock-like](https://en.wikipedia.org/wiki/Seqlock) approach might do just fine. The sequence field could be used to preserve exclusive writer access, as well as enough help for readers to determine whether they got an atomic snapshot of all k-words. If lock-freedom is important, then in languages with GC there is an even simpler option which is to use an immutable data structure and let the writers do a single-word CAS swapping the pointer to the old values with the new one. Finally, a good old lock could be used, optionally a much more scalable reader-writer one.