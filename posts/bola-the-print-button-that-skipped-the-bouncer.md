---
title: "BOLA: The Print Button That Skipped the Bouncer"
date: August 2026
source: https://blog.imperonlabs.com/blog/bola-the-print-button-that-skipped-the-bouncer.html
---

Web · Access Control

# BOLA: The Print Button That Skipped the Bouncer

By Imperonlabs Research · August 2026 · 10 min read

The front door was guarded. I asked the main endpoint for another district's data and it politely returned an empty array, exactly as it should. Then I asked the print action on the same route the same question, and it handed me 132 student records from a district I had no business seeing. The bouncer was real. He just was not standing at every door.

**Scope note.** Sanitized from a real engagement on a multi tenant education SaaS. Hostnames, IDs, and counts are examples. The point is the pattern, not the platform.

The product is a multi tenant platform for school districts. District A must never see District B, and that isolation is the whole security promise of the product. I was testing from a teacher account, the lowest privilege role, scoped to a single district in a single session. The interesting bugs in this kind of app are almost never in the main data endpoint everyone remembers to protect. They are in the derivative actions bolted onto it later.

## Establish the baseline first

Before I hunt for a broken door I confirm the working one, because that is what tells me the app knows how to say no. I took a session identifier from another district and put it on the primary student action plans endpoint:

```
GET /api/admin/v1/student_action_plans?session_id=<foreign> HTTP/2
Host: app.example.com
Cookie: <teacher session>
```

Response: `200 OK` with an empty array, `[]`. Correct behavior. The endpoint resolved the foreign `session_id`, saw it was out of my scope, and returned nothing. Authorization here works. That is exactly the moment to go looking for the endpoint where it does not, because now I know what a properly denied request looks like.

## The sibling that nobody guarded

Real apps grow siblings. The base collection endpoint gets a `/print`, an `/export`, an `/email`, a `/report`. Each one is usually written by whoever needed it that sprint, and the authorization decorator that protects the base action does not always come along for the ride. So I took the same foreign session and hit the print action on the same route prefix:

```
GET /api/admin/v1/student_action_plans/print?session_id=<foreign> HTTP/2
Host: app.example.com
Cookie: <same teacher session>
Accept: application/json
```

Response: `200 OK` with 132 student action plans belonging to a district I could not touch through the front door. Same session, same parameter, same data underneath. The only difference was five characters of path, and those five characters routed me around the check.

One request, one parameter, two very different answers

Teacher + foreign session\_id
lowest privilege, out of scope

/student\_action\_plans
resolves scope, denies
200 OK []

/student\_action\_plans/print
no scope check on this action
200 OK 132 records

Authorization lived on the base action and never made it onto the sibling. The object id was accepted without asking whether the caller was allowed to see that object.

## Proving it is a real breach, not an artifact

A pile of records is not a finding until I can show it is genuinely someone else's data and not, say, my own results echoed back or a demo dataset. I ran three checks. First, counts: my own session returned 24 records, while two foreign sessions returned 132 and 100, and the sets did not overlap. If this were my data leaking back I would see the same 24 every time. Second, content: the `home_language` distribution in the leaked set matched the real demographics of the target district, not mine. Third, correlation: those exact counts matched what an authorized `super_admin` view showed for those districts. Three independent signals all agreeing that this is real cross tenant data.

## Why this is platform wide, not a one off

The `session_id` values were structured and predictable, not random tokens, so an attacker does not need to already know a target's id. They can enumerate. Combine that with the fact that the lowest privilege role can perform the leak and that hundreds of teacher accounts exist, and the impact is not one district reading one other district. It is any teacher account systematically harvesting student action plans across every customer on the platform: curriculum levels, reading focus areas, strategy codes, home languages, and parent email addresses. The tenant isolation that is the entire selling point of the product does not hold on this action.

## How to fix it properly

The code level fix is to give the print action the same authorization the base endpoint already has: resolve the `session_id`, verify it falls within the caller's authorized scope, and return the same denial the base endpoint returns (an empty set, a `403`, or a `404`) when it does not. But fixing one action is treating a symptom. The systemic fix is to audit every sibling of every collection, the `export`, `email`, `report`, and `download` actions included, and apply the identical tenant check to all of them, ideally through shared middleware rather than a decorator that has to be remembered per route. Then lock it in with a regression test: a low privilege role requests a foreign `session_id` on `/print` and must get the denied response.

## How to approach BOLA on any target

1. Find the working denial first. Confirm the app can say no, so you know what a blocked request looks like.
2. Enumerate the siblings of every protected endpoint: print, export, email, report, download, and any other verb hung off the same route.
3. Replay the exact request that was correctly denied against each sibling, changing nothing but the path.
4. Prove the data is genuinely foreign with counts, content, and an authorized cross reference before you report.
5. Check whether the object id is predictable. Enumerable ids turn one leak into a platform wide harvest.

The one line to remember
Authorization is not a property of an endpoint. It is a property of every action on every endpoint, and the one that leaks is almost never the one you would think to test first.

### Does every action on your API enforce tenant isolation, or just the main one?

Imperonlabs tests the print buttons, the export links, and the report actions that ship after the security review is done. Book a scoping call and we will find the sibling endpoint that skipped the check.

[Request an assessment](https://imperonlabs.com/#contact)
