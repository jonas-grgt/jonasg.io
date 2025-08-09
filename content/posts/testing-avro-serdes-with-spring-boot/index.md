---
title: "Testing Avro Serialization and Deserialization with Schema Registry in a Spring Boot Application"
head_title: "Testing Avro Serialization and Deserialization with Schema Registry in a Spring Boot Application"
description: "Testing Avro Serialization and Deserialization with Schema Registry in a Spring Boot Application"
date: 2025-08-09
tags:
  - java
  - avro
  - kafka
  - spring-boot
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

No bean overrides just serdes configuration
<!--more-->
---

There are multiple ways to test Avro publication and consumption in a Spring Boot application.

For simplicity, this post will use the terms "serializer" and "serialization," but the same principles apply to 
"deserializers" and "deserialization."

## Testcontainers

An approach is to replicate your production environment using *Testcontainers*.
You can spin up a *Schema Registry* container and connect to it in your tests.

However, this adds a fair amount of complexity and overhead, which can often be avoided while still giving you the
confidence that your Avro serialization and deserialization work as expected, which is ultimately what you want to
test.

## MockSchemaRegistryClient implementation

The `kafka-schema-registry-client` artifact provides us with a `MockSchemaRegistryClient` implementation of the
`SchemaRegistryClient`.
It works as a stub, keeping all schemas in memory.

### How to wire the Mock impl. in your Spring-Boot application?

There are numerous ways to let your _Spring-Boot_ application make use of the `MockSchemaRegistryClient`.

Browsing through the source code you would find the following dependency hierarchy.

```text
[SchemaRegistryClient]
       ▲
       │
[KafkaAvroSerializer]
       ▲
       │
[DefaultKafkaProducerFactory]
       ▲
       │
[KafkaTemplate]
```

The `KafkaTemplate` bean is autoconfigured by *Spring-Boot* and depends upon a `ProducerFactory` which on its turn
requires a `Serializer`.

That `Serializer` in our case is the `KafkaAvroSerializer` which depends upon a `SchemaRegistryClient`.

Given that setup it only feels natural to use **inversion of control** to swap in a `MockSchemaRegistry` for tests and 
use a `CachedSchemaRegistryClient` elsewhere.

This works fine, but there’s a catch; if you create your own `KafkaAvroSerializer`, you’ll have to call its
`configure()`
method yourself, otherwise important serialization properties won’t be set.

And honestly, wiring all of this up by hand feels a bit silly in a  _Spring-Boot_ world where things are supposed to be
auto-configured for you.

### The way it was intended

Actually to wire in a **Mock** implementation you do not need to rely on *Spring-Boot* as it is provided out of the box 
by the schema-registry client code.

All you have to do is configure your `KafkaAvroSerializer` with a `schema.registry.url` property 
that starts with `mock://`.

```yaml
spring.kafka.properties.schema.registry.url: mock://
```

The above will make the `KafkaAvroSerializer` automatically use a `MockSchemaRegistryClient`.

#### How does mock:// work?

As mentioned earlier, when `configure()` is called on the `KafkaAvroSerializer` (which happens automatically by the
`KafkaProducer`), it ends up in `SchemaRegistryClientFactory`.
That factory selects the impl. based upon the given URL as such:

```java
String mockScope = MockSchemaRegistry.validateAndMaybeGetMockScope(baseUrls);
if (mockScope !=null){
    return MockSchemaRegistry.getClientForScope(mockScope, providers);
} else {
    return new CachedSchemaRegistryClient(...);
}
```

So no extra wiring, no manual serializer config, no hassle.

#### Mock scopes

The `mock://` URL can include a scope, which is a name that links a specific `MockSchemaRegistryClient` instance to a
particular URL.

For example, `mock://test` will create a `MockSchemaRegistryClient` for the scope named **"test"**. Any other producer
or
consumer configured with the same **mock://test** URL will share this same client and its in-memory schemas.

Different scopes are useful for:
- simulating multiple Schema Registry instances
- isolating test scenarios
- avoiding conflicts in parallel tests

In most cases you can just use `mock://` without a scope name, as it defaults to using an empty string (**""**) as the
scope. This means all
components configured with `mock://` will also share the same `MockSchemaRegistryClient` instance.

### A word on the placement of the schema.registry.url property

There are basically 4 location where you can put the property and this goes for all serdes specific properties:

#### 1. Global Kafka client properties

Applied to all producers, consumers, and Kafka Streams clients.

```yaml
spring.kafka.properties.schema.registry.url: mock://
```

#### 2. Producer-only properties

```yaml
spring.kafka.producer.properties.schema.registry.url: mock://
```

#### 3. Consumer-only properties

```yaml
spring.kafka.consumer.properties.schema.registry.url: mock://
```

#### 4. Kafka-Stream-specific properties

```yaml
spring.kafka.streams.properties.schema.registry.url: mock://
```



