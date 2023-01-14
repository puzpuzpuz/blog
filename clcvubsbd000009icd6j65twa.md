# Multithreaded Scatter-Gather Execution Model for Analytical Queries

Today we'll be discussing an approach used in some analytical databases to speed up the execution of queries at the cost of additional HW resources, namely CPU cores and memory. The query execution model is usually referred to as "scatter-gather", yet it's hard to find an article with a good amount of detail for this model (at least, I failed to do that), so I decided to write a brief post on the topic.

To be slightly more concrete, here is a simple example of a query that can benefit from scatter-gather execution:

```sql
SELECT sensor_id, max(temp), min(temp), avg(temp)
FROM temperature
WHERE sensor_id IN (402, 1202, 3983)
GROUP BY sensor_id;
```

To be able to use the scatter-gather model, the storage format used in the database needs to support parallelism, i.e. it should be possible to divide the on-disk data into individual parts that can be independently scanned. In practice, this usually means log-structured storage format which is in many cases also columnar.

Now, let's consider the execution of the above query.

### Scatter

For the sake of simplicity, let's assume that there is no index on the \`sensor\_id\` column and table columns are stored in append-only log files. In this case, it's trivial to split the log files to be scanned into N chunks based on offsets. That's basically what the original thread does when scattering the query execution. Each of the chunks is serialized into a task object/struct along with the query execution plan details, such as selected columns, aggregate functions and filter, and written into an in-memory queue to be picked up by worker threads.

Let's refer to the original thread as the orchestrator thread. If the original thread belongs to the same thread pool, it must participate as a worker, i.e. it should poll tasks from the queue and execute them. That's to avoid starvation and deadlocks in the situation when all threads try to orchestrate their own queries.

When a worker thread picks up a task, it starts executing it. It scans through the data, applies the filter (`WHERE` clause) and calculates the intermediate result for each aggregate function (`min`, `max` and `avg` in our example). A convenient way to store the result is a hash table holding `<int, <int, int, float>>` key-value pairs. Here, we use `int` type for the key assuming that `sensor_id` is an integer column and each `<int, int, float>` tuple stands for the intermediate results of our three aggregate functions.

Once a task gets executed, the worker needs to write the intermediate result (hash table) into an in-memory queue to be consumed and gathered by the orchestrator thread.

### Gather (and merge)

The orchestrator thread needs a hash table to store query results. Initially, it's empty, but as soon as the thread consumes (gathers) a result from one of the workers, it needs to merge two tables. The merge is simple thanks to the natural properties of the aggregate functions:

```java
min(A ∪ B) = min(min(A), min(B))
max(A ∪ B) = max(max(A), max(B))
count(A ∪ B) = count(A) + count(B)
sum(A ∪ B) = sum(A) + sum(B)
avg(A ∪ B) = sum(A ∪ B) / count(A ∪ B)
```

It's not a complete list, but most scalar aggregate functions assume little data to be stored as their state, so they fit into scatter-gather nicely.

As soon as the orchestrator gathers and merges the last task result, it has the query result to be returned to the client.

### Conclusion

Scatter-gather(-merge) is a simple single-stage execution model. It works nicely for relatively trivial GROUP BY queries with an optional WHERE clause while more complex queries involving JOINs require a more complex multi-stage parallel execution.

Of course, [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl%27s_law) applies to the scatter-gather model, so if the serial part of the total work is significant, the speed up from parallelism will be humble. But if the database uses blocking disk I/O, there might be a benefit even in the degenerate case when each group has a single row to be scanned.

One more advantage of this model is that it applies to distributed databases naturally. We can easily swap "thread" with "node" in the above text with no other changes except for the storage requirement. The data has to be sharded across cluster nodes and the orchestrator node has to be aware of the sharding scheme so that it's aware of the data location when it distributes the work.

I'm interested in learning more about analytical query execution models and not only, so if you have anything to share, don't hesitate to write a comment. Have fun coding and see you next time.