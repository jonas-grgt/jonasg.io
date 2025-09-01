---
title: Keeping Spring Boot Test Configurations Simple and Maintainable
head_title: Keeping Spring Boot Test Configurations Simple and Maintainable
description: How to manage `application.yml` files in Spring Boot tests without duplication.
date: 2025-09-01
tags:
  - java
  - spring-boot
  - testing
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

How to manage `application.yml` files in Spring Boot tests without duplication.
<!--more-->
---
Spring Boot applications rely on an `application.yml` file to define configuration.
Things get a little trickier when writing tests, since both `src/main/resources` and `src/test/resources` can contain
configuration files. Understanding how Spring Boot handles them will help you avoid duplication and keep test
configuration clean.

## How Spring Boot Resolves application.yml

When running the application normally, Spring Boot loads configuration from `src/main/resources/application.yml`.

When running tests, the classpath is a combination of main and test outputs, with test classes and resources placed
ahead of main ones.

This means:

- If you only have `src/main/resources/application.yml`, that file will be used during tests as well.
- If you also define `src/test/resources/application.yml`, the test version will take precedence. The main version won’t
  be
  loaded at all unless you explicitly import it.

So the test application.yml does not “merge” with the main one, it shadows it.

## Our Definition of A Clean Setup

Our main goal is avoiding duplication. Without a deliberate setup, you often end up copying the entire `application.yml`
into `src/test/resources` and maintaining two versions side by side. This makes configuration brittle and harder to
manage.

A clean setup should:

- Keep the main configuration as the single source of truth.
- Allow test-specific overrides without repeating the full configuration.

## A Clean Setup with Profiles and Imports

To avoid duplicating configuration between test and main, you can make the test configuration explicitly import the main
configuration and then layer test-specific overrides on top.

**1.** `src/test/resources/application.yml`

```yaml
spring:
  profiles:
    active: test
  config:
    import: file:src/main/resources/application.yml
```

This setup:

- Activates the test profile whenever tests run, so it will load `application-test.yml` files.
- Reuses the main `application.yml` by importing it.

**2.** `src/test/resources/application-test.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    hibernate:
      ddl-auto: create-drop
```

Here you define overrides that apply only when the test profile is active. Typical use cases include swapping out the
database connection, adjusting logging levels, or changing service endpoints.

## Why This Works Well
- **Single source of truth**: Your main application.yml stays the canonical configuration.
- **Minimal duplication**: Test-specific overrides live in one dedicated file.
- **Profile isolation**: Tests always run with the test profile, avoiding accidental reliance on production defaults.
- **Explicit precedence**: By importing the main config, you make it clear that tests extend it rather than replace it.