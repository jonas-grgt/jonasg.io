---
title: "<span style=\"color: #606a79; font-size: 90%;\">Test First － Part 5</span><br/> TL;DR"
head_title: "Test First - TL;DR" 
description: "Too long; didn't read of the Test First series." 
date: 2024-03-17
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
TL;DR of the Test First series.
<!--more-->
---

As we conclude the Test First series, I'd like to revisit and emphasize the key takeaways from our discussions.

Test Driven Development (TDD) is a practical tool that,
when understood and **applied correctly, within the right context**, can be highly beneficial.
TDD isn't about [adhering to a strict dogma](/posts/tdd-is-not-a-religion/),
or looking down on those who haven't adopted it;

**TDD is fundamentally about ensuring that your codebase remains robust against regressions during:**

- Refactorings
- Introduction of new features
- Modifications to existing features

TDD achieves this by playing a crucial role in the design of our applications.
Once we have developed **a working feature**,
regardless of its initial code quality or elegance,
having robust tests in place enables us to modify the internal workings with confidence.
This is what the whole red-green-refactor cycle is about.

Such an approach ensures that changes made to improve
or optimize the code
(after our initial green tests or even months later when refactoring or altering the feature's behavior)
do not inadvertently introduce errors or regressions.

Do not confuse a working feature by working code.<b style="color: #3da6b1;"> 
A feature is implemented by code, but the code itself is not the feature nor the goal 
of what should be tested.</b>

To effectively achieve this,
it's crucial to write tests at an appropriate level of abstraction.
But what is the right level of abstraction?
# Stop wasting time Unit Testing
A Unit Test, as all to often misunderstood as a method or class test,
is definitely not the right level of abstraction
to form a proper foundation of any test suite.
For that reason, the classical testing [pyramid has become obsolete.](/posts/relevance-of-the-classical-testing-pyramid//)

The foundation of your test codebase, if based on testing each layer,
class, or method in isolation, tends to be counterproductive.
This approach can result in a fragile codebase,
one that is prone to failure with the slightest changes,
rather than fortifying it against regressions.

At the heart of your testing approach should be a focus on behavior,
followed by implementation details.
Grasping this distinction within your own codebase is a significant aspect of test architecture.
Understanding that tests should primarily verify the behavior of the system —
how it reacts and what it does —
and then consider the specific ways these behaviors are implemented, is key.
This approach ensures that tests remain relevant and valuable,
even as the implementation evolves over time.
# Architecture also for our tests
It's vital to treat our test codebase with the same seriousness as our production code architecture.
Planning and understanding the [rationale behind the structure of the test suite are essential](/posts/architecture-is-not-only-for-production-code/).
This clarity should extend to all project participants,
ensuring a cohesive and effective testing strategy.

# Test First
Writing tests
before production code encourages thoughtful consideration of the system's requirements and behaviors.
It also puts the focus on testing architecture -
figuring out how to test the features - enhances the process.
In contrast,
starting directly with production code can turn these considerations into hurdles.

A Test-first mindset aims to enable the creation of better, more change-resilient production code, 
by integrating testing as a fundamental part of the development process,
if conducted at the right level of abstraction.

<b style="color: #3da6b1;">In the end,
our tests are more important than our production code.</b>
Because without tests,
we can't be confident to change our production code
and in doing so being able to move our software products forward at a steady peace.
