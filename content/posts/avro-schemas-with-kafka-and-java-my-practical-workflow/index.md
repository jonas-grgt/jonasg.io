---
title: "Avro Schemas Generation and Registration with Kafka and Java: My Practical Workflow"
head_title: "Avro Schemas Generation and Registration with Kafka and Java: My Practical Workflow"
description: "Defining, Registering Schemas, and Generating Java Classes"
date: 2025-05-16
tags:
  - java
  - avro
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

Defining, registering, and generating Avro-based schemas with Java and Maven.
<!--more-->
---

Over the past couple of years, I've been using [Apache Avro](https://avro.apache.org/) as
a data
format to publish data on Kafka.
I've seen quite a few setups and have come to appreciate one in particular, which I'll
share with you.

This post comes with a complete example project on GitHub to help you follow along.

{{< icon github >}}&nbsp;[https://github.com/jonas-grgt/avro-demo](https://github.com/jonas-grgt/avro-demo)

## Project structure

The project is structured into three Maven modules to promote modularity and reuse. Each
can be released as a separate artifact and used independently by consumers — or by the
producing application itself.

- `app` containing the application code
- `events` containing the schemas and the Java DTOs based on those schemas
- `event-mothers` containing my beloved [Object Mothers](https://jonasg.io/posts/object-mother/) objects for the events

The **app** module naturally depends on **events** in order to publish events.

Both **events** and **event-mothers** are released as separate artifacts.

This allows consumers to depend on the **events** module to reuse the generated DTOs
without
needing to know much about the underlying schema, since it's already embedded in the
generated classes (as an inner class `$Schema`). They can also optionally include the
**event-mothers** module for
convenient test data generation.

## Avro Schema Definition

Avro is a data serialization framework that relies on schemas, which are defined in JSON
and registered as such in the schema-registry.
An alternative, more readable approach that aligns better with how you write regular
code,
is to use [Avro IDL](https://avro.apache.org/docs/1.12.0/idl-language/)(Interface
Definition Language).

I prefer the IDL format because it provides a more natural way to represent data
structures.
Just compare the following identical schemas: the first one in JSON and the latter in IDL.

```avroschema
{
  "type": "record",
  "name": "BlogPostCreated",
  "fields": [
    {
      "name": "title",
      "type": "string"
    },
    {
      "name": "content",
      "type": "string"
    },
    {
      "name": "tags",
      "type": {
        "type": "array",
        "items": "string"
      }
    },
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "Status",
        "symbols": [
          "DRAFT",
          "PUBLISHED",
          "UNKNOWN"
        ],
        "default": "UNKNOWN"
      },
      "default": "DRAFT"
    },
    {
      "name": "timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      }
    },
    {
      "name": "properties",
      "type": {
        "type": "record",
        "name": "PostProperties",
        "fields": [
          {
            "name": "author",
            "type": "string"
          },
          {
            "name": "category",
            "type": "string"
          },
          {
            "name": "language",
            "type": "string"
          }
        ]
      }
    }
  ]
}
```

```avroidl
enum Status {
	DRAFT,
	PUBLISHED,
	UNKNOWN
} = UNKNOWN;

record PostProperties {
	string author;
	string category;
	string language;
}

record BlogPostCreated {
	string title;
	string content;
	array<string> tags;
	Status status = "DRAFT";
	@logicalType("timestamp-millis")
	long timestamp;
	PostProperties properties;
}
```

## Java Class Generation

Avro provides a Maven plugin to generate Java classes from Avro JSON schemas and Avro IDL
files.

The following snippet configures the Maven plugin to generate Java classes from Avro IDL
files (.avsc).
It generates them into the `target/generated-sources` directory based on the `events.avdl`
file located in `src/main/resources`.

```xml

<plugin>
	<groupId>org.apache.avro</groupId>
	<artifactId>avro-maven-plugin</artifactId>
	<version>${avro-maven-plugin.version}</version>
	<executions>
		<execution>
			<phase>generate-sources</phase>
			<goals>
				<goal>idl-protocol</goal>
			</goals>
			<configuration>
				<!-- by default all *.avdl files are picked up from the src/main/resources directory -->
				<sourceDirectory>src/main/resources</sourceDirectory>
				<outputDirectory>${project.build.directory}/generated-sources
				</outputDirectory>

				<!-- use java.lang.String instead of org.apache.avro.util.Utf8 -->
				<stringType>String</stringType>
				<!-- use java.math.BigDecimal for bytes with decimal logical type -->
				<enableDecimalLogicalType>true</enableDecimalLogicalType>
				<!-- generate getters that return java.util.Optional -->
				<gettersReturnOptional>true</gettersReturnOptional>
				<!-- only use Optional for fields that are nullable in the schema -->
				<optionalGettersForNullableFieldsOnly>true
				</optionalGettersForNullableFieldsOnly>
			</configuration>
		</execution>
	</executions>
</plugin>
```

### The Schema Registry does not support Avro IDL directly — it only accepts Avro JSON (AVSC) schemas.

As discussed in
a [previous post](https://jonasg.io/posts/kafka-without-structure-is-just-kafka/),
before taking any technical steps to register schemas, you need to decide whether to
manage them in a centralized or distributed manner.

Since schemas are part of the source code, I prefer to manage them in a distributed way
using the
[kafka-schema-registry-maven-plugin](https://docs.confluent.io/platform/current/schema-registry/develop/maven-plugin.html).

However, there's a problem: the kafka-schema-registry-maven-plugin only works with JSON
schemas — not IDL files.

🤔 So how do we get the Avro IDL files registered in the schema-registry?

#### maven-avdl-to-avsc-plugin

[The maven-avdl-to-avsc-plugin](https://github.com/jonas-grgt/avdl-to-avsc-maven-plugin)
converts Avro IDL files to Avro JSON (.avsc) files, which can then be registered with the
schema registry.

You just need to point it to your Avro IDL files and specify the output directory.

```xml

<plugin>
	<groupId>io.jonasg</groupId>
	<artifactId>avdl-to-avsc-maven-plugin</artifactId>
	<version>${avdl-to-avsc-maven-plugin.version}</version>
	<executions>
		<execution>
			<goals>
				<goal>avdl-to-avsc</goal>
			</goals>
			<configuration>
				<avdlDirectory>${avdl.dir}</avdlDirectory>
				<avscDirectory>${project.build.directory}/generated-sources/avsc
				</avscDirectory>
			</configuration>
		</execution>
	</executions>
</plugin>
```

### Registering the Avro Schemas

Now that the Avro IDL files have been converted to JSON, we can register them with the
schema registry:

```xml

<profile>
	<id>dev</id>
	<build>
		<plugins>
			<plugin>
				<groupId>io.confluent</groupId>
				<artifactId>kafka-schema-registry-maven-plugin</artifactId>
				<version>${kafka-schema-registry-maven-plugin.version}</version>
				<configuration>
					<schemaRegistryUrls>
						<param>${schema.registry.url}</param>
					</schemaRegistryUrls>
					<userInfoConfig>
						${schema.registry.api.key}:${schema.registry.api.secret}
					</userInfoConfig>
					<subjects>
						<dev.orders>
							target/generated-sources/avsc/Order.avsc
						</dev.orders>
					</subjects>

					<compatibilityLevels>
						<dev.orders>
							FORWARD_TRANSITIVE
						</dev.orders>
					</compatibilityLevels>
				</configuration>
			</plugin>
```

Note that there is no one-to-one mapping between the Avro IDL files and the generated
JSON.
Behind the scenes every avro `record` and `enum` is converted to its own avsc JSON file.

This means when you have one big AVDL file, it will be split into multiple smaller AVSC
files, and you will need to refer to those avsc files when registering schemas.

To register schemas and set their compatibility level effectively (ideally in a CI
pipeline), you can use:

```bash
mvn package schema-registry:register -pl events -P dev -ntp
mvn package schema-registry:set-compatibility -pl events -P dev -ntp
```

#### A little caveat

The `maven-avro-plugin` will generate a JSON schema (inside the generated Java class) that
, during serialization, will be compared to the already registered schema.

The caveat is that, if there are differences between the schemas, serialization will fail
with an exception.

Because we configured the `maven-avro-plugin` to use `java.lang.String` instead of
`org.apache.avro.util.Utf8` as the string type. This information is captured within the
Schema inside the generated class as such:

```json
{
  "type": {
    "type": "string",
    "avro.java.string": "String"
  }
}
```

Yet this information is not inside the JSON schema generated by the
`avdl-to-avsc-maven-plugin` because it just translates AVDL to AVSC. 

For that reason the serialization will fail, and you will be presented by a very confusing 
**schema not found exception** when using the `kafka-avro-serializer` to serialize the object.

If we could remove this extra metadata from the Java-generated schema, the two schemas
would become identical. While it's not possible during class generation, it is possible
during serialization — by setting the following property:

```yml
spring:
  kafka:
    properties:
      avro.remove.java.properties: true
```

This instructs the serializer to strip Java-specific metadata (like avro.java.string) from
the schema before comparing it to the one registered in the Schema Registry.