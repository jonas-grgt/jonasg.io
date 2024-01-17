---
title: "<span style=\"color: #606a79; font-size: 90%;\">Test First － Part 3</span><br/> Ancient Testing Pyramids"
head_title: "Ancient Test Structures Of The Past"
description: "Ancient Test Structures Of The Past" 
date: 2023-12-27
tags:
  - java
  - testing
  - TDD
published: false
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---
On the Relevance of the Classical Testing Pyramid
<!--more-->
---
In [the second part](unit-test-ambiguity) I stress the importance of focussing on testing actual requirements, behavior,
use-cases not implementation details like methods and classes.
Through shedding a new light on the classical testing pyrmaid,
this article will show that there are exceptions to this and at times it is necessary to test implementation details.

## Classical Pyramid

The idea is that your test suite consists out of 
- a lot of Unit Tests
- fewer Integration Tests
- as little as possible End To End tests
