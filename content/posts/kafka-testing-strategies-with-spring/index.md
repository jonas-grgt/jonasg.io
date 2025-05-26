---
title: "Kafka Testing Strategies with Spring"
head_title: "Kafka Testing Strategies with Spring"
description: "Kafka Testing Strategies with Spring"
date: 2025-05-25
tags:
  - java
  - spring-boot
  - spring-kafka
  - kafka
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

How to test a spring based application that uses Kafka?
<!--more-->
---

The goal of this post is to share my experience with testing a spring based application
that uses Kafka.
My aim has always been to have a fast feedback loop when testing my Kafka-based
applications.

I will cover verifying that messages are published to Kafka, consuming them in tests.

A fully working example project accompanying this post can be found at:

{{<icon github>}}&nbsp;[https://github.com/jonas-grgt/spring-kafka-testing-demo](https://github.com/jonas-grgt/spring-kafka-testing-demo)

## The system under test

In this example, I will test a simple application that publishes a `FraudSuspected` event.

```shell
@PostMapping("/creditcard/transactions")
void process(@RequestBody CreditCardTransaction transaction) {

    if (transaction.amount() != null &&
        transaction.amount().compareTo(new BigDecimal("10000")) >= 0) {

        String fraudId = idGenerator.generateId();
        kafkaTemplate.send(fraudAlertTopic, fraudId, FraudSuspected.newBuilder()
                .setFraudId(fraudId)
                .setSuspicionReason(SuspicionReason.UNUSUAL_AMOUNT)
                .setAmount(transaction.amount())
                .build());

    }
}
```

## Abstract away your topic configuration

In one of my previous posts I emphasized having
a [topic naming convention](https://jonasg.io/posts/kafka-without-structure-is-just-kafka/#topic-naming-convention).
These fully qualified topic names can get long, convoluted and obnoxious to manage in your
tests. In the beginning they can also be subjected to change and in that case I prefer to
only have to define or change the topic in one place.

For example, when working with the `dev.transactions.creditcard.fraud-alert.events` topic,
I never refer to it by its full name—I just call it the "fraud alert topic."

So, I use configuration shorthand in my configuration:

```yaml
my-app:
  topics:
    fraud-alert: ${env}.transactions.creditcard.fraud-alert.events
```

I aim for having one **root** `application.yaml` and replace everything that differs per
environment, as I did with the `env` variable above.

This way I can use the `@Value` annotation to inject the topic —by reference— in my tests.

```java

@Value("${my-app.topics.fraud-alert}")
String fraudAlertTopic;
```

## The startup costs

With the newer Kafka Native images, the container startup time has dropped significantly.
That said, the Spring context initialization still tends to dominate the total test boot
time. So optimizing how often that context is created becomes critical.

In my case for all integration tests I prefer to have **one context to rule them all**.

## Sharing Testcontainers and Spring Contexts

If you let your containers lifecycle be managed by **Testcontainers**, using the standard
`@Testcontainers` and `@Container` annotations, you will end up with a new Kafka container
for each test class.

This forces you to create a new Spring context for each test class as well since the
`ConnectionDetails` will differ between tests.

To keep my test suite fast and lean, I optimize for:

- a single Spring context shared across all tests
- a single Kafka container reused by all test classes

Let me show you how I set that up.

### Singleton containers managed by spring for the win

When you expose a `Container` as a Spring Bean, it will be started and stopped
automatically by Spring and be active for the lifetime of the application context—
as long as all the tests share the same Spring context.

```java

@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    KafkaContainer kafkaContainer() {
        return new KafkaContainer("apache/kafka-native:3.8.0")
                .withReuse(true)
                .withExposedPorts(9092, 9093);
    }

}

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class MyKafkaTest {
    // ...
}
```

Notice the `@ServiceConnection` annotation, it binds the kafka connection details to
spring-boot's configuration properties. In other words it will automatically set the
`spring.kafka.bootstrap-servers` etc...



## Verify records are published

### Consumer setup

`spring-kafka` comes with a handy utility class `KafkaTestUtils` that allows us to consume
messages from a topic in a blocking way.
It either waits for an expected number of messages to be received or for a specific
timeout to be reached —default 60 seconds which is a bit too high for me.

It does require a `KafkaConsumer` to be passed in. Luckily, we can leverage the
`ConsumerFactory` that is already autoconfigured by spring-boot.

```java

@Autowired
ConsumerFactory<String, Object> consumerFactory;

@Test
void myTest() {
    this.consumer = consumerFactory.createConsumer("test-group", "-test-client");
    this.consumer.subscribe(List.of(fraudAlertTopic));
    ConsumerRecords<String, Object> records = KafkaTestUtils
            .getRecords(this.consumer, Duration.ofSeconds(5));
}
```

### Close the group before leaving the test

Make sure to close the consumer at the end of your test. If you don't, the group
remains active, and the next test run using the same group ID will be slowed down due to a
rejoin process.

During this delay, you’ll see a log message similar to the following:

```shell
[Consumer clientId=test.bank-test-client, groupId=test-group] (Re-)joining group
```

⚠️
Avoid closing the consumer group inside the test method, as it won’t run if the test fails
or is interrupted.
Instead, use an `@AfterEach` method to ensure the consumer group is reliably closed after
each test.
For that you will need to assign it to an instance variable.

```java

@AfterEach
void tearDown() {
    if (this.consumer != null) {
        this.consumer.close();
    }
}
```

### Randomize your group ID

Alternatively, you can use a random group ID for each test to avoid the rejoin process.
This can be useful when you want to run tests in parallel or when your tests errors before
reaching the `@AfterEach` method. Even with a random group ID, I would still opt to always
try to close the group.

## Async timing issues

Tests like these can be flaky if you're not careful.

### Async in nature

The actual send to Kafka happens **asynchronously**, and may be batched depending on the
producer configuration.
The `ListenableFuture` returned by `KafkaTemplate` also reveals this async behavior.

If you want to force messages to be sent immediately, in a blocking manner, you can call:

```java
kafkaTemplate.flush();
```

In any case, it’s usually better to embrace the asynchronous nature and use **Awaitility** to
handle timing gracefully:

```java
Awaitility.await()
        .atMost(Duration.ofSeconds(5))
        .untilAsserted(() -> {
            ConsumerRecords<String, Object> records = KafkaTestUtils
                    .getRecords(this.consumer, Duration.ofMillis(200));
        });
```
Note that the `Duration` passed to `getRecords` is shorter than the Awaitility timeout.
This way, if the records aren't available yet, Awaitility will simply retry on the next poll.

### Consumer Group Registration

Kafka initial registration of a consumer group is when `Consumer.poll()` is first called —
in our case from the `KafkaTestUtils.getRecords()` method.
If a message is published before that call, and `auto.offset.reset` is set to `latest` 
(the default), the consumer will miss the record entirely.

This means that your first test will fail and subsequent tests might succeed depending on
how they assert things.

To avoid this issue, I set the `auto.offset.reset` property to `earliest` in my
**test** `application.yml`.

⚠️ Be careful when setting this property in your main or production
`application.yml` as it will affect your production consumers.

```yaml
spring:
  kafka:
    consumer:
      properties:
        auto-offset-reset: earliest
```

An alternative is to create a custom `ConsumerFactory` for tests only and set the
`auto.offset.reset` property there.

### Lenient assertions

Because I use the **singleton Testcontainers** approach, topics may contain leftover data
from previous tests.

To avoid flaky tests, I rely on **random ids** and **lenient assertions**, meaning don't
blindly assert the exact number of messages received,
but rather rely on specific ids or shape of the data to ensure you are asserting the
records from your test.

### Reusable TestContainers

An added benefit of this approach is that I can reuse the container across test runs.

The container starts up once, and on subsequent runs it is reused with its existing state.
In my case, this isn’t a problem—on the contrary, it helps speed things up.

To enable container reuse, I set the following in my `~/.testcontainers.properties` file:

```properties
testcontainers.reuse.enable=true
```

## Final Test Example

```java

@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfiguration.class)
class FraudDetectionTests {

    @Autowired
    ConsumerFactory<String, Object> consumerFactory;

    @Autowired
    MockMvc mockMvc;

    @Autowired
    StubIdGenerator stubIdGenerator;

    @Value("${bank.topics.fraud-alert}")
    String fraudAlertTopic;

    private Consumer<String, Object> consumer;

    @AfterEach
    void tearDown() {
        if (this.consumer != null) {
            this.consumer.close();
        }
    }

    @Test
    void fraudSuspectedWhenAmountIsUnusual() throws Exception {
        // given
        String fraudId = UUID.randomUUID().toString();
        stubIdGenerator.setNextId(fraudId);

        // when
        mockMvc.perform(post("/creditcard/transactions")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "amount": "10000.001"
                                }
                                """))
                .andExpect(status().isOk());

        // then
        this.consumer = consumerFactory.createConsumer("test-group", "-test-client");
        this.consumer.subscribe(List.of(fraudAlertTopic));

        Awaitility.await()
                .atMost(Duration.ofSeconds(5))
                .untilAsserted(() -> {
                    ConsumerRecords<String, Object> records = KafkaTestUtils
                            .getRecords(this.consumer, Duration.of(200));

                    assertThat(records)
                            .satisfiesOnlyOnce(record -> {
                                assertThat(record.key()).isEqualTo(fraudId);
                                assertThat(record.value())
                                        .isInstanceOfSatisfying(FraudSuspected.class, c -> {
                                            assertThat(c.getFraudId()).isEqualTo(fraudId);
                                            assertThat(c.getAmount()).isPresent()
                                                    .get()
                                                    .usingComparator(BigDecimal::compareTo)
                                                    .isEqualTo(new BigDecimal("10000.001"));
                                            assertThat(c.getSuspicionReason()).isEqualTo(UNUSUAL_AMOUNT);
                                        });
                            });
                });
    }
}
```

## Boilerplate

I must admit that there is still some **boilerplate** code in the tests, something that
I'll abstract away in a follow-up post.