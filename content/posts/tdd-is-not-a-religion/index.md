---
title: "Part 1: TDD is Not a Religion"
description: "TDD is Not a Religion" 
date: 2023-12-14
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
On the religious nature of TDD
<!--more-->
---
Today, I practice a form of Test-Driven Development (TDD) whenever and wherever possible. 
However, this wasn't always the case. At the beginning of my career, I was oblivious of it. 
To be honest, I wasn't even focussed on writing tests; I was already content if the production code worked, 
and only then I would write a test.

As much as TDD practitioners would like, the adoption of the methodology, introduced by *Kent Beck* nearly 20 years ago, 
remains relatively low[^1]. 
Conversely, an intriguing paradox emerges when we look beyond the mainstream landscape. 
TDD remains very much alive, championed by a large group of developers who diligently preach its virtues on social 
media channels and conferences.

In this article series, my goal is to offer a fresh perspective on TDD that goes beyond its common glorification 
served with oversimplified examples and rigid doctrine. I will provide practical insights to make TDD more 
recognizable and applicable to the complex challenges developers encounter today. 
Ultimately, shifting away from TDD towards the very similar yet subtly different and profoundly more usable 
<b style="color: #3da6b1;">Test First</b> approach.

## No TDD Never Implied no Tests
Compared to its adoption rate, it is prodigious to see TDD listed in many developer vacancies. 
Nevertheless, despite my experience spanning more than a decade in diverse teams, companies and projects, 
I've rarely encountered developers who wholeheartedly adopted TDD.

> TDD is like a rare bird in the wild, seldom seen but fascinating when it does appear.

<B style="color: #3da6b1;">Not applying TDD never implied the absence of tests altogether.</b> 
In fact, every project I've worked on has maintained a fair share of tests.
Likewise, a significant number of developers with whom I've collaborated regarded a robust test suite as a 
cornerstone for the success of their projects. However, what I found to be a rarity was the practice of writing 
tests first, using these tests to guide the design and engaging in the classic red-green-refactor cycle.

In these environments, testing was perceived as a safety net rather than a compass for design. 
Developers wrote tests after the code was implemented, and the purpose of these tests was primarily to 
confirm that the code worked as intended and to warn them of regression later on.
## A Valuable Skill or Dogmatic Doctrine?
TDD undeniably played a significant role in putting testing on the map. 
It had a huge impact on how we develop software today and tomorrow. Nevertheless, 
it's high time to confront the reality of the dogmatic perception TDD has acquired and cast a more 
illuminating spotlight on it.

While TDD stands as a valuable methodology for ensuring software quality,
<b style="color: #3da6b1;">I do not blindly want to follow its stringent approach</b>. 
While its rules[^2] can helpful, they sometimes inadvertently constrain our approach to problem-solving:
- Not writing production code unless a failing test exists.
- Not writing more of a test than is necessary to fail.
- Not writing more production code than is needed to pass the one failing test.

TDD is a skill that takes practice and experience to really benefit from it. 
It cannot be rigorously applied everywhere; experience will teach you this

>  Misapplication or overzealous adherence to TDD principles can lead to exactly the opposite of what it might try to achieve.

In the world of software development, having a diverse skill set is essential. Therefore,
<b style="color: #3da6b1;">I see TDD as one of the many skills that make up a developer's arsenal</b>, 
each contributing towards becoming a well-rounded and proficient software engineer.
## Emphasizing the Destination Over the Journey
In practice, I've learned that the presence of tests is more important than the specific details of how they are written. 
Whether tests are written using the TDD methodology or not, what truly matters is their role in ensuring 
the software's correctness, guarding against regressions, serving as documentation and all together allow our 
software to evolve at a steady pace.<b style="color: #3da6b1;"> While TDD may provide a structured approach, the ultimate goal is maintainable, reliable and well-tested software.</b>
## TDD Evolving with the Changing Software Landscape
It's essential to acknowledge that the world of software development has undergone a remarkable transformation since TDD was first introduced nearly two decades ago. With the exponential growth of the web came and need for of microservices, the advent of new programming languages, the evolution of architectural patterns, new types of databases, queues and the ever-increasing speed and power of our computers. We are no longer creating the same applications as we did in the past.
With the advent of microservices I've seen the domain shrink and the integration part of applications grow.
I argue that these shifts have undoubtedly influenced the way we write tests.

I firmly advocate <b style="color: #3da6b1;">attempting to write your tests first</b>, as it's a valuable approach that doesn't necessarily entail the strict adherence to the rules prescribed by traditional TDD.

Exploring the advantages of writing tests first will be our focus in the upcoming sections. 
However, before looking into that, the next episode will address **The Unit Test Ambiguity**, as it often hinder discussions about TDD.

[^1]: Diffblue. (2020). "DevOps and Testing Report." Retrieved from [https://www.diffblue.com/DevOps/research_papers/2020-devops-and-testing-report/](https://www.diffblue.com/DevOps/research_papers/2020-devops-and-testing-report/)
[^2]: TheThreeRulesOfTdd. (2023). http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd
