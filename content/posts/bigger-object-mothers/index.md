---
title: "Bigger Mothers - Simplify Your Big (JSON) Data Test Setups"
head_title: Bigger Mothers - Simplify Your Big (JSON) Data Test Setups
description: "Bigger Mothers - Simplify Your Big (JSON) Data Test Setups" 
date: 2026-02-20
draft: true
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
When working with large JSON structures in tests, the signal-to-noise ratio quickly collapses. 

Tests either rely on opaque JSON files or embed massive JSON blobs inline, making it difficult to see what actually matters.

This post shows how to apply the [Object Mother](https://jonasg.io/posts/object-mother/) pattern to JSON test data, allowing tests to <b style="color: #3da6b1;">emphasize relevant parts (fields) while hiding irrelevant structure</b>.

## The Object Mother recap

The Object Mother pattern provides a central place to construct test objects with sensible defaults, while allowing tests to override only the fields relevant to their scenario.

## JSON test data

Imagine a testing scenario where you're dealing with a JSON blob that's quite extensive,
say it consists of 150 fields.

In your test, however, **only a handful of these fields are critical**,
requiring specific values to ensure the test is valid.
However, your code requires the submission of the entire JSON object in order to function.

### Common pitfalls

**1. The File Scavenger Hunt**

Loading JSON from external files leads to a maintenance nightmare.

- **Cognitive Load:** You have to click back and forth between the test and the `data.json` file to understand what is being tested.
- **Hidden Intent:** It’s impossible to tell which of those 150 fields are actually relevant to the current test case.
- **File Explosion:** At worse you end up with `data_v1.json`, `data_final.json`, and `data_modified_for_test_3.json`.

```java
@Autowired
ResourceLoader resourceLoader;

@Test
void testThatRequiresJsonData() {
	var path = Paths.get(resourceLoader.getResource("classpath:test-data/testWithJsonData/data.json").getURI());
	String testData = new String(Files.readAllBytes(path));
```

**2. The Blob of Doom**

Inlining a massive JSON string directly into your test.

- **Noise:** Your test logic is buried under 100+ lines of curly braces and quotes.
- **Fragility:** The slightest change in structure requires a massive find-and-replace across multiple tests.

```java
@Test
void testWithMessyJson() {
    // Is it the 'status' that matters? The 'id'? 
    // You have to read 150 lines to find out.
    String json = """
        {
          "id": "123",
          "status": "ON_GOING",
          ... 148 more lines ...
        }
        """;
}
```



The above constructions negates everything Object Mother brings to the table, even if you abstract the file loading code  away in some kind of reusable component.

## Data Mothers

What if we could use [Object Mothers just like we did for Java Objects](https://jonasg.io/posts/object-mother/) so that the above test would look like this:

```java
String budgetReport = BudgetReportMother.defaultBudget()
	.withProperty("id", "23b48cb7-58f5-4a6c-a551-8b13b286a360")
	.withStatus("status", "ON_GOING")
  // this will completely remove the property from the resulting JSON string
	.withRemovedProperty("mainInvoiceNumber") 
	.build();
```

This makes it immediately obvious which fields matter for the test. Everything else fades into the background, reducing cognitive load and improving readability.

How it's done:

```java
public class BudgetReportMother {
	public static Builder defaultBudget() {
		return new Builder("mother-data/budget-report.json");
	}

  /// Pre-configured scenario for common test cases
  public static Builder ongoingBudget() {
      return budget()
	        .withStatus("status", "ON_GOING")
          .withId("default-id");
  }

	public static class Builder extends AbstractJsonObjectMotherBuilder<Builder> {

		public Builder(String filePath) {
			super(filePath);
		}

		public Builder withStatus(String status) {
			this.withProperty("status", status);
			return this;
		}
	}
}
```

The clue lies in the usage of `AbstractJsonObjectMotherBuilder` which is responsible for loading
and manipulating the JSON data file. 
It is part of the **data-object-mother** utility library I made available at: https://


