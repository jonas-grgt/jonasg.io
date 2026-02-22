---
title: "Why Kafka Streams JVM Looks Healthy Before Getting OOMKilled"
head_title: "Why Kafka Streams JVM Looks Healthy Before Getting OOMKilled"
description: How to tame RocksDB memory consumption in Kafka Streams.
date: 2025-12-17
tags:
  - java
  - spring-boot
  - kafka
  - kafka-streams
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

How to tame RocksDB memory consumption in Kafka Streams.
<!--more-->
---

## The memory problem

Kafka Streams applications often behave perfectly in development and then mysteriously fall apart once deployed to
Kubernetes. Pods get OOMKilled, memory usage steadily climbs, and scaling replicas only seems to make things worse.

At first glance, this feels like a JVM or maybe even a Kubernetes problem.

- You’ve defined k8s memory limits.<br/>
- You’ve sized your pods generously.<br/>
- And yet… Kafka Streams keeps eating memory until Kubernetes pulls the plug.<br/>

The root issue is subtle but critical:

> Kafka Streams assumes it can use all memory available - not aware of the memory limits imposed by Kubernetes.

## Where does Kafka Streams eat memory?

But how can that be? Isn’t the JVM since Java 17, inside the container already aware of pod memory limits?

Yes but Kafka Streams, more precisely **RocksDB** isn't.

The main culprit is **stateful processing**.

Kafka Streams state-stores are backed by **RocksDB** by default.  
RocksDB uses **off-heap memory**, completely outside the JVM heap.

This means:

- The JVM may look perfectly healthy
- Garbage collection may be calm

And yet…  
Total process memory keeps growing.

From Kubernetes’ point of view, there is no distinction between:

- JVM heap
- JVM off-heap
- Native memory used by RocksDB

## How can we tame Kafka Streams memory usage?

Kafka Streams does give us a hook to control how much memory RocksDB is allowed to consume.

That hook is `org.apache.kafka.streams.state.RocksDBConfigSetter` set through:

```java
streamProperties.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG,
                     CustomRocksDbConfigSetter .class);
```

It allows you to configure the [`RocksDBStore`](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/state/internals/RocksDBStore.java).

It exposes a large number of configuration options with subtle interactions and non-obvious trade-offs.

Rather than trying to reason about every RocksDB knob, it’s much more effective to focus on the few settings that
actually dominate memory usage.

### Read Cache

The **BlockCache** is a dedicated area of *off-heap memory* used to speed up read operations by caching data blocks read
from disk.

By default, RocksDB allocates a separate BlockCache per RocksDB instance, with a default size of approximately 50MiB.

Since Kafka Streams creates one RocksDB instance per state-store partition per segment (segment is only applicable for
Versioned - and - Windowed - Stores), this means that for a state-store with 10 partitions and one segment, the default
BlockCache usage would be roughly:

10 × 50 MiB = 500MiB

**500MiB** just for one state-store read caching!

### Write Buffer

RocksDB uses the concept of **MemTables** as a write buffer.

By default, Kafka Streams does not set a [
`WriteBufferManager`](https://github.com/facebook/rocksdb/blob/5a06787a26ee4035b5c7d46ea4b3d80fc9bc02c9/java/src/main/java/org/rocksdb/WriteBufferManager.java#L11)
and therefore **does not set a unified, global memory limit for all MemTables across all partitions**.

## Unified global Memory limit

A RocksDB instance is created per partition (segment), hence even if we apply strict memory limits we still need to
multiply them by the number of partitions to be safe.

This can become convoluted, especially if you have:

- a different number of partitions per environment
- the number of state-stores will differ per application

Therefore, I advocate creating a **global memory limit** that is used by all stores in your application.
It greatly simplifies reasoning about memory usage.

```java
import java.util.Map;

public class CustomRocksDbConfigSetter implements RocksDBConfigSetter {

    // Total cache size (read and write) shared across all RocksDB instances in this JVM
    private static final long CACHE_SIZE_BYTES = 512L * 1024 * 1024; // 512 MiB

    // Total write buffer (MemTables) size shared across all RocksDB instances
    private static final long WRITE_BUFFER_SIZE_BYTES = 125L * 1024 * 1024; // 125 MiB

    // RocksDB data block size
    private static final long BLOCK_SIZE_BYTES = 64L * 1024; // 64 KB

    private static final Cache SHARED_CACHE;

    private static final WriteBufferManager SHARED_WRITE_BUFFER_MANAGER;

    static {
        // Class initialization is guaranteed to be thread-safe by the JVM.
        // These objects are safely published to all threads.
        SHARED_CACHE = new LRUCache(CACHE_SIZE_BYTES);
        SHARED_WRITE_BUFFER_MANAGER =
                new WriteBufferManager(WRITE_BUFFER_SIZE_BYTES, SHARED_CACHE);
    }

    public CustomRocksDbConfigSetter() {
        // Instantiated by Kafka Streams (RocksDBStore)
    }

    @Override
    public void setConfig(String storeName, Options options, Map<String, Object> configs) {

        // Reuse the config so not to lose the default setting you're not overriding.
        var tableConfig = (BlockBasedTableConfig) options.tableFormatConfig();

        // Read path tuning
        tableConfig.setBlockCache(SHARED_CACHE);
        tableConfig.setBlockSize(BLOCK_SIZE_BYTES);
        // By default, index, filter, and compression dictionary blocks are cached outside of block cache.
        // Setting this prevents this and makes sure to cache index and filter blocks in block cache.
        tableConfig.setCacheIndexAndFilterBlocks(true);

        options.setTableFormatConfig(tableConfig);

        // Write path tuning
        options.setWriteBufferManager(SHARED_WRITE_BUFFER_MANAGER);
    }
}
```

With the above configuration, RocksDB memory is now bounded to 512MiB.

No matter how many:

- state-stores
- partitions
- segments

exist in your application.

This already removes the biggest source of unbounded memory growth.
But we are not done yet.

## Sizing your Pod

If you have a Kubernetes memory limit of **2GiB**, you must account for the "invisible" RocksDB overhead.

| Component            |  Size   |                                                                  Reasoning |
|----------------------|:-------:|---------------------------------------------------------------------------:|
| RocksDB Shared Cache | 512MiB  |             This covers both the Block Cache and the Write Buffer Manager. |
| JVM Heap (`-Xmx`)    | 1024MiB | Leaves room for your application logic and Kafka Streams internal objects. |
| Overhead/Metaspace   | 512MiB  |                  Covers JVM stack threads, Metaspace, and native overhead. |

The custom implementation of `RocksDBConfigSetter` defines static values.
Maybe your application is more write heavy than read and another is not so you'll want to set these values dynamically.
And maybe you want different limits per environment, or maybe you want a shared **configurable** class for all your
applications?

In the [next part](https://jonasg.io/posts/externalize-rocksdb-configuration-in-spring-boot/), I’ll show you how to move these magic numbers into your `application.yaml` and create a reusable,
**spring-aware** `RocksDBConfigSetter` that works across all your stream processing application.
