---
title: ES_线程池_分析
date: 2020-04-19 14:37:14
tags: [Elasticsearch]
---

本文将介绍 ES 的线程池和请求等待队列等知识。

施工中

<!--more-->

## 线程池 参数介绍

`_cat/thread_pool`API 返回了线程池的相关信息，其中 `active` 表示活跃的线程数量，`queue` 表示正在等待中的任务数量，`queue_size`则表示当前 queue 的 size了。当队列中等待的任务超过了 `queue_size`时，请求将会被拒绝。

### 获取线程池配置

```console
// 两种 API 都可以查询线程池的状态
GET /_cat/thread_pool?v&h=id,core,name,type,active,queue,queue_size,rejected,largest,min,max
GET /_nodes/thread_pool

id                     name                type                  active queue queue_size rejected largest min max
K65T0Ik-TW-KCAHLzq4f_A analyze             fixed                      0     0         16        0       0   1   1
K65T0Ik-TW-KCAHLzq4f_A ccr                 fixed                      0     0        100        0       0  32  32
K65T0Ik-TW-KCAHLzq4f_A fetch_shard_started scaling                    0     0         -1        0       0   1  16
K65T0Ik-TW-KCAHLzq4f_A fetch_shard_store   scaling                    0     0         -1        0       0   1  16
K65T0Ik-TW-KCAHLzq4f_A flush               scaling                    0     0         -1        0       4   1   4
K65T0Ik-TW-KCAHLzq4f_A force_merge         fixed                      0     0         -1        0       0   1   1
K65T0Ik-TW-KCAHLzq4f_A generic             scaling                    0     0         -1        0       5   4 128
K65T0Ik-TW-KCAHLzq4f_A get                 fixed                      0     0       1000        0       8   8   8
K65T0Ik-TW-KCAHLzq4f_A index               fixed                      0     0        200        0       8   8   8
K65T0Ik-TW-KCAHLzq4f_A listener            fixed                      0     0         -1        0       4   4   4
K65T0Ik-TW-KCAHLzq4f_A management          scaling                    1     0         -1        0       4   1   5
K65T0Ik-TW-KCAHLzq4f_A refresh             scaling                    0     0         -1        0       2   1   4
K65T0Ik-TW-KCAHLzq4f_A rollup_indexing     fixed                      0     0          4        0       0   4   4
K65T0Ik-TW-KCAHLzq4f_A search              fixed_auto_queue_size      0     0       1000        0      13  13  13
K65T0Ik-TW-KCAHLzq4f_A search_throttled    fixed_auto_queue_size      0     0        100        0       0   1   1
K65T0Ik-TW-KCAHLzq4f_A security-token-key  fixed                      0     0       1000        0       0   1   1
K65T0Ik-TW-KCAHLzq4f_A snapshot            scaling                    0     0         -1        0       0   1   4
K65T0Ik-TW-KCAHLzq4f_A warmer              scaling                    0     0         -1        0       0   1   4
K65T0Ik-TW-KCAHLzq4f_A watcher             fixed                      0     0       1000        0       0  40  40
K65T0Ik-TW-KCAHLzq4f_A write               fixed                      0     0        200        0       8   8   8
```

| Field Name       | Alias | Description                                                  |
| ---------------- | ----- | ------------------------------------------------------------ |
| `type`           | `t`   | The current (*) type of thread pool (`fixed` or `scaling`)   |
| **`active`**     | `a`   | The number of active threads in the current thread pool      |
| `size`           | `s`   | The number of threads in the current thread pool             |
| **`queue`**      | `q`   | The number of tasks in the queue for the current thread pool |
| **`queue_size`** | `qs`  | The maximum number of tasks permitted in the queue for the current thread pool |
| **`rejected`**   | `r`   | The number of tasks rejected by the thread pool executor     |
| `largest`        | `l`   | The highest number of active threads in the current thread pool |
| `completed`      | `c`   | The number of tasks completed by the thread pool executor    |
| `min`            | `mi`  | The configured minimum number of active threads allowed in the current thread pool |
| `max`            | `ma`  | The configured maximum number of active threads allowed in the current thread pool |
| `keep_alive`     | `k`   | The configured keep alive time for threads                   |

### [线程池类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html#modules-threadpool)

对于不同的功能，节点内部有不同的线程池来处理，区分线程池可以使节点更好的管理线程内存的消费。

许多线程池都由对应的队列，用于存储 Pending 请求而不是丢弃。

以下是各个请求的功能，按需查看。

- **`generic`**

  For generic operations (for example, background node discovery). Thread pool type is `scaling`.

- **`search`**

  For count/search/suggest operations. Thread pool type is `fixed_auto_queue_size` with a size of `int((# of available_processors * 3) / 2) + 1`, and initial queue_size of `1000`.

- **`search_throttled`**

  For count/search/suggest/get operations on `search_throttled indices`. Thread pool type is `fixed_auto_queue_size` with a size of `1`, and initial queue_size of `100`.

- **`get`**

  For get operations. Thread pool type is `fixed` with a size of `# of available processors`, queue_size of `1000`.

- **`analyze`**

  For analyze requests. Thread pool type is `fixed` with a size of `1`, queue size of `16`.

- **`write`**

  For single-document index/delete/update and bulk requests. Thread pool type is `fixed` with a size of `# of available processors`, queue_size of `200`. The maximum size for this pool is `1 + # of available processors`.

- **`snapshot`**

  For snapshot/restore operations. Thread pool type is `scaling` with a keep-alive of `5m` and a max of `min(5, (# of available processors)/2)`.

- **`warmer`**

  For segment warm-up operations. Thread pool type is `scaling` with a keep-alive of `5m` and a max of `min(5, (# of available processors)/2)`.

- **`refresh`**

  For refresh operations. Thread pool type is `scaling` with a keep-alive of `5m` and a max of `min(10, (# of available processors)/2)`.

- **`listener`**

  Mainly for java client executing of action when listener threaded is set to `true`. Thread pool type is `scaling` with a default max of `min(10, (# of available processors)/2)`.

- **`fetch_shard_started`**

  For listing shard states. Thread pool type is `scaling` with keep-alive of `5m` and a default maximum size of `2 * # of available processors`.

- **`fetch_shard_store`**

  For listing shard stores. Thread pool type is `scaling` with keep-alive of `5m` and a default maximum size of `2 * # of available processors`.

- **`flush`**

  For [flush](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-flush.html), [synced flush](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-synced-flush-api.html), and [translog](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html) `fsync` operations. Thread pool type is `scaling` with a keep-alive of `5m` and a default maximum size of `min(5, (# of available processors)/2)`.

- **`force_merge`**

  For [force merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-forcemerge.html) operations. Thread pool type is `fixed` with a size of 1 and an unbounded queue size.

- **`management`**

  For cluster management. Thread pool type is `scaling` with a keep-alive of `5m` and a default maximum size of `5`.

### 线程池的参数

线程池有三种不同的模式，分别为 `fixed`, `scaling`, `fixed-auto-queue-size`，对于不同的类型的线程池，配置参数也不相同。

**fixed**

```yaml
thread_pool:
    write:
        size: 30
        queue_size: 1000
```

**scaling**

`scaling`类型的线程池持有动态数量的线程。线程数量随集群负载 和 `core & max`的值成比例变化。

`keep_alive`参数决定一个线程空闲多久会被移除。

```yaml
thread_pool:
    warmer:
        core: 1
        max: 8
        keep_alive: 2m
```

[**fixed-auto-queue-size**](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html#fixed-auto-queue-size)

实验性功能，请根据具体版本，查看对应官方文档。

### 处理器核数设置

处理器核数是自动感知的，线程池设置也是基于此自动设置的，可以通过以下的配置来修改该值，如果你确定你的场景需要修改这个值。 

>  NOTE: 这是个专家级的配置，会牵涉到许多不同的配置，比如说改变 GC 线程的数量，pinning processes to cores等等。

```yaml
processors: 2
```

此处介绍几个需要覆盖`processors`值的场景：

1. 当我们需要 ES只使用一部分处理器性能时，比如一台机器上运行了2个 ES 实例，我们会希望每个实例只使用一半的 CPU 资源，就可以手动修改处理器核数。
2. 当ES 错误的检测了节点的 CPU 核数时，这个值可以通过 GET /_nodes，查看 OS 对象的值来获取。

## 是否需要变更线程池的大小

Elasticsearch 默认的线程设置已经是很合理。

对于所有的线程池（除了 `search` ），**线程个数是根据 CPU 核心数设置的**。 如果你有 8 个核，你可以同时运行的只有 8 个线程，只分配 8 个线程给任何特定的线程池是有道理的。

`Search`线程池设置的大一点，配置为 `int（（ 核心数 ＊ 3 ）／ 2 ）＋ 1` 。

你可能会认为某些线程可能会阻塞（如磁盘上的 I／O 操作），所以你才想加大线程的。对于 Elasticsearch 来说这并不是一个问题：因为大多数 I／O 的操作是由 Lucene 线程管理的，而不是 Elasticsearch。

此外，线程池通过传递彼此之间的工作配合。你不必再因为它正在等待磁盘写操作而担心网络线程阻塞， 因为网络线程早已把这个工作交给另外的线程池，并且网络进行了响应。

最后，你的处理器的计算能力是有限的，拥有**更多的线程会导致你的处理器频繁切换线程上下文**。 一个处理器同时只能运行一个线程。所以当它需要切换到其它不同的线程的时候，它会存储当前的状态（寄存器等等），然后加载另外一个线程。 如果幸运的话，这个切换发生在同一个核心，如果不幸的话，这个切换可能发生在不同的核心，这就需要在内核间总线上进行传输。

这个上下文的切换，会给 CPU 时钟周期带来管理调度的开销；在现代的 CPUs 上，开销估计高达 30 μs。也就是说线程会被堵塞超过 30 μs，如果这个时间用于线程的运行，极有可能早就结束了。

**所以，下次请不要调整线程池的线程数。如果你真 *想调整* ， 一定要关注你的 CPU 核心数，最多设置成核心数的两倍，再多了都是浪费。**



## 修改 Therad Pool Queue Size

当同时接收过多请求，队列等待请求数量大于 `queue_size` 时，额外的请求就会被拒绝。

单纯的增加线程数量，对于提高系统的性能，作用有限，参考[是否需要变更线程池的大小](#是否需要变更线程池的大小)

直接增加等待队列的大小是相对可取的方案。

```yml
thread_pool.search.queue_size: 500
#queue_size允许控制没有线程执行它们的挂起请求队列的初始大小。

thread_pool.search.size: 200
#size参数控制线程数，默认为核心数乘以5。

thread_pool.search.min_queue_size:10
#min_queue_size设置控制queue_size可以调整到的最小量。

thread_pool.search.max_queue_size: 1000
#max_queue_size设置控制queue_size可以调整到的最大量。

thread_pool.search.auto_queue_frame_size: 2000
#auto_queue_frame_size设置控制在调整队列之前进行测量的操作数。它应该足够大，以便单个操作不会过度偏向计算。

thread_pool.search.target_response_time: 6s
#target_response_time是时间值设置，指示线程池队列中任务的目标平均响应时间。如果任务通常超过此时间，则将调低线程池队列以拒绝任务。
```

### 示例场景

通过代码生成大量写请求，来测试请求过载的场景。

目前设置 write 线程数量为1，发送1W 个请求，还是没有监测到 queue队列的拥堵现象。

考虑多线程发送请求，提高压力。

//todo

## Reference

[ESV6.7 _cat/therad_pool api introductin](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/cat-thread-pool.html)

[ES Don't touch these setting](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dont-touch-these-settings.html)

[ES7.6 Thread Pool Type](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html#modules-threadpool)