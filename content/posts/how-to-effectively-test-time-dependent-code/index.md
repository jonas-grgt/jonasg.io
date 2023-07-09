---
title: "How to Effectively Test Time-Dependent Code: Unit and Spring-Based Strategies" 
description: Explore strategies for reliable testing of time-dependent code, including techniques to mitigate flakiness in tests and enable precise time manipulation within your test suites.
date: 2023-07-09
tags:
  - java
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

Explore strategies for reliable testing of time-dependent code, including techniques to mitigate flakiness in tests and enable precise time manipulation within your test suites.
<!--more-->

By default, Java relies on the system clock to determine the current date and time. While in most cases this approach works fine when our system clock remains unchanged, it can quickly turn into a nightmare upon a simple faulty configuration change outside of your control.
I have first-hand witnessed the consequences of an innocent OS upgrade silently altering the default time zone of a system. The impact was nothing short of disastrous, as all timestamps generated and stored by the application were suddenly off by a couple of hours.
This post delves into the root cause of the issue, and as we will see, it presents a simple solution that at the same time enables us to write more robust tests and gain better control over the time-related aspects of our Java applications.
## The time is now
The culprit of having our code depend on the system clock is the use of static `java.time...now()` methods such as:
- `LocalDate.now()`
- `LocalDateTime.now()`
- `ZonedDateTime.now()`
- `OffsetDateTime.now()`
- `Instant.now()`
  Examining the implementation of the `ZonedDateTime.now()` method in the JDK, we can see it delegates to another `now` method that takes a `java.time.Clock` instance as a parameter.
```java
public final class ZonedDateTime
        implements Temporal, ChronoZonedDateTime<LocalDate>, Serializable {
        
	public static LocalTime now() {
		return now(Clock.systemDefaultZone());
	}

    public static ZonedDateTime now(Clock clock) {
        Objects.requireNonNull(clock, "clock");
        final Instant now = clock.instant();  // called once
        return ofInstant(now, clock.getZone());
    }
```
Within `java.time.clock` we can find a solution for ensuring our time related code is independent of the system's time zone, while also facilitating easier testing. To further clarify this point, let's refer to an excerpt from its documentation:

>Use of a Clock is optional. All key date-time classes also have a now() factory method that uses the system clock in the default time zone. The primary purpose of this abstraction is to allow alternate clocks to be plugged in as and when required. Applications use an object to obtain the current time rather than a static method. This can simplify testing.
>
>[Java documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/Clock.html)

Think of `java.time.Clock` as a physical wall clock that can be easily replaced with another clock within a different time zone. By passing along a `java.time.Clock` in all the `java.time...now()` methods, we decouple date-time generation from the system's clock  and simultaneously make it easier to test. As also stated in the documentation of the `ZonedDateTime.now(Clock clock)` method:
> Using this method allows the use of an alternate clock for testing. The alternate clock may be introduced using dependency injection.
>
> [Java documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/ZonedDateTime.html#now(java.time.Clock))

### Controlling the correct time zone
Passing a clock bound to a specific time zone frees us from relying solely on the system's clock for date-time generation.
```java
ZonedDateTime.now(Clock.system(ZoneId.of("Europe/Brussels")))
```
By introducing a clock dependency, our production code becomes an area where accessing `java.time.Clock` is necessary. The same requirement extends to our tests, offering us significant benefits. In the next section, we will focus on the tests first and then proceed to examine our production code.
### Fixating the clock
Having fine-grained control over date-time generation in our tests is indispensable for avoiding flaky tests. Java helps by allowing us to *fixate* the clock on a certain date-time within a specific time zone:
```java
Clock.fixed(Instant.parse("1985-02-25T23:00:00.00Z"), 
        ZoneId.of("Europe/Brussels"));
```
Let's take the following piece of production code we want to test:
```java
public class Order {

	private Status status;
	private LocalDateTime processDateTime; 

	public Order markAsProcessed() {
		this.processDateTime = LocalDateTime.now();
		this.status = PROCESSED;
	}
}
```
By adding a `java.time.Clock` as a dependency this becomes easily and accurately testsable:
```java
public Order markAsProcessed(Clock clock) {
	this.processDateTime = LocalDateTime.now(clock);
```
Which would result in the following test:
```java
public class OrderTest {

	@Test
	void 
		// given
		var order = OrderMother.newOrder(); 

		// when
        var clock = Clock.fixed(Instant.parse("1985-02-25T23:00:00.00Z"), 
                ZoneId.of("Europe/Brussels");
		order.markAsProcessed(clock);

		// then
		assertThat(order.processDateTime())
          .isEqualTo(LocalDateTime.parse("1985-02-26T00:00:00"));
	}
}
```
Notice how in the assertion the day has moved on by one hour and one day, because the date-time is generated in a +1 time zone.
### Alternative solutions
#### Truncating time for more precision
There are alternative solutions available for testing date-time generation if you *prefer not to rely* on `java.time.Clock`. However, it's important to note that relying solely on the system's default clock carries its own risks.

The problem with asserting the generated date-time is that even a small amount of time between generating the date-time and asserting it can lead to faulty assertions. Consequently, this test example without the clock will likely be unreliable:
```java
public class OrderTest {

	@Test
	void 
		// given
		var order = OrderMother.newOrder(); 

		// when
		order.markAsProcessed();

		// then
		assertThat(order.processDateTime())
			.isEqualTo(LocalDateTime.now());
	}
}
```
This test is prone to flakiness and will probably fail consistently due to the time that elapses between the `markAsProcessed` method's `LocalDateTime.now()` call and the actual assertion. To mitigate this, you can truncate the generated date-time to seconds or milliseconds and perform the same truncation in the assertion. While this approach reduces flakiness, it doesn't guarantee 100% accuracy. But in most cases that trade-off is acceptable.
Here's an example of truncating to seconds in your production code:
```java
Instant.now().truncatedTo(ChronoUnit.SECONDS)
```
And this can be tested like so:
```java
public class OrderTest {

	@Test
	void 
		// given
		var order = OrderMother.newOrder(); 

		// when
		order.markAsProcessed();

		// then
		assertThat(order.processDateTime())
			.isEqualTo(LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS));
	}
}

```
Wondering what this `OrderMother` is, check out my previous article on [mothers](https://jonasg.io/posts/object-mother)
#### Assert generated time is close enough to now
A second alternative is to utilize [AssertJ](https://joel-costigliola.github.io/assertj)'s `.closeTo` assertion methods, which provides a convenient way to assert values within a specified range. Here are a couple of examples to illustrate this:
```java
var localDateTime = LocalDateTime.now(Clock.systemUTC()); 
assertThat(localDateTime)
	.isCloseToUtcNow(within(1, ChronoUnit.SECONDS));

var instant = Instant.parse("2000-01-01T00:00:00.00Z")
assertThat(instant)
	.isCloseTo("1999-12-31T23:59:59.99Z", within(10, ChronoUnit.MILLIS))
```
### Clock as a spring bean
Going forward with the clock. Whenever we require access to the current date-time, it is necessary to have access to the clock. However, passing the clock object throughout our code can become cumbersome. 
Fortunately, most modern applications make use of an IoC (Inversion of Control) Container, which alleviates this burden. As a result, we will expose the Clock as an object eligible for inversion of control or, in Spring terminology, convert it into a bean.
```java
@Configuration
class ClockConfiguration {

	@Bean
	Clock clock(@Value("${app.time.zone-id}") String zoneId){
		return Clock.system(ZoneId.of(zoneId));
	}
}
```
Depending on our needs, we have the option to either hard-code the active timezone or, as demonstrated in the example above, make it configurable. In either case, this approach allows us to conveniently inject the clock whenever necessary, as illustrated in the example below.
```java
@Service
public class OrderService {

	private final Clock clock;
	private final OrderRepository orderRepository;
	
	public OrderService(OrderRepository orderRepository, Clock clock) {
		this.clock = clock;
		this.orderRepository = orderRepository;
	}

	public void processOrder(OrderId orderId) {
		var order = orderRepository.findById(orderId);
		order.markAsProcessed(clock);
		// etc ..
	}
}
```
Particularly within testing scenarios, having a clock eligible for inversion of control becomes critical. As it empowers us to precisely control and manipulate time-related behavior in our tests.
Just like in our unit tests, let's start by exposing a fixated clock.
```java
@Configuration
public class TestClockConfiguration {

	@Bean
	@Primary
	Clock fixedClock() {
		return Clock.fixed(Instant.parse("1985-02-25T23:00:00.00Z"), 
                ZoneId.of("Europe/Brussels");
	}

}
```
To ensure consistent time behavior across all tests within a Spring context, we can include the above configuration in our test package. This will, due to the use of `@Primary` override the clock defined in our production code with a fixed clock.
Now we can assert the order to be marked as processed at the fixed date-time.

What if we require precise control over the clock at a per-test level within a single Spring context, without the need to create a new context for each case where a different clock is desired?
Or maybe we want to play with our current date-time and actually move time forward or maybe even rewind it? 
The next section will cover these use-cases.
### Mutable clock
The default `java.time.Clock` implementation is immutable in the sense that you can not change it's current date-time or timezone.
By incorporating a mutable clock that can manipulate time or be set to a specific date-time within our tests, we can avoid the need for multiple Spring contexts.
```java
public class MutableTestClock extends Clock {

	private Instant instant;

	private final ZoneId zone;

	public MutableClock(Instant instant, ZoneId zone) {
		this.instant = instant;
		this.zone = zone;
	}

	@Override
	public ZoneId getZone() {
		return zone;
	}

	@Override
	public Clock withZone(ZoneId zone) {
		return new MutableTestClock(instant, zone);
	}

	@Override
	public Instant instant() {
		return instant;
	}

	public void fastForward(TemporalAmount temporalAmount) {
		set(instant().plus(temporalAmount));
	}

	public void rewind(TemporalAmount temporalAmount) {
		set(instant().minus(temporalAmount));
	}

	public void set(Instant instant) {
		this.instant = instant;
	}

	public static MutableClock fixed(Instant instant, ZoneId zone) {
		return new MutableTestClock(instant, zone);
	}

	public static MutableClock fixed(OffsetDateTime offsetDateTime) {
		return fixed(offsetDateTime.toInstant(), offsetDateTime.getOffset());
	}
}
```
Exposing the mutable clock:
```java
@Configuration
public class TestClockConfiguration {

	@Bean
	@Primary
	Clock testClock(@Value("${app.time.zone-id}") String zoneId) {
		return new MutableClock(Instant.parse("1985-02-25T23:00:00.00Z"), 
                ZoneId.of(zoneId));
	}

}
```
Now, let's put this knowledge to good use by writing a test for the following piece of production code:
```java
@Component
public class OrderProcessor {

	private final Clock clock;

	private final LocalTime startOfWorkingDay = LocalTime.of(8, 0); 

	private final LocalTime endOfWorkingDay = LocalTime.of(22, 0); 

	public OrderProcessor(Clock clock) {
		this.clock = clock;
	}

	public void processOrder(Order order) {
		if (isWithinWorkingHours()) {
			processNow(order);
		} else {
			processLater(order);
		}
	}

	public boolean isWithinWorkingHours() {
	        LocalTime now = LocalTime.now(clock)
	        return !now.isBefore(startOfWorkingDay) 
                   && !time.isAfter(endOfWorkingDay);
	}
}
```
```java
public class OrderProcessingFeatureTest {

	private final MutableClock mutableClock;

	public OrderProcessingFeatureTest(Clock clock) {
		this.mutableClock = (MutableClock)clock;
	}

    @Test
	void shouldProcessOrderWithinWorkingHours() {
		// given
		mutableClock.set(Instant.parse("2023-02-25T13:00:00.00Z"))

		// when
        var resultActions = mockMvc.perform(post("/orders/ORD567890")
            .contentType(MediaType.APPLICATION_JSON)
            .content(toJson(OrderMother.defaultOrder()));

		// then
        resultActions.andExpect(status().isOk())
	        .andExpect(jsonPath("$.status").value("PROCESSED"));
	}
    
    @Test
	void shouldProcessOrderLaterWhenReceivedOutsideOfWorkingHours() {
		// given
		mutableClock.set(Instant.parse("2023-02-25T23:00:00.00Z"))

		// when
        var resultActions = mockMvc.perform(post("/orders/ORD4785669")
            .contentType(MediaType.APPLICATION_JSON)
            .content(toJson(OrderMother.defaultOrder()));

		// then
        resultActions.andExpect(status().isOk())
	        .andExpect(jsonPath("$.status").value("PROCESS_LATER"));
	}
}

```
In conclusion, leveraging the `java.time.Clock` class and its capability to decouple date-time generation from the system clock empowers us to write more effective tests for time-dependent code.
This approach not only grants us greater control over time-related aspects but also acts as a safeguard against issues arising from system clock changes, as I have personally encountered.
