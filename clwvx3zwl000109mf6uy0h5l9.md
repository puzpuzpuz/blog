---
title: "An mmap-based hash table optimization"
seoDescription: "mmap is a controversial topic among the database community. But it does a decent job for an analytical database and unlocks some optimizations."
datePublished: Sat Jun 01 2024 09:35:46 GMT+0000 (Coordinated Universal Time)
cuid: clwvx3zwl000109mf6uy0h5l9
slug: an-mmap-based-hash-table-optimization
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/NIJuEQw0RKg/upload/b057886190582173a306c4057f3e9e4e.jpeg
tags: performance, databases, data-structures

---

mmap is a [controversial](https://db.cs.cmu.edu/mmap-cidr2022/) [topic](https://www.symas.com/post/are-you-sure-you-want-to-use-mmap-in-your-dbms) [among](https://ravendb.net/articles/re-are-you-sure-you-want-to-use-mmap-in-your-database-management-system) the database community. The common belief is that it's no good for a database, but in my experience, it does a decent job for an analytical database, when you want to scan columns stored in a column-oriented format. Moreover, it unlocks some optimizations, unavailable with other file I/O APIs. This brief note describes one such optimization recently [added](https://github.com/questdb/questdb/pull/4435) to QuestDB by [Jaromir Hamala](https://x.com/jerrinot).

As you may already know, hash tables play a [crucial role](https://puzpuzpuz.dev/multithreaded-scatter-gather-execution-model-for-analytical-queries) not only in hash joins, but in parallel GROUP BY execution. There is no "silver bullet" design for a hash table, so having data structures specialized for a given scenario is the key to efficient query execution. QuestDB's SQL engine currently uses 6 hash tables written in Java and one written in C++. The numbers are very likely to grow in the future.

The hash table of our interest is aimed for GROUP BY queries over a single varchar key, like the following one:

```sql
SELECT URL, COUNT(*) AS c
FROM hits
ORDER BY c DESC
LIMIT 10;
```

It's a hash table with linear probing. The data is stored in a contiguous chunk of native memory and has the following layout for key/value pairs:

```markdown
| Hash code 32 LSBs |    Size    | Varchar pointer | Value columns 0..V |
+-------------------+------------+-----------------+--------------------+
|       4 bytes     |  4 bytes   |     8 bytes     |         -          |
+-------------------+------------+-----------------+--------------------+
```

The trick is that the 3rd field here contains pointers to mmapped memory. So, the hash table itself doesn't hold copies of varchar byte arrays but instead points at external (stable) memory. This way, we don't need to allocate additional memory and do a varchar copy. Another nice side effect is a lower memory footprint, all thanks to page cache memory being elastic, i.e. the OS evicts pages on memory pressure and reloads them from the disk on future access.

Based on our benchmarks, this hash table is 10-30% faster than our "default" [hash table](https://questdb.io/blog/building-faster-hash-table-high-performance-sql-joins/). While not much, in the case of slow queries, the execution time is reduced by several seconds, so it's worth it.

Have fun coding and see you next time.