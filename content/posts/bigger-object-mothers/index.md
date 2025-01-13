---
title: "Bigger Mothers - Simplify Your Big (JSON) Data Test Setups"
head_title: Bigger Mothers - Simplify Your Big (JSON) Data Test Setups
description: "Bigger Mothers - Simplify Your Big (JSON) Data Test Setups" 
date: 2024-03-20
published: false
tags:
  - java
  - testing
  - TDD
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---
Make JSON Test Data Manageable
<!--more-->
---
For those who have been following my blog, you'll recognize my emphasis on the importance of keeping your test setup code DRY (Don't Repeat Yourself) and manageable. This is where the concept of the Object Mother shines.

However, when you start dealing with more complex scenarios involving Java Objects that span dozens of fields and properties, or when your test setups call for data structures like JSON instead of Java Objects, things get trickier.

In this post, we'll explore how to tackle these challenges effectively, using strategies that apply to both Java and JSON testing environments.

## JSON Mothers

Imagine a testing scenario where you're dealing with a JSON blob that's quite extensive –
say, it consists of 150 fields.
In your test, however, only a handful of these fields are critical,
requiring specific values to ensure the test is valid.
However, your test requires the submission of the entire JSON object.

This is an ideal scenario for Object Mothers as they allow you to emphasise on what matter and hide the irrelevant parts.

Object Mothers only give birth to Java Object that is why I have seen solution that try to manage  test data through files
```java
@Autowired
ResourceLoader resourceLoader;

@Test
void testThatRequiresJsonData() {
	var path = Paths.get(resourceLoader.getResource("classpath:test-data/testWithJsonData/data.json").getURI());
	String testData = new String(Files.readAllBytes(path));
```
To above construction negates everything Object Mother bring to the table, even if you abstract it away in some kind of reusable component.
```java
@Component
public class TestDataLoader {

    private final ResourceLoader resourceLoader;

	public TestDataLoader(ResourceLoader resourceLoader) {
		this.resourceLoader = resourceLoader;
	}

    public <T> T loadTestData(String path) throws IOException {
        String json = new String(Files.readAllBytes(Paths.get(resourceLoader.getResource("classpath:" + path).getURI())));
    }
}

@Test
void testWithJsonData() throws Exception {
	String testData = testDataLoader.loadTestData("test-data/testWithJsonData/data.json");
```

## Data Mothers

What if we could use Object Mothers just like we did for Java Objects so that the above test would look like this:

```java
String budgetReport = BudgetReportMother.defaultBudget()
	.withId("reportId123")
	.withBudgetPlaylistId("23b48cb7-58f5-4a6c-a551-8b13b286a360")
	.withStatus(ON_GOING)
	.withoutMainInvoiceNumber()
	.build();
```

This clearly shows that `id`, `budgetPlaylistIds` and `status` are relevant for our test to have the given value. It communicates your intention clearly.

How it's done:

```java
public class BudgetReportMother {
	public static Builder defaultBudget() {
		return new Builder("mother-data/budget-report.json");
	}

	public static class Builder extends AbstractJsonObjectMotherBuilder<Builder> {

		public Builder(String filePath) {
			super(filePath);
		}

		public Builder withId(String id) {
			this.withProperty("id", id);
			return this;
		}

		public Builder withBudgetPlaylistId(String budgetPlaylistId) {
			this.withProperty("budgetPlaylistId", budgetPlaylistId);
			return this;
		}

		public Builder withStatus(ReportProgressStatus status) {
			this.withProperty("status", status.name());
			return this;
		}

		public Builder withoutMainInvoiceNumber() {
			this.withRemovedProperty("mainInvoiceNumber");
			return this;
		}

		@Override
		public String build() {
			return super.build();
		}
	}
}
```

The clue lies in the usage of `AbstractJsonObjectMotherBuilder` which is responsible for loading
and manipulating the JSON data file. 
It is part of a utlity library I made available at:
