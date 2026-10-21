---
title: "Who takes over when the autopilot quits"
head_title: "Who takes over when the autopilot quits"
description: "The case against black box engineering"
date: 2026-10-21
tags:
  - ai
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

The case against black box engineering
<!--more-->
---

After watching [DHH's Rails World 2026 keynote](https://www.youtube.com/watch?v=vDjW_dRyKXY), I was amazed by the ideas
he presented and all the possibilities that come with them. Yet at the same time I felt **unsettled**, **scared** for my
future as a software engineer, and honestly **angry**.

Since then, I've been thinking about it constantly, and about what it means for me. This is a write-up of my current
thoughts as of October 2026. I'm sure my opinion will change as I learn more.

If you go with DHH's vision, the future of software engineering is a dark factory. You write a prompt, and the AI writes
the code. The programming language doesn't matter at all, so English has become the new programming language. You don't
read the code, you just check whether it works.

DHH isn't the only one talking about it. **Dark factories** are something people are really advocating in different
places. Using the current models, I do see the appeal, although I don't think we're there yet.

I get why people like the idea. Management wants to ship faster and cheaper, AI vendors want you to use their tools
more,
and developers have always liked automating away the boring work.

But engineering cannot be compared with creating a generic, reproducible factory part. The code has to be debugged,
trusted and maintained long after it's generated, and someone has to understand it when it breaks.

## Skill Atrophy

Most devs will recognize the friendly manager who likes to mention that he also wrote code back in the day. And you
realize that in today's world he wouldn't be able to do anything. The skill faded because he stopped using it.

Writing code is a muscle. If you stop using it, you lose it, and with it the ability to tell whether the output is any
good.

But it's not only about writing code. Narrowing down a problem is a muscle too: forming a hypothesis, ruling things out,
and making the problem smaller until you find the cause. That can only be done if you have some kind of technical and
business understanding. That is what you rely on when something breaks in production. In a dark factory, nobody
practices it anymore, so every engineer becomes that manager.

Aviation has seen this before. Pilots fly on autopilot for roughly 90 to 95% of flight time, which is why the FAA urges
them to hand-fly regularly. When the autopilot disengaged on
[Air France 447 in 2009](https://en.wikipedia.org/wiki/Air_France_Flight_447), the pilots misread a stall and crashed,
killing all 228 people on board. Investigators found no convincing evidence that their manual flying skills had been
properly maintained.

Engineers who stop using their engineering muscle will end up in the same situation.

## You still have a choice

I use AI, and it makes me better. I'm not against it.

But there is a choice here. You can get lazy and slowly become a dark factory engineer who prompts, glances at the
output, and ships. Or you can choose not to, at least for now.

We need to understand more than a piece of code, and more than a single codebase. Joel Spolsky made this point years ago
in *Things You Should Never Do, Part I*: the ugly parts of old code often hold knowledge that was earned the hard way,
and you lose it if you don't understand why it's there. We need to understand the whole system, on a business level and
a technical level. There is so much going on, and so much that can go wrong. You need to be in charge and know what is
happening.

If you're the engineer who just presses enter, you'd better find another job. Now is the time to rise and say: I really
want to understand how this works, deeply. And you can do that with AI. Used well, it helps you understand things on a
deeper level and makes you a better engineer, not one who's easy to replace.