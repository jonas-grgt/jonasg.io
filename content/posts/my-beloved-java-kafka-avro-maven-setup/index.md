---
title: "My Beloved Java Kafka Avro Maven Setup"
head_title: "My Beloved Java Kafka Avro Maven Setup"
description: "A way to setup a Java project with Kafka and Avro"
date: 2025-04-05
draft: true
tags:
  - java
  - avro
  - kafka
  - maven
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

Schema First Approach with Avro and Kafka in Java
<!--more-->
---

For the last couple of years I've been working with Kafka and Avro in Java projects.
I've seen quite a few setups and I've come to appreciate a certain one that I'll share
with you.

This follows the schema first approach, where we define our Avro schemas first as opposed
to generating the schemas from Java classes.

## Avro DTO generation

Avro is a data serialization framework that relies on schemas, which are defined in JSON.
An alternative, more readable and closer to how you write your own code,
way to define schemas is to use Avro IDL (Interface Definition Language).

JSON based schemas are stored in .avcs suffixed files, while Avro IDL based schemas are
stored in .avdl suffixed files.

```avroidl

enum Status {
	DRAFT,
	PUBLISHED,
	UNKNOWN
} = UNKNOWN;

record BlogPostCreated {

	string title;

	string content;

	array<string> tags;

	Status status = "DRAFT";

	@logicalType("timestamp-millis")
	long timestamp;
}
```

By utilizing the **avro-maven-plugin** you can generate Java classes from these schemas.
It supports both JSON and Avro IDL based schemas.

```xml

<dependency>
	<groupId>org.apache.avro</groupId>
	<artifactId>avro</artifactId>
	<version>1.10.2</version>
</dependency>
```

This will pick up the Avro IDL files from the `src/main/resources` + `${avdl.dir}`
directory and generate Java classes in the `target/generated-sources` directory.

```xml

<plugin>
	<groupId>org.apache.avro</groupId>
	<artifactId>avro-maven-plugin</artifactId>
	<version>${avro.version}</version>
	<executions>
		<execution>
			<phase>generate-sources</phase>
			<goals>
				<goal>idl-protocol</goal>
			</goals>
			<configuration>
				<sourceDirectory>src/main/resources</sourceDirectory>

				<imports>
					<import>${avdl.dir}/events.avdl</import>
				</imports>

				<outputDirectory>${project.build.directory}/generated-sources
				</outputDirectory>

				<stringType>String</stringType>
				<enableDecimalLogicalType>true</enableDecimalLogicalType>
				<gettersReturnOptional>true</gettersReturnOptional>
				<optionalGettersForNullableFieldsOnly>true
				</optionalGettersForNullableFieldsOnly>
			</configuration>
		</execution>
	</executions>
</plugin>
```

## Useful Avro DTO generation configuration options

| Configuration Option                                                                | Description                                                    | Default Value  |
|-------------------------------------------------------------------------------------|----------------------------------------------------------------|----------------|
| `<stringType>String</stringType>`                                                   | Use `String` instead of `CharSequence` for `string` fields.    | `CharSequence` |
| `<enableDecimalLogicalType>true</enableDecimalLogicalType>`                         | Use `BigDecimal` instead of `ByteBuffer` for `decimal` fields. | `ByteBuffer`   |
| `<gettersReturnOptional>true</gettersReturnOptional>`                               | Use `Optional` for nullable fields.                            | `false`        |
| `<optionalGettersForNullableFieldsOnly>true</optionalGettersForNullableFieldsOnly>` | Use `Optional` getters only for nullable fields.               | `false`        |
| `<createSetters>false</createSetters>`                                              | Used to generate Java classes with setters.                    | `true`         |
| `<fieldVisibility>PUBLIC</fieldVisibility>`                                         | Change field visibility                                        | `PRIVATE`      |

Now we can use the generated Java classes in our code.
Next step is to get the Avro schema's actually registered.

## Avro schema registration

Confluent its `kafka-schema-registry-maven-plugin` is a great way to register Avro schemas
in the Schema Registry.
‼️ The only issue is that it can only work with JSON (.avsc files) based schemas.
So we need to convert our Avro IDL based (avdl) schemas to JSON (avsc) based schemas.

### AVDL to AVSC conversion

The avro library luckily provides a tool to convert Avro IDL based schemas to JSON based
schemas.

### Actual schema registration

```xml
<profiles>
    <profile>
        <id>ST</id>
        <build>
            <plugins>
                <!-- avsc to schema registry-->
                <plugin>
                    <groupId>io.confluent</groupId>
                    <artifactId>kafka-schema-registry-maven-plugin</artifactId>
                    <version>${confluent.version}</version>
                    <configuration>
                        <schemaRegistryUrls>
                            <param>${schema.registry.url}</param>
                        </schemaRegistryUrls>
                        <userInfoConfig>
                            ${schema.registry.api.key}:${schema.registry.api.secret}
                        </userInfoConfig>
                        <subjects>
                            <!-- +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-->
                            <!-- TOPIC: ST.bpost-be.mas.internal.alert.events	                -->
                            <!-- +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-->
                            <ST.bpost-be.mas.internal.alert.eventsmas.alerter.AlertFired>
                                target/generated-sources/avsc/AlertFired.avsc
                            </ST.bpost-be.mas.internal.alert.eventsmas.alerter.AlertFired>
                            
                            <ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.AlertFiringScheduled>
                                target/generated-sources/avsc/AlertFiringScheduled.avsc
                            </ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.AlertFiringScheduled>
                            
                            <ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.AlterFiringCancelled>
                                target/generated-sources/avsc/AlterFiringCancelled.avsc
                            </ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.AlterFiringCancelled>
                            
                            <ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.ScheduledAlertExpectationMet>
                                target/generated-sources/avsc/ScheduledAlertExpectationMet.avsc
                            </ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.ScheduledAlertExpectationMet>
                            
                            <ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.ScheduledAlertExpectationMetLate>
                                target/generated-sources/avsc/ScheduledAlertExpectationMetLate.avsc
                            </ST.bpost-be.mas.internal.alert.events-bpostbe.mas.alerter.ScheduledAlertExpectationMetLate>
                        </subjects>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```