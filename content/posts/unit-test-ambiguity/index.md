---
title: "<span style=\"color: #606a79; font-size: 90%;\">Test First － Part 2</span><br/> The Unit Test Ambiguity"
head_title: The Unit Test Ambiguity
description: "TDD is Not a Religion" 
date: 2023-12-27
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
Why the Unit in Unit Testing does **not** matter.
<!--more-->
---
In the first part of this series, I stress the importance of prioritizing a well-designed and overall reliable 
test suite over strict adherence to the TDD mantra.
However, defining what constitutes a well-designed and reliable test suite raises the question of 
<b style="color: #3da6b1;">what should be tested, or more precisely, what the unit under test is.</b>

If you were to ask a group of developers to give a definition of a Unit Test, you'd likely receive a plethora 
of different responses. 
Likewise, within the industry, there is a lot of ambiguity around what a Unit Test exactly is. 
We simply cannot agree on a common concept what the unit exactly represents.

In the second part of this testing series, I aim to clarify this ambiguity by defining what actually should be tested 
or what the unit actually is.
This will pave the way for a deeper understanding of the true value of writing tests.
## Not The Smallest Testable Module
Unfortunately, the term "Unit Testing" has become synonymous with testing overall. Conversely, a common definition 
of Unit Testing is the smallest testable component that can be isolated[^1]. For many, this translates to testing a 
single public method or class, leading to the pitfalls of brittle test suites and misconceptions about testing and 
difficulties with TDD or how I like to call it Test First.
## Unit Tests: A Source of Brittleness
Test code bases composed of numerous small tests focused on methods and classes often suffer from brittleness. 
A brittle test suite breaks with even the slightest code alterations unrelated to changing requirements. 
This becomes particularly evident when routine code refactors, such as adding a dependency or a parameter, 
result in test failures.

If your test code base fractures in response to minor alterations **without changes in requirements**, 
it's a clear indication of a poorly designed test suite. Awareness of this issue may be lacking, especially if code 
coverage numbers present a seemingly positive picture. However, focusing solely on code coverage provides a narrow 
perspective on the overall health of your test code base.

Additionally, these types of test suites fall short in narrating the essence of the application. 
Instead of conveying the domain, they tend to focus on technical implementation details. 
Similar to the concept of screaming architecture, tests should vividly scream what the application is about.[^2]

The fragility and lack of documentation in tests can be attributed to an excessive emphasis on testing code in isolation. 
However, it prompts the question: 
<b style="color: #3da6b1;">what else should be tested if not the code we just wrote or want to write?</b>
### A pinnacle of brittle tests focused on a technical implementation detail
Admittedly, a very simplistic example, yet I have already encountered tests that look almost exactly like this:
```java
class PersonService {
	public void createPerson(Person person) {
		personRepository.save(person);
	}
}
```

```java
var personRepository = mock(PersonRepository.class);
var personService = new PersonService(personRepository);

var person = PersonMother.male();
personService.createPerson(person);

verify(personRepository).save(person);
```
## The Test Trigger
> The trigger for a test is not a class nor a public method<br/>
> － Ian Cooper － TDD where did it all go wrong[^3]

For an extended period, adding a new class or method was exactly what was driving my tests. 
Ironically, it was this idea on testing that made TDD comprehensible for me.

This commitment found additional reinforcement through the abundance of straightforward examples and tutorials 
available on writing tests and TDD, all consistently emphasizing explicit code testing. 

<b style="color: #3da6b1;">I believe that one should not test code explicitly, most of the time.</b>
So then naturally, the question arises, what should be tested explicitly if not code?
## Behavior First
Today I no longer let the creation of a method or class drive my tests. 
Instead, I focus on a fundamental question: **why do I write code?**
I write code because there is always a requirement for a system to behave in a certain way.
And that I try to capture in my tests.

This perspective should guide testing *most of the time.*
Tests should stem from the requirements expected to be implemented, how the system should behave and
not the methods and classes implementing those requirements.

> Explicitly Test Behavior To Implicitly Test The Code Covering That Behavior

<b style="color: #3da6b1;">By explicitly testing the expected behavior of your application, 
you will implicitly test almost all of your code.</b>
Any portions left untested by this method are likely unrelated to expected behaviors, relate to exceptional cases, 
such as checked exceptions, or configuration classes. As a consequence, your code coverage should be naturally high.

I emphasized the significance of letting behavior guide your testing in the majority of cases. 
However, are there instances when this may not be the optimal approach? 
This question will be answered in the upcoming third installment of my Test First Series,
where we go back in time to take a look at the classical Testing Pyramid.
[^1]: Wikipedia (2023) https://en.wikipedia.org/w/index.php?title=Unit_testing&oldid=1186674931
[^2]: Screaming Architecture (2021) https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html
[^3]: TDD where did it all go wrong (2017) https://youtu.be/EZ05e7EMOLM?si=2M-K1F0O53EYChU0

