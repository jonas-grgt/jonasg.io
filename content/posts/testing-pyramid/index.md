---
title: "<span style=\"color: #606a79; font-size: 90%;\">Test First － Part 3</span><br/> Ancient Testing Pyramids"
head_title: "Ancient Test Structures Of The Past"
description: "Ancient Test Structures Of The Past" 
date: 2024-01-26
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

## TL;DR

The classical testing pyramid focuses on the cost of writing, running,
and maintaining tests,
which is a very narrow and single-minded perspective
not adapted to today's technology landscape.

---


In [the second part](unit-test-ambiguity) part I stress the importance of focussing on testing actual requirements, behavior,
use-cases not implementation details like methods and classes. Yet things aren't merely as simple as that.
Through shedding a new light on the classical testing pyramid,
this article will show that at times testing your software isn't as black and white and that there are a lot of grey zones. That you should not only test behavior but at times it is necessary to test implementation details.

# A short history lesson
Most developers are familiar with the concept of the testing pyramid, introduced by Mike Cohn in his book "Succeeding with Agile: Software Development Using Scrum" back in 2009. This metaphor illustrates an ideal distribution of different test types, emphasising a strong foundation of unit tests, followed by integration tests, and topped with a smaller number of end-to-end tests.

The pyramid focusses on one aspect of test type distribution that based upon the cost, in time and money, of writing running and maintaining your tests. Nothing can be said about it's distribution regarding the cost yet it is the only value it regards while there are others.
![](pyramid.png)

# Weak Foundation
In today's software development world, the testing pyramid's foundation isn't as sturdy as it once was. It's built on the idea that having many unit tests equals a well-tested application, but this can be **highly misleading.** As we discussed before, the concept of unit testing in our industry is unclear, leading us to write a lot of isolated tests while mocking out all external dependencies which in its turn leads to a brittle test code base.
The focus on technical distribution steers us away from the real goal of **testing behavior**.

In essence, the classical testing pyramid tells us how to test things, but it's not that relevant because it doesn't help us:
- prevent regression
- document or software
- refactor freely
- evolve our applications
  all of that when making changes without altering requirements.
# Honeycomb to the rescue
Because of a changing technology landscape, we do not architect applications in the same way as when the classical testing pyramid was created nor do we have to work with the same kind of hard and software tools.

A shift has been going on for over a decade towards thinner services away from monolithic applications. Such a service is often much more simplified than decades ago were the business responsibility is trimmed down whereas the integration parts with other services and infrastructural components have gained importance.

A jump in cpu cycles together with a leap forwards in virtualisation lead us in an area were it is possible to spin-up a whole database in a matter of seconds, using tools like Testcontainers.

For those reasons the testing honeycomb emerged putting a bigger emphasis on integration tests  and focusses to a lesser extend on implementation detail and integrated tests
# A Honeycomb++
While I can see absolute value in the testing honeycomb I do like a slight alteration.
![](honeycomb.png)


## Behavior

At its core you test the behavior of your system by whatever means that is, unit or integration test. The choice or rather that architectural decision is yours to take and is different for every context.
## Implementation detail
## Integrated 
