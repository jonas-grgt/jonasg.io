---
title: "<span style=\"color: #606a79; font-size: 90%;\">Test First － Part 4</span><br/> Architecture is not only about production code"
head_title: "Architecture is not only about production code"
description: "Architecture is not only about production code" 
date: 2024-03-10
tags:
  - java
  - testing
  - TDD
published: true
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---
Your tests also deserve some architecting
<!--more-->
---

## TL;DR
Partition first by behavior, then by architectural layers

---
Architecture plays a critical role at all levels of the software development process, yet one level often gets overlooked.

Typically, our architectural efforts are focused on production code, 
<b style="color: #3da6b1;">often overlooking a vital component: the test code.</b> 
To maximize the benefits of your test suite,
integrating your test design into your overall architectural decisions is essential.
## Your tests also deserve some architecting
The test suite is often treated with an **indifferent attitude**, as if its structure and 
design are of minor importance.
Even when somewhat architecturally cared for,
it seldom extends beyond dictating a certain percentage of code coverage,
which on its own is a poor metric of quality.

A thoughtfully designed test suite is essential for the sustained success of any application,
facilitating continuous and steady advancement of the production code base.
It also contributes to a coherent and comprehensible base for future changes.
These benefits are frequently underestimated.

A testing architecture must provide clear guidelines on several essential aspects:
- **Levels and Types of Tests**: specify which types of tests are conducted at different levels of the application and explain the reasoning behind these choices.
- **Test Data Management**: define strategies for setting up and tearing down test environments, which could include using patterns like 
  - Test Object Managements using, i.e. [Object Mothers](/posts/object-mother) 
  - Spring Context Management.
  - databases, message-brokers and other infrastructural components _data setup and clean-up strategies_.
- **Reusable Components and Extensibility**: Identify available reusable components, such as standardized fixtures or helper methods. 

Documenting these and other resources promotes efficiency, reduces redundancy, and fosters uniformity across the testing code base.

## The right level of partitioning
Software architecture abstracts an application into **layers or components**,
each with a dedicated responsibility
What often happens is that <b style="color: #3da6b1;">these layers are used to partition 
your tests by a specific type of test.</b>

This line of thinking leads to a brittle test code base that breaks down upon every refactoring.

Even worse is when all layers are tested individually in isolation completed by one big integration test. 
Ever saw a mapper being tested in isolation? That's a clear sign of this anti-pattern.

If not a mapper-test then maybe a test like this looks recognizable?
```java

class PersonService {
    public void createPerson(Person person) {
        personRepository.save(person);
    }
}

// Test Code
var personRepository = mock(PersonRepository.class);
var personService = new PersonService(personRepository);

var person = PersonMother.male();
personService.createPerson(person);

verify(personRepository).save(person);
```

At first glance, this test seems reasonable – it checks if `PersonService` calls `save` on `personRepository`. 
However, this test is inherently flawed as it tightly couples the test to a specific implementation detail of PersonService. 
The issue here is that if the implementation of `createPerson` changes, even if the overall behavior remains the same, the test may fail. 

This creates a fragile testing environment, where tests break due to implementation changes rather than actual bugs or behavioral changes.

Still not convinced? Then you probably misunderstood what is meant by behavior.
## Behavior once more
By **behavior**, I'm referring to  <b style="color: #3da6b1;">the overall functionality of the application as it relates to business rules and user interactions, 
not the details of code, methods, or classes.</b> This behavior is the practical manifestation of what the application does, making the code itself a secondary, supporting detail.

To build more robust tests, it's vital to partition tests based on behavior before partitioning them based on architectural layers.
Partitioning based on architectural layers is not wrong perse, but it should not be the primary partitioning strategy. 

By focusing on what the application is supposed to do, rather than how it does it (its implementation), tests become more resilient to changes in your codebase. 

