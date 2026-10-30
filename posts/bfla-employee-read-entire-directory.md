---
title: "BFLA: How a Read-Only Employee Account Exposed an Entire Workforce"
date: August 2026
source: https://blog.imperonlabs.com/blog/bfla-employee-read-entire-directory.html
---

Web · Access Control

# BFLA: How a Read-Only Employee Account Exposed an Entire Workforce

By Imperonlabs Research · August 2026 · 7 min read

The developers thought carefully about who was allowed to change data. They forgot to think about who was allowed to see it. From the lowest-privileged account the platform issues, a single collection endpoint returned the complete record of every person in the company. This is broken function level authorization, and it lived entirely on the read path.

Scope note. This is a generalized account of a real assessment. The target was a SaaS workforce and scheduling platform. The company name, hostnames and record contents are withheld, and every value shown is illustrative.

## The setup

The platform does what workforce software does. It manages staff, schedules and pay, and it ships with the usual role hierarchy: administrator, manager, employee. The interface behaves correctly. Sign in as an ordinary employee and the UI shows you your own profile and your own schedule, nothing more. As far as the screen is concerned, an employee is boxed into their own row.

That boxed-in feeling is exactly what makes this class of bug survive. The front end enforces the boundary so convincingly that nobody goes looking for it on the wire. We did.

## The finding

Authenticated as a standard employee, the account with the least privilege the system grants, we watched the traffic behind the profile page. Two endpoints stood out:

```
GET /api/employees/{id}    # a single employee record
GET /api/employees         # the collection
```

The single-record endpoint was interesting, but the collection was the whole ballgame. A plain request to `/api/employees`, sent with nothing more than an ordinary employee session, returned the full organization directory. Not a filtered list. Not a set of public fields. Every colleague's complete record: home addresses, personal phone numbers, dates of birth and hourly pay rates.

Write operations were locked down properly. Try to modify another employee and the server said no. The team had built authorization, tested it, and shipped it. They had just built it on one side of the door. Reads walked straight through.

Two doors, one lock

Write path · PUT / POST / DELETE
Caller role is checked against the target record. Employee editing
a colleague is rejected. This side was tested and it holds.

Read path · GET /api/employees
No caller check at all. The handler returns the entire collection to
anyone with a valid session. This side was never questioned.

Read and write are separate authorization decisions. Get one right and the other wrong, and the app looks secure while it leaks.

## Why scanners walk right past it

An automated scanner has nothing to grab onto here. The endpoint answers `200 OK` with well-formed JSON. There is no injection, no error, no stack trace, no timing anomaly, no reflected input. From a machine's point of view the response is indistinguishable from a legitimate one, because it is a legitimate response, sent to the wrong person. A scanner flags what looks broken. Authorization gaps on read paths do not look broken. They look like a working API.

This is why the finding needs a human with a low-privilege account and the patience to ask, for every list the app renders, who is actually allowed to receive this. The bug is not in the bytes. It is in the mismatch between the caller and the contents.

## Impact

Any employee, from the day they were onboarded, could exfiltrate the personal and financial data of the entire workforce with one request. That is mass PII disclosure and compensation disclosure in a single call: the kind of exposure that turns into an internal trust problem, a regulatory problem, and a headline, in that order. Severity: High. The access required to trigger it was the lowest the platform hands out, which is precisely what makes it worse.

## How to fix it

The fix is not a patch on one endpoint, it is a change in how the team reasons about handlers.

- Filter collection endpoints by the caller's authorization. A list must return only the records that caller is entitled to see, scoped server-side, never trimmed only in the UI.
- On single-record reads, verify the caller's permission for that specific record before returning it, the same way you already do for writes.
- Treat read and write as two independent authorization decisions. Passing one says nothing about the other. Every GET deserves the same scrutiny you gave every PUT.

## The lesson

When you test access control, do it from the least-privileged account you can get, and spend as much time on reads as on writes. The instinct is to hammer the state-changing calls, because those feel dangerous. But the quiet `GET` that returns a collection is where broken function level authorization loves to hide, and it hides best in the most ordinary employee-facing screens, the ones everyone assumes are already boxed in.

The one line to remember
Write authorization and read authorization are different locks. An app that guards one and forgets the other will look secure right up until an intern downloads the whole company.

### Does every list your API returns know who is allowed to read it?

Imperonlabs tests access control from the lowest-privileged account up, on the read paths scanners never flag. Book a scoping call and we will show you what your ordinary users can already see.

[Request an assessment](https://imperonlabs.com/#contact)
