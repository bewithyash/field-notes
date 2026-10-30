---
title: "Business Logic Flaws: The Bugs Scanners Never Find"
date: August 2026
source: https://blog.imperonlabs.com/blog/business-logic-flaws.html
---

Web · Methodology

# Business Logic Flaws: The Bugs Scanners Never Find

By Imperonlabs Research · August 2026 · 13 min read

A scanner is very good at catching a broken rule of HTTP. Reflected input, an injectable parameter, a missing header. It is useless at catching a broken rule of your business, because nobody told it what your business promised. That gap, between what a feature is supposed to guarantee and what the code actually enforces, is where the bugs that pay the most quietly live.

Scope note. Every example below is demonstrated on test objects with minimal sequences. You prove a logic flaw by showing the invariant break once, not by draining a real balance to see how far it goes.

Injection, XSS and misconfiguration are violations of technical rules, and a machine can enumerate those because the rules are universal. A business logic flaw is a violation of an operational rule that only your application knows: one signup discount ever, no shipping before payment settles, no withdrawal beyond the confirmed balance. The scanner sends a request, gets a 200, and moves on. It has no idea it just watched a user apply the same one-time coupon eleven times.

## Start by writing down the invariant

Before I touch a single request, I state in plain language what the feature is supposed to guarantee. Not the happy path, the guarantee. For example:

- A user can only apply one signup discount, ever.
- An order cannot ship until payment has settled.
- A withdrawal cannot exceed the confirmed balance.

Every one of those is a sentence with an implied "always" or "never" in it, and a logic bug is any sequence of legal requests that makes the sentence false. Once the invariant is written down, testing stops being "let me poke at parameters" and becomes "let me find the request order that breaks this promise." That reframing is the whole method.

## Four patterns that break almost everything

### 1. Out of order

State machines assume the client walks them in order. So do not. Call the checkout confirmation before the payment step. Fire the fulfilment webhook before the entity it references exists. Skip the verification stage and jump straight to the action it was supposed to gate. A surprising number of flows never re-check earlier state at a later step, because the developer assumed the only way to reach step three was through steps one and two.

### 2. Repeat it

This is the highest-value pattern in commerce and fintech, so I always test it. Replay the exact request that grants a benefit. Apply the coupon twice. Redeem the gift card from ten connections at once. The vulnerability is almost always a race between the check and the commit: the code reads "coupon not yet used," ten requests all read the same "not yet used" before any of them writes "used," and a one-time benefit becomes an unlimited one. If a request grants money, credit, or quota, fire it in parallel and count the successes.

The check-then-commit race, drawn out

time →

req 1
req 2
req 3

read: unused

read: unused

read: unused

apply + write: used

apply + write: used

apply + write: used

All three read "unused" before any of them wrote "used." One-time becomes three-time.

Every request validated correctly. The invariant still broke, because validation and commit were not atomic.

### 3. Wrong value

Feed the logic values it never expected a human to send. A negative quantity that credits instead of debits. A client-controlled price field that the server trusts. A currency swap that pays out in the stronger denomination. Decimal precision abuse where `0.0001` rounds to free. Type confusion that flips a branch. The server validated that the field was present and numeric. It never validated that the number made business sense.

### 4. Wrong actor

Let the wrong person trigger the right action. An unprivileged user reaching a privileged endpoint directly. A maker approving their own request in a maker-checker flow, collapsing a two-person control into one. A downgraded account that kept the capability its old tier granted. Authorization bugs of this shape hide because the UI never offers the action, so nobody tested what happens when you call it anyway.

## Test relationships, not requests

The mechanical mistake I see most is testing requests in isolation. Logic lives in the relationship between requests, so I capture a complete, legitimate flow first, then mutate the relationship: reorder it, repeat it, replace a value halfway through, swap the actor on one leg. And I watch state, not status codes. Balance, credits, tier, quota, order status. I record the number before the mutation and after it, because a logic flaw often returns a clean 200 while the invariant quietly breaks behind it.

## Proving it without doing damage

A logic flaw is proven by the smallest sequence that makes the invariant false, run against a test object, with the resulting state shown before and after. Apply the coupon twice and show the balance dropping by two discounts. You do not need to redeem a hundred gift cards to prove the race exists. Demonstrate the break, document the state, and hand triage a story it cannot argue with: here is the promise, here is the sequence, here is the promise broken.

The one line to remember
A scanner tests whether your code follows the rules of HTTP. Only a human who wrote down your invariants can test whether your code follows the rules of your business, and that is exactly where the money is.

### What does your checkout actually enforce?

Imperonlabs tests the promises your features make, not just their parameters. We map your invariants and hunt the request sequences that break them. Book a scoping call.

[Request an assessment](https://imperonlabs.com/#contact)
