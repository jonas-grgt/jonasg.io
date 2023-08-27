---
title: "The subtle Art of Java Test Method Naming" 
description: "Exploring the Depth of Test Method Naming: Beyond Conventions to Enhance Clarity."
date: 2023-08-27
tags:
  - java
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

Tests must be encapsulated within methods that demand a name, and as with all aspects of programming, naming often proves to be the most challenging task.
<!--more-->
---
In navigating the software landscape, I've encountered varied approaches to naming test methods. Some teams grant developers the freedom to name tests as they see fit. However, this often leads to a mix of clear, cryptic, misleading, or ambiguous test method names.

Conversely, certain teams enforce test method naming conventions to address these issues. Nevertheless, conventions can feel rigid, clumsy, and unfitting, sometimes resulting in problems similar to having no convention at all.

This article embarks on a journey to devise a practical strategy for defining test method names and their structure, all rooted in their inherent value. So, what truly defines the value of a test method name?
## The documentation value
Tests offer an invaluable yet often underestimated asset: their inherent documentation capabilities. As you write tests, you define the boundaries of your business logic or explore the capabilities of your infrastructural code, often without recognizing that you're undertaking an additional vital task — **documenting the capabilities of your system**.

The documentation value tests have goes beyond the static nature of traditional documentation found in sources like company wikis. **Tests are living pieces of documentation**, continuously evolving due to their executable nature. No feature or requirements should be left untested, hence it is documented. And when a test fails, often signalling regression or a change in the required behavior, you are prompted to correct it, ensuring both the accuracy of your system and the correctness of your documentation.
## The Anatomy of a Superb Name
Conventional wisdom often stipulates that test method names should be concise and comprehensible, yet these directives occasionally lack the depth required to holistically define a well-documented test method name. Here are a few recommendations that I endorse:
### Transcending Standard Java Conventions
Although underscores in method names seldom grace production Java code, I propose that they're an exception in the realm of test code. Test code operates under different rules and priorities, with clarity taking the utmost importance. Since test method names can occasionally become lengthy and words might feel crammed together, incorporating the option to use underscores provides a breathing space that contributes to more readable and comprehensible test names. Additionally, an underscore serves as a perfect separator, creating distinct and clear sections within the method name.

This also applies to moving beyond conventional Java method casing rules. Later you'll find some unconventional usages of casing for the sake of readability.
### Ubiquitous language
The ubiquitous language, gathered from business lingo and seamlessly integrated into your production code, should seamlessly extend to your test code, including method names.
### Move away from describing: What is being tested
If the essence of a test method name resides in its documentation value, then it's logical to encapsulate the tested behavior within that name. Yet, countless tests merely encapsulate **what's being tested, neglecting to describe the behavior**. This seemingly subtle distinction wields remarkable power.

> 💡Ask yourself, would the name be clear for a non-technical individual?

👎 Examples of subpar test names that **solely** detail **what's being tested**:
```java
void expireInvoiceTest() {
void shouldStartReportJob() {
void addPineappleToppingToPizzaTest() {
void testCalculateIllegalVATRate() {
```

Glossing over these test names leaves you guessing about the governing rules and expected behavior.
- What triggers an invoice's expiration and what follows?
- What initiates the job and its subsequent outcomes?
- Is topping a pizza with pineapples allowed?
- What makes a VAT rate illegal?
### Avoid giving Examples
When writing a test, even when we are considering the behavior, we often transition from a general definition to a more concrete version. This is perfectly normal, as the test will implement the behavior using actual data or values. However, these example values should **not imply** their presence in our test method names

> 💡Exclude exemplar data from test method names

👎 Examples of subpar test names that give examples of describing behavior:
```java
void testExpireInvoiceOnAugust15th() {
void testReportJobStartsAt8AM() {
void testCalculateMinus10PercentVatRate() {
```
- A specific date limits the understanding of the tested behavior to this exact date, lacking generality.
- Focuses on a particular time, limiting the overal comprehension of when the job should start.
- Mentions a precise VAT rate, which might not be clear in conveying the overall behavior.
## The issue with test naming conventions
The value in test method naming conventions often only remains confined to providing structural guidance. These conventions are indeed useful tools, ensuring codebase coherence and aiding new developers in quickly adapting to a consistent naming strategy.

>💡️Without a primary focus on describing the behavior, conventions can sometimes fall short.

Most conventions allow you to document the behavioral aspects of your codebase, irrespective of the structure they impose. As we've explored, a truly effective test method name should encapsulate the behavior, requirement, or feature being tested, though achieving this is often easier said than done.
## Deconstructing Common Conventions
For the remainder of this article, we'll explore commonly found conventions. All examples will be written based on the following fictional requirement:

> 📝 An outstanding invoice should incur an automatic 10% fee 30 days after its publication date.
### Convention: must start with `should`
One commonly encountered convention insists on using a `should` prefix. While this might naturally spotlight the outcome or the aspect under test, it somewhat sidelines the behavior.

👎 Lacks complete behavior description:
```java
void shouldAutomaticallyIncur10PercentFee() {
```
---
👎 Drifts from the ubiquitous language by using:
- *add* instead of *incur*
- *costs* instead of *fee*
- *unpaid* instead of *outstanding*
- *creation* instead of *publication*
```java
void shouldAutomaticallAdd10PercentExtraCostsToAnUnpaidInvoice30DaysAfterCreationDate() {
```
---
👍 Offers a more behavior-centric approach:
```java
void shouldAutomaticallIncur10PercentFeeToAnOutstandingInvoice30DaysAfterPublicationDate() {
```
---
👍 Emphasizes clarity by sectioning with underscores:
```java
void shouldAutomaticallIncur10PercentFee_ToAnOutstandingInvoice_30DaysAfterPublicationDate() {
```
### Convention: `Given_state_When_action_Then_outcome`
The Given When Then paradigm offers a clear and structured way to capture all aspects of a well-defined behavior. It is well-recognized and used across various domains, making it familiar to developers and non-technical stakeholders alike.
However, it's important to note that this convention can become verbose due to the repeated use of Given, When, Then. This structure might lead to lengthy test method names, which could potentially impact readability and maintainability.

👍
```java
void Given_OustandingInvoice_When_30DaysAfterPublicationDate_Then_ShouldIncur10PercentFee() {
```
### Convention: `Givenstate_Whenaction_Thenoutcome`
Applying the same convention with a touch of brevity. However, the decision to include underscores ultimately hinges on individual taste and preference. 

👍
```java
void GivenOustandingInvoice_When30DaysAfterPublicationDate_ThenShouldIncur10PercentFee() {
```
## Structural layout

Frequently when testing a requirement, you will need several tests to fully cover it. In this process, these tests can share similar components or structures, resulting in unnecessary redundancy in the test method name.
To mitigate the lengthening of test method names, an interesting approach is to abstract these repetitions into distinct code segments. With the advent of JUnit 5, achieving this is feasible through the use of the `@Nested` annotation, allowing you to segregate the repetitions into separate classes.

```java
class InvoiceTest {
    @Nested
    class GivenAnOustandingInvoice {

        @Test
        void whenPaymentExceeds30DaysLimit_Then_Incur10PercentFee() {

        @Test
        void whenWithin30DaysLimit_Then_NoFreeIsIncurred() {
    }
```

```java
class OustandingInvoiceFeatureTest {
	@Nested
	class ShouldIncur10PercentFee {
	
		@Test
		void whenPaymentExceeds30DaysLimit() {
	
		@Test
		void whenWithin30DaysLimit_Then_NoFreeIsIncurred() {
```

However, this approach challenges common conventions, as most conventions primarily focus on method names.

----
Through my own experiences, I've explored various strategies and reached a definitive realization: while conventions offer structure, they might not inherently capture the intended focus on the behavior under test. It's also worth considering expanding conventions beyond solely method naming and allowing them to be employed within nested classes, providing a holistic approach to enhancing test clarity.

No convention will be perfect, but it's important to select one from the beginning and stick with it.

>💡When selecting a convention, **it is essential to recognize that the ultimate objective of a test name is to succinctly encapsulate the intended behavior.**

Upon completing a test, I've found it beneficial to revisit the test method itself and contemplate the following question:

> 💡 If I were to revisit this test within a year or if a new colleague were to read it, would the name effectively convey the behavior without needing a deep dive into the implementation details?
