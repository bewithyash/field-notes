---
title: "Threat Modeling That Ships"
date: August 2026
source: https://blog.imperonlabs.com/blog/threat-modeling-that-ships.html
---

Methodology · Architecture

# Threat Modeling That Ships

By Imperonlabs Research · August 2026 · 12 min read

A threat model that does not change the design or the test plan was theatre. It is not a document you produce to satisfy an auditor, it is a decision-making exercise, and the decision it ends in is a ranked list of changes. If nothing gets reprioritised, rebuilt, or retested because of it, you did not model threats. You drew a picture.

Working rule. The output that ships is fifteen plausible threats you can act on, sorted by impact times likelihood. Not two hundred exhaustive ones nobody reads. Ruthless filtering is the skill, not enumeration.

## Four questions, and the one people get backwards

Adam Shostack's framing is still the cleanest way in, and it is four questions.

1. What are we building? Draw the system: components, data stores, and the flows between them, on one diagram.
2. What can go wrong? Enumerate threats adversarially.
3. What are we going to do about it? Produce ranked, concrete mitigations.
4. Did we do a good job? Check the work honestly.

The near-universal failure is inverting the effort. Teams spend an afternoon perfecting the diagram, colour-coding the arrows, aligning the boxes, and then five rushed minutes on "what can go wrong" at the end of the meeting. Flip it. The diagram exists to serve the enumeration. Get it good enough to reason over, then spend your real energy on question two.

## Draw one diagram, then find the red lines

One diagram, showing where data lives and where it moves. The value is not in the boxes, it is in the lines where data crosses from one owner or privilege level to another. Internet to edge. Application to database. Tenant to tenant. Unauthenticated to authenticated. Those are your trust boundaries, and every boundary owes you a control. In my experience most real vulnerabilities are exactly one thing: a missing or wrong control on a boundary crossing. Mark the boundaries and you have found where to point question two.

## STRIDE, applied and then filtered

At each boundary I walk STRIDE and keep only what is plausible for this system.

- **Spoofing**, weak or missing authentication of a party.
- **Tampering**, integrity of data or code in transit or at rest.
- **Repudiation**, gaps in the audit trail.
- **Information disclosure**, authorization or encryption failures.
- **Denial of service**, availability controls.
- **Elevation of privilege**, the category that usually ends up at the top of the list.

The temptation is to write down all two hundred things STRIDE can generate. Resist it. A threat that is not plausible against this system, or whose impact is trivial, or whose likelihood is a rounding error, is noise that buries the fifteen that matter. Filtering is not laziness. It is the entire point.

From enumeration to a list you can ship

enumerate
~200 raw
STRIDE per
boundary

filter
plausible only
impact x
likelihood

15 ranked changes
design + test plan
move because of them
The narrow end is the deliverable. The wide end is just raw material.

Enumerate wide, filter hard, ship narrow. A threat model is judged by what it changes, not by how long the list is.

## The threats teams forget

Some categories vanish because they sit in nobody's mental model of "the app." These are the ones I make a point of raising:

- Service-to-service authentication, and background jobs running with far more privilege than they need.
- The assumption that anything labelled "internal" is therefore safe, so it never gets a control.
- Multi-tenant data isolation at every layer, not just the one where it was designed in.
- Supply chain, meaning the CI/CD pipeline that can push code straight to production.
- Failure-state behaviour, above all the authorization check that fails open when the identity service times out.

## Rank roughly, and keep it alive

Score each surviving threat by impact times likelihood. The goal is prioritisation, not false precision, so a coarse high-medium-low on each axis is plenty to sort the list. Then treat the model as a living document. It is updated when the architecture changes, not filed once and cited in an audit two years later when it no longer describes the system at all. A threat model is only true for the design it was drawn against.

The one line to remember
Judge a threat model by what changed because of it. If the design and the test plan look identical before and after, you produced a diagram, not a decision.

### Does your threat model change anything?

Imperonlabs runs threat modeling that ends where it should, in a short ranked list your engineers ship against and your testers hunt from. Book a scoping call.

[Request a session](https://imperonlabs.com/#contact)
