---
title: "Externalize RocksDB configuration in spring-boot"
head_title: "Externalize RocksDB configuration in spring-boot"
description: How to make your RocksDB memory management configurable in spring-boot.
date: 2026-02-22
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

How to make your RocksDB memory management configurable in spring-boot.
<!--more-->
---

In the [previous post](https://jonasg.io/posts/why-kafka-streams-jvm-looks-healthy-before-getting-oomkilled/), 
we saw how to configure RocksDB to use a unified, 
bounded memory pool instead of silently consuming all available off-heap memory in Kafka Streams.

This configuration was rather hard-coded, as Kafka Streams itself instantiates the `RocksDBConfigSetter` class,
meaning all values have to be statically defined. 


In this post, I’ll show you how to configure your `RocksDBConfigSetter` using values derived from your Spring Boot `application.yml`.

By doing so, you will be able to  dynamically tune your RocksDB memory limits as they may differ per environment or 
even applications (as you can package it into a reusable auto-configuration that can be shared throughout your organization).

## From static to dynamic

**Step 1** is to hold the settings **statically**, as this is required by Kafka Streams.

```java
public class ConfigurableRocksDbConfigSetter implements RocksDBConfigSetter {

    private static Map<String, Object> configs = Map.of();

    public ConfigurableRocksDbConfigSetter(Map<String, Object> configuration) {
      ConfigurableRocksDbConfigSetter.configs = configuration;
    }
```

A Map might be too dynamic, which keys are present and what values do they hold?
Therefore we'll use a concrete type based upon the `AbstractConfig` you quite often find in the Kafka ecosystem.
```java
public class RocksDbConfig extends AbstractConfig {

    private static final ConfigDef CONFIG;

    public RocksDbConfig(Map<?, ?> originals) {
        this(originals, true);
    }

    public RocksDbConfig(Map<?, ?> originals, boolean doLog) {
        super(CONFIG, originals, doLog);
    }

    public static final String TOTAL_WRITE_BUFFER_SIZE_CONFIG =
            "kafka.streams.rocksdb.total-write-buffer-size";
    public static final long DEFAULT_TOTAL_WRITE_BUFFER_SIZE =
            75 * 1024 * 1024L;
    public static final String WRITE_BUFFER_SIZE_DOC =
            "Defines the total - over all stores and tasks - size of the write buffer.";

    public static final String TOTAL_READ_CACHE_SIZE_CONFIG =
            "kafka.streams.rocksdb.total-read-cache-size";
    public static final long DEFAULT_TOTAL_READ_CACHE_SIZE =
            125 * 1024 * 1024L;
    public static final String TOTAL_READ_CACHE_SIZE_DOC =
            "Defines the total - over all stores and tasks - size of the read cache.";

    public static final String BLOCK_SIZE_CONFIG =
            "kafka.streams.rocksdb.block-size";
    public static final long DEFAULT_BLOCK_SIZE =
            4096L;
    public static final String BLOCK_SIZE_DOC =
            "The data chunk size (Read Block Size) for RocksDB.";

    static {
        CONFIG = new ConfigDef()

                .define(
                        TOTAL_WRITE_BUFFER_SIZE_CONFIG,
                        Type.LONG,
                        DEFAULT_TOTAL_WRITE_BUFFER_SIZE,
                        Range.atLeast(0),
                        Importance.HIGH,
                        WRITE_BUFFER_SIZE_DOC)

                .define(
                        TOTAL_READ_CACHE_SIZE_CONFIG,
                        Type.LONG,
                        DEFAULT_TOTAL_READ_CACHE_SIZE,
                        Range.atLeast(0),
                        Importance.HIGH,
                        TOTAL_READ_CACHE_SIZE_DOC)

                .define(
                        BLOCK_SIZE_CONFIG,
                        Type.LONG,
                        DEFAULT_BLOCK_SIZE,
                        Range.atLeast(0),
                        Importance.HIGH,
                        BLOCK_SIZE_DOC);
    }

    public long totalWriteBufferSize() {
        return getLong(TOTAL_WRITE_BUFFER_SIZE_CONFIG, DEFAULT_TOTAL_WRITE_BUFFER_SIZE);
    }

    public long totalReadCacheSize() {
        return getLong(TOTAL_READ_CACHE_SIZE_CONFIG, DEFAULT_TOTAL_READ_CACHE_SIZE);
    }

    public long blockSize() {
        return getLong(BLOCK_SIZE_CONFIG, DEFAULT_BLOCK_SIZE);
    }

    public long getLong(String key, long defaultValue) {
        var value = originals().getOrDefault(key, defaultValue);
        if (value instanceof Number number) {
            return number.longValue();
        } else if (value instanceof String str) {
            return Long.parseLong(str);
        }
        return defaultValue;
    }
}

```

Of course you can extend this with additional properties as you require.

**Step 2** is to move from class initialization (which was guaranteed to be thread-safe by the JVM)
towards initializing the shared cache in the `setConfig` method while keeping it thread-safe.

```java
public class ConfigurableRocksDbConfigSetter implements RocksDBConfigSetter {

  private static volatile Cache SHARED_CACHE;

  private static volatile WriteBufferManager SHARED_WRITE_BUFFER_MANAGER;

  @Override
  public void setConfig(String storeName, Options options, Map<String, Object> configs) {
      var rocksDbConfig = new RocksDbConfig(ConfigurableRocksDbConfigSetter.configs);

      var tableConfig = (BlockBasedTableConfig) options.tableFormatConfig();

      if (SHARED_CACHE == null || SHARED_WRITE_BUFFER_MANAGER == null) {
          synchronized (ConfigurableRocksDbConfigSetter.class) {
              if (SHARED_CACHE == null) {
                  SHARED_CACHE = new LRUCache(
                          rocksDbConfig.totalReadCacheSize() + rocksDbConfig.totalWriteBufferSize());
              }
              if (SHARED_WRITE_BUFFER_MANAGER == null) {
                  SHARED_WRITE_BUFFER_MANAGER = new WriteBufferManager(rocksDbConfig.totalWriteBufferSize(),
                          SHARED_CACHE);
              }
          }
      }

      tableConfig.setBlockCache(SHARED_CACHE);
      // The data chunk size (Read Block Size) for RocksDB.
      tableConfig.setBlockSize(rocksDbConfig.blockSize());
      // put index and filter blocks inside the block cache
      tableConfig.setCacheIndexAndFilterBlocks(true);

      options.setTableFormatConfig(tableConfig);

      options.setWriteBufferManager(SHARED_WRITE_BUFFER_MANAGER);
  }
```

**Step 3** is wiring everything up with spring based properties.
Therefor we'll create a bean out of the `ConfigurableRocksDbConfigSetter`. 
But didn't Kafka Streams instantiate this class?
Although Kafka Streams instantiates the `RocksDBConfigSetter` itself, 
we register a Spring bean early in the **auto-configuration** phase. 
This ensures the static configuration map is populated *before* Kafka Streams creates its own instance.

```java
@AutoConfiguration(before = { KafkaAutoConfiguration.class })
@ConditionalOnClass({ KafkaStreams.class })
@EnableConfigurationProperties(RocksDbProperties.class)
public class KafkaStreamsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ConfigurableRocksDbConfigSetter ConfigurableRocksDbConfigSetter(RocksDbProperties properties) {
        return new ConfigurableRocksDbConfigSetter(properties.toMap());
    }
```

```java
@ConfigurationProperties(prefix = "kafka.streams.rocksdb")
public class RocksDbProperties {

    /**
     * Defines the total - over all stores and tasks - size of the write buffer.
     */
    private String totalWriteBufferSize = SizeParser.asMiB(RocksDbConfig.DEFAULT_TOTAL_WRITE_BUFFER_SIZE);

    /**
     * Defines the total - over all stores and tasks - size of the read cache.
     */
    private String totalReadCacheSize = SizeParser.asMiB(RocksDbConfig.DEFAULT_TOTAL_READ_CACHE_SIZE);

    /**
     * Defines the chunk size (Read Block Size) for RocksDB.
     */
    private String blockSize = SizeParser.asKiB(RocksDbConfig.DEFAULT_BLOCK_SIZE);

    public long getTotalWriteBufferSize() {
        return SizeParser.parse(totalWriteBufferSize);
    }

    public void setTotalWriteBufferSize(String totalWriteBufferSize) {
        this.totalWriteBufferSize = totalWriteBufferSize;
    }

    public long getTotalReadCacheSize() {
        return SizeParser.parse(totalReadCacheSize);
    }

    public void setTotalReadCacheSize(String totalReadCacheSize) {
        this.totalReadCacheSize = totalReadCacheSize;
    }

    public long getBlockSize() {
        return SizeParser.parse(blockSize);
    }

    public void setBlockSize(String blockSize) {
        this.blockSize = blockSize;
    }

    public Map<String, Object> toMap() {
        return Map.of(
                RocksDbConfig.TOTAL_WRITE_BUFFER_SIZE_CONFIG, getTotalWriteBufferSize(),
                RocksDbConfig.TOTAL_READ_CACHE_SIZE_CONFIG, getTotalReadCacheSize(),
                RocksDbConfig.BLOCK_SIZE_CONFIG, getBlockSize());
    }
}
```

Now you can define your RocksDB properties in your `application.yml` file:
```yaml
kafak.streams.rocksdb.total-write-buffer-size: 500MiB

```
I like to define sizes using human-readable units such as `MiB` and `KiB`, rather than raw byte values, 
as it makes configuration clearer and far less error-prone when tuning memory across environments.

```java
public final class SizeParser {

  private static final long KIB = 1024L;

  private static final long MIB = KIB * 1024L;

  private static final Pattern PATTERN = Pattern.compile("(?i)\\s*([0-9]+(?:\\.[0-9]+)?)\\s*(KiB|MiB|GiB)?\\s*");

  private SizeParser() {
  }

  public static long parse(String input) {
      if (input == null || input.trim().isEmpty()) {
          throw new IllegalArgumentException("Size value cannot be null or empty");
      }

      Matcher matcher = PATTERN.matcher(input);
      if (!matcher.matches()) {
          throw new IllegalArgumentException("Invalid size format: " + input);
      }

      double number = Double.parseDouble(matcher.group(1));
      String unit = matcher.group(2);

      if (unit == null) {
          // No suffix defaults to bytes
          return (long) number;
      }

      return switch (unit.toUpperCase()) {
          case "KIB" -> (long) (number * 1024L);
          case "MIB" -> (long) (number * 1024L * 1024L);
          case "GIB" -> (long) (number * 1024L * 1024L * 1024L);
          default -> throw new IllegalArgumentException("Unknown unit: " + unit);
      };
  }

  /**
    * Converts a byte count into a human-readable MiB string.
    * Example: 134217728 -> "128 MiB"
    */
  public static String asMiB(long bytes) {
      if (bytes < 0) {
          throw new IllegalArgumentException("Bytes cannot be negative");
      }

      double mib = (double) bytes / MIB;

      if (mib == Math.floor(mib)) {
          return String.format("%.0fMiB", mib);
      }

      return String.format("%.2fMiB", mib);
  }

  /**
    * Converts a byte count into a human-readable KiB string.
    * Example: 2048 -> "2 KiB"
    */
  public static String asKiB(long bytes) {
      if (bytes < 0) {
          throw new IllegalArgumentException("Bytes cannot be negative");
      }

      double kib = (double) bytes / KIB;

      if (kib == Math.floor(kib)) {
          return String.format("%.0fKiB", kib);
      }

      return String.format("%.2fKiB", kib);
  }
}
```
