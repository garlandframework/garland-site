---
title: "Why I Built Garland"
description: "Laid off after training AI to write tests, I built a typed-chain Java framework that keeps AI-generated tests honest — and changed how I see the SDET role."
date: 2026-06-16
draft: false
tags: ["java", "testing", "AI", "microservices"]
---

## How it started

The story of this framework started the day I was laid off from the company I worked at. I was asked to train AI to write automated integration tests, and then one day I just lost access to company resources, the repo, all of it. The company decided they didn't need me anymore.

Was the AI good enough to make my role less necessary? Probably, at least partly. Was it also just bad timing — budget cuts, restructuring, the usual reasons companies give? Probably that too. Nobody gave me a percentage. I just got a layoff.

That's not the most original story either way — "laid off SDET decides to create his own framework." But I did it anyway.

## The problem

Weeks of training AI to write tests let me collect information and notice patterns in how it actually generates tests. Yes, it can generate tests, and very often quite successfully. But the problem is that "very often" is not "always," and the cost of an error can be huge — especially if the error is hidden somewhere in backend logic. Even the most detailed .md specs don't prevent AI from hallucinating and reinventing the wheel. And over time, those specs turn into one huge, poorly structured mix of instructions and exceptions.

Another problem is the code itself — HTTP interactions and assertions are done one way, Kafka is something different, Mongo and Postgres something else again. That's not just a big space for AI to hallucinate in, it's also painful for a human to review. Wrapping those interactions in nice helpers doesn't solve the problem fully, either — it just hides it. The test itself gets cleaner, but the internals of the helpers are still the same hallucination space.

So if the code is difficult or impossible to review, and the hallucination factor isn't zero, how can the tests be trusted? The answer seemed obvious: make it easy to review, and put guardrails around the AI so it follows the rules and logic of one framework — instead of generating one more db layer just to select a row from a new table.

## Looking for a solution

If we look at the backend testing process in general (and basically any testing), it's often sending data to a system and verifying that the outcome matches what's expected. In HTTP: JSON in — JSON out. In Postgres: entity persisted — entity can be found. In Mongo, similar. In Kafka: object pushed to a queue — object can be read from the queue.

Basically it's always object in — object out, and the whole process can be represented as a chain of functions that take some object as input, do something with it, and output some object. And that's exactly what Garland is: a chain of steps where each step is a function. This applies not just to tests, but to the framework's internals too.

When AI is forced to generate chains where the type returned by each step must fit the type accepted by the next step, it leaves much less space for hallucination. If the AI decides to generate something weird, it simply won't compile. And a DSL that looks like a chain isn't difficult to review either.

Garland puts restrictions on what can be done and how it can be done, for a reason: to make it easier to generate and review.

## Implementation

Garland was designed to be modular from the very beginning. Garland-core is basically a generalized version of the chain mentioned above — it knows nothing about HTTP or Mongo, it just defines the main idea: a chain of functions. garland-http, garland-kafka, and others are the actual implementation for a particular case, but they're the same at their core — chains of functions. For example, HTTP is a chain: object in → serialize (function) → make the actual call (function) → deserialize (function) → assert result (function) → output object. Kafka and Mongo are the same, even simpler because there's no serialization/deserialization.

Of course, somewhere deep under the hood there's an actual wrapper for the http client, kafka client, etc., but it's never accessed directly and isn't meant to be. Since there's always some time lag between http → db → kafka → mongo, a time tolerance and retry mechanism is built into it.

Each module exposes a test client to the user, and that test client lets you make a call to http, kafka, or another part of the system, specify a check, configure retry, and define the expected result. All in one chain.

The next step is chaining http to kafka to db and so on. It's not something that's often needed, but when you need it — you need it bad. Debugging this manually — checking what data goes into the http call, what's actually persisted in the db, whether the object shows up in kafka afterward, and what happens in mongo after that — takes time. And that's exactly the moment when Garland can help.

Data returned by http can easily be transformed into entities or any other object needed, using data mapping libraries, and that transformation step can be a function too. And if it can be a function, then it can chain together different parts: http to db to kafka, for example. Of course, mapping should be carefully reviewed by a human, because this is where actual business logic may hide. But that's not a big price to pay, and testers should know this anyway.

## Test generation

After implementing the core and its modules as DSL guardrails, I found I needed a real system to check if my framework actually worked. And that's how the garland-demo project came to be. I needed a real service, real Kafka, Mongo, Postgres — and it had to be something that actually works, not just mocked. The whole idea of Garland was that it should test a real system.

So I asked Claude to generate two services that write data into a db, put messages on Kafka, and then write to Mongo. A completely vibecoded system, very far from perfect. And that was the point — I had to check whether my generated tests would find any bugs.

Having experience with the .md spec mess, I decided to use Claude Code's slash commands and organize them by domain — users domain, orders domain, etc. The idea was simple: generate tests using the Garland pipeline DSL, compile, run, and fix using AI — then have it learn from the mistakes it made and write that down to .md files. This isn't much different from any existing AI workflow. What was different was the content and size of the .md files.

Since the AI is always guardrailed, the code always follows the same pattern, and the examples it puts into the .md files look the same. As a result, the AI needs fewer examples to understand your system, and it adds fewer exceptions to the .md files. Overall, the size of the .md files becomes bearable to review and understand.

After a few iterations of corrections, Claude Code started to generate a quite good set of tests for my vibecoded system, and it actually found bugs. And there was no need to train it for weeks — it was done in a day or so.

## What I learned

There is a lot of controversy around AI testing, and some people even say SDET is dead. Well, probably some aspects of SDET work will become obsolete — writing tests that cover an endpoint's negative scenarios is not a valuable skill anymore. AI can do that. One might also ask me: you were replaced by AI, and as a result you built a framework that helps AI be even more effective — what's the point?

The answer is that while building the framework and generating the tests, I started to change my view of the system under test. When you no longer need to spend days covering an endpoint with primitive negative scenarios, you start asking different questions. What is the purpose of this data flow? How critical is it? How does it transform data? Which part of the data is critical, and which isn't? How do different domains relate to each other? And so on.

Those are questions a regular SDET has no time to think about, because they're too busy with trivial test cases. But they're questions AI cannot easily answer, and they actually require understanding the whole product — which makes an SDET something more than just a producer of negative test cases for HTTP endpoints. SDET is dead, long live the SDET!

---

## Links

- [Quick start](/docs/tutorials/quickstart/)
- [GitHub](https://github.com/garlandframework/garland)
- [Demo project](https://github.com/garlandframework/garland-demo)
