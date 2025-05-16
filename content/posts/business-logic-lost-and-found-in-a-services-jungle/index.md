---
title: "Business Logic: Lost and Found in a Services Jungle"
head_title: "Business Logic: Lost and Found in a Services Jungle"
description: "How to make your code speak the language of business"
date: 2024-08-20
draft: true
tags:
  - java
  - architecture
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

How to make your code speak the language of business
<!--more-->
---

**⚠️ Disclaimer: The following content only applies to fairly large projects that are
business logic heavy.**

The all encompassing set of service classes does not need any introductions as we all have
encountered them in our projects. They are the backbone of our applications, the place
that executes and or orchestrates the business logic over a single aggregate.

- `UserService`
- `OrderService`
- `ProductService`
- `PaymentService`

## What is wrong with services?

Service classes have the tendency to become a dumping ground for business logic molding it
into a big ball of CRUD. 
At times, it is very deceiving to mold business logic into CRUD operations because, in the
end; looking at it from a technical point, a lot of business logic simply boils down to
doing a CRUDy database operation.

In example:

- Registering a user on a technical level, essentially boils down to creating a record in the
database. 
- Placing an order is just a matter of creating a record in the database.
- Changing the delivery address is just a matter of updating a record in the database.


But are they really?

### Ubiquitous language
