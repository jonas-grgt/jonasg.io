---
published: false
title: "Mastering the Object Mother"
date: 2020-08-07
tags:
  - java
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

Discover how the Object Mother concept empowers developers to effortlessly generate intricate test objects, enhancing code readability, maintainability, and overall testing efficiency.
<!--more-->

# The creational problem
A good structured tests exists out of a **Given**, when and then part.
Within the **Given** part very often you're in need of a set of object that will 
drive your test. In most project those object can become complex or lengthy to create.

# Test Data Factory Fixture Generators

```java
class Foo {
    public void foo() {

    }
}
```
