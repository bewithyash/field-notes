---
title: "A Login Is Not a Permission: Broken Authorization Across a Scheduling API"
date: October 2026
source: https://blog.imperonlabs.com/blog/broken-authorization-workforce-scheduling-api.html
---

Web · API · Authorization

# A Login Is Not a Permission: Broken Authorization Across a Scheduling API

By Yash Pandya · October 2026 · 16 min read

The authentication on this app was genuinely good. The password reset was eventually hardened, the session tokens were signed and tamper proof, and I could not forge a single claim. None of that mattered. The app answered exactly one question at the door, "are you logged in?", and then let every logged-in user walk into every other team's calendar, rewrite its rules, edit its shifts, and read the whole company's staff directory. A login is not a permission. This one handed everybody the keys to everybody.

**Scope note.** This is a sanitized account of a real engagement on a workforce and shift-scheduling SaaS. I have renamed the product to **ShiftGrid**, put it on `shifts.example.com`, and replaced the vendor, the staff, and the identifying route strings. The HTTP mechanics, status codes, severities, and weakness classes are reproduced exactly as I observed them.

ShiftGrid is the kind of product that runs the back of a hospital, a call center, or a retail chain. Managers build weekly schedules, set coverage rules ("I need at least three people on the floor Saturday morning"), drop shifts onto a grid, and employees look up when they work. It is a boring, load-bearing piece of software, which is exactly the kind I enjoy, because the interesting failures in boring software are almost never in the crypto. They are in the question the app forgets to ask, and this one forgot a big one.

I want to be fair to the team before I take the app apart, because the context is the point. I threw a full authentication battery at this target and it held. The tokens were signed bearer tokens carrying a subject, a tenant identifier, and a permission array. Algorithm confusion, a stripped signature, `alg:none`, an injected `kid` and a remote `jku`, a tampered tenant identifier with the original signature kept: every one came back `401`. Signature integrity held and the cross-tenant boundary was a wall. So this is not a story about a broken token. It is a story about an app that validated the token perfectly and then trusted it to mean something it did not mean.

## The target: one backend, many teams

The data model is the thing to hold in your head, because every finding is a consequence of it. One tenant (think of it as one company) contains many schedules. A schedule belongs to a team or a department. Inside a schedule you have coverage requirements, shift definitions, employee rows, and assignments. People have roles and permissions, and crucially, people have a relationship to a schedule or they do not. A manager *manages* some schedules. An employee is a *member* of the schedules they work on. Most employees are related to very few schedules, and related to none of the rest.

For the engagement I worked with three identities inside a single tenant, so nothing I did ever crossed a company boundary. The victim object owner was an administrator account with the full permission set, the manager of a particular schedule I will call schedule 2144. Against it I pointed two attacker identities:

- A **low-privilege** user holding two narrow permissions, roughly "view my own schedule" and "edit my own profile". It managed no schedules and was not a member of schedule 2144. In every meaningful sense it was a rank and file employee.
- A **zero-permission** user whose token carried no permission claim at all. A valid login with, on paper, the right to do nothing but exist.

The whole test reduces to one sentence. Can an identity that has no relationship to schedule 2144, and in one case no permissions whatsoever, nonetheless read it, change it, export it, and enumerate the people behind it? The answer, five different ways, was yes.

## BFLA and BOLA, and why the difference matters

Two classes of broken authorization run through this report, and it is worth being precise about them because the fixes are different. People lump them together as "access control bugs" and then write a fix for one that does nothing for the other.

**Broken Function Level Authorization** (BFLA, OWASP API5) is about the *action*. Some operations are supposed to be reserved for a role or a permission. Creating a coverage rule, editing a shift definition, and reading the entire staff directory are manager and administrator functions. BFLA is when a user who lacks that function level can invoke it anyway. The telling signature is the zero-permission account succeeding, because that account is not supposed to be able to perform privileged *actions* on anything at all.

**Broken Object Level Authorization** (BOLA, OWASP API1), also called IDOR, is about the *object*. The user may well be allowed to perform the action in general, on their own data. The failure is that the server does not check whether *this specific object*, named by an identifier in the request, actually belongs to them. The telling signature is that you change a number in the URL and get back somebody else's record.

The clean way to keep them apart: BFLA asks "are you allowed to do this kind of thing?" and BOLA asks "are you allowed to do it to *this*?" A hardened app answers both on every request, at the data layer, not in the menu that decided which buttons to render. ShiftGrid answered neither consistently. I will flag which class each finding belongs to as I go.

## Finding 1: anyone could rewrite any team's coverage rules

Coverage requirements are the rules a schedule has to satisfy, the "three people on Saturday morning" constraints. They are the skeleton a manager builds a week around. They are created, edited, and deleted through the schedule they belong to. This is a manager function on a specific object, so a clean implementation has to pass two checks: is the caller a manager, and does the caller manage *this* schedule.

I replayed the manager's own create request verbatim, but signed with the low-privilege employee's token. The employee managed nothing and had no relationship to schedule 2144.

```
POST /api/schedules/2144/coverage HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>
Content-Type: application/json

{"DayOfWeekOrDate": "2026-12-15"}

HTTP/1.1 201 Created
Content-Type: application/json

{"id": 154, "scheduleId": 2144, "date": "2026-12-15",
 "required": 0, "shiftDefinitionId": null, "alertEnabled": true}
```

`201 Created`. An employee with no relationship to this schedule had just minted a coverage rule inside it and been handed the new object identifier, 154. That turned the next two verbs into a formality. I pointed a modify and a delete at the id I had just created:

```
PUT /api/coverage/154 HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>
Content-Type: application/json

{"Required": 99, "AlertEnabled": false, "ShiftDefinitionId": 269465}

HTTP/1.1 200 OK

DELETE /api/coverage/154 HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>

HTTP/1.1 200 OK
```

Both returned `200` and both persisted. Full create, modify, and delete over another team's coverage rules, from an account that should not have been able to see the schedule at all. This one is a hybrid: BFLA because a non-manager performed a manager-only action, and BOLA because it happened on a schedule the account had no relationship to. The server checked the token was valid, found a route, and ran the handler. That is the whole decision it made.

One detail matters for how you read the next findings. When I sent a malformed body, the server answered `400` with a validation error. A `400` is not an authorization decision, it is the server telling you it parsed your request and did not like the shape. The moment I sent a well-formed body, I got my `201`. Nowhere in that flow did the server say `403`. It never once asked whether I was allowed, and that absence is the finding.

## Finding 2: editing and deleting shifts I did not own

Shift definitions are where the money is, literally. A shift carries a start, a stop, a break, and a computed count of paid hours, and paid hours flow downstream into payroll and cost reports. The write surface is the obvious two verbs, `PUT /api/shifts/{id}` to modify and `DELETE /api/shifts/{id}` to remove. Both are manager functions on a specific schedule's object.

I took a throwaway shift that the administrator owned on schedule 2144 and sent a modify as the low-privilege employee. I deliberately made the payload nasty, to show the blast radius rather than just prove the gate was open. I renamed the shift, set the break to a negative number, and claimed eleven paid hours:

```
PUT /api/shifts/321764 HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>
Content-Type: application/json

{"id": 321764, "scheduleId": 2144, "name": "EDITED_BY_OUTSIDER",
 "abbrev": "RV", "start": "08:00", "stop": "17:00",
 "breakMinutes": -120, "paidHours": 11, "isTimeOff": true, "active": true}

HTTP/1.1 200 OK
```

`200`, persisted. Then I sent the exact same request again, this time signed with the **zero-permission** token, the account whose permission array was empty:

```
PUT /api/shifts/321764 HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <zero-permission token>
Content-Type: application/json

{ ... same body ... }

HTTP/1.1 200 OK
```

Also `200`, also persisted. This is the cleanest BFLA proof in the engagement: an account that holds no permissions at all rewrote a payroll bearing shift on a schedule it had never seen. There is no object relationship to debate and no role to quibble over. The function simply did not check that the caller was entitled to invoke it. A negative break time inflating the paid hours is not a creative exploit, it is just what happens when a write endpoint trusts whatever a client sends and never asks who the client is.

The delete was the same story. As the low-privilege employee:

```
DELETE /api/shifts/321764 HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>

HTTP/1.1 200 OK
```

The delete was a soft delete, and there was no API path to restore it. So any authenticated user could quietly remove shift definitions from any schedule in the tenant, and the people whose schedules they were could not undo it through the product. On a system that tells a hospital floor who is working tonight, that is not vandalism, that is an operational incident.

As a control I sent a `PUT` with no token at all, and got `401`. That is the tell. The app enforced *authentication* on this route, the `401` proves it. It simply never got around to enforcing *authorization*. Those are two different gates, and only one of them was installed.

## Finding 3: exporting a roster by guessing a number

Every scheduling product has an export button, because managers live in spreadsheets. ShiftGrid exposed it as `GET /api/schedules/{id}/export` with a format parameter, and it returned the whole roster for a schedule, every person on it with their names and employee identifiers, as CSV or as a PDF.

The identifier in that URL is the only thing standing between a user and a schedule's roster. So I asked the obvious question, as the low-privilege employee who was not a member of schedule 2144, and simply requested the export:

```
GET /api/schedules/2144/export?format=csv HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <low-privilege employee token>

HTTP/1.1 200 OK
Content-Type: text/csv

(the full multi-employee roster: names and employee ids)
```

`200`, with the entire team's roster in the body. I switched the format to `pdf` and received a formatted document of about 27 KB, the same roster, suitable for printing and handing around. This is textbook BOLA, the OWASP API1 case and CWE-639: authorization bypass through a user controlled key. The function (export) was fine for a user to call on their own schedules. The server just never verified that schedule 2144 was one of them. Change the number, get a different team's roster, and there are only so many numbers. People underrate this in triage because "it is only a read", but it is a read of other people's names and staff identifiers at scale, enumerable by walking an integer. An export endpoint is a database query wearing a nicer hat, and it needs the same object check the database would want.

## Finding 4: the whole staff directory, to anyone with a token

The directory endpoint, `GET /api/employees`, is not scoped to an object at all. It returns the tenant's people. That makes it a pure function level question: who is allowed to enumerate the workforce? The intended answer is managers and administrators. The actual answer was anyone who could log in.

As the low-privilege employee, and again as the zero-permission account, I called it:

```
GET /api/employees HTTP/1.1
Host: shifts.example.com
Authorization: Bearer <any authenticated token>

HTTP/1.1 200 OK
Content-Type: application/json

[ { "id": 27702, "fullName": "...", "email": "...",
    "departmentId": 1039, "locationId": 872, "positionId": 1551,
    "supervisorEmployeeId": 26203, "userId": 17751,
    "canLogin": true, "isAdmin": false, "hireDate": null, ... },
  ... 22 records ... ]
```

`200`, with the full directory. Twenty-two records in this tenant, and each one was not a name and nothing else. Each record carried work contact details, the department, location, and position identifiers, the supervisor chain through `supervisorEmployeeId`, the linkage between an employee and a login account through `userId` and `canLogin`, and an `isAdmin` flag that told me, for free, exactly which accounts to care about. A zero-permission user could pull the org chart and a map of which accounts were administrators. That is reconnaissance gold, and it is a privacy breach in its own right regardless of what you do next with it.

This is BFLA, OWASP API5, CWE-862, because the function has no gate at all for the caller's role. The directory is not a single object you own or do not own, it is a privileged list, and listing it is a privileged action. The app treated authentication as sufficient authorization for it.

## Finding 5: reading a schedule I was not a member of

The last one is the most instructive, because it sat right next to a control that actually worked. A schedule exposes several read sub-resources: `/detail` for its metadata, `/grid` for the calendar view, `/rows` for the per-employee rows, and `/shifts` for its shift definitions. As the low-privilege employee, a non-member of schedule 2144, I asked for all four:

```
GET /api/schedules/2144/detail HTTP/1.1          -> 200 (full content)
GET /api/schedules/2144/grid?start=2026-11-16&days=7  -> 200 (full content)
GET /api/schedules/2144/rows HTTP/1.1            -> 200 (full content)
GET /api/schedules/2144/shifts HTTP/1.1          -> 200 (full content)
(Authorization: Bearer <low-privilege employee token> on each)
```

All four returned `200` with the schedule's full content: its name, the employee rows including an administrator's employee identifier, the shift definitions, and the assignments. A non-member read a schedule end to end. That is BOLA again, object level, because membership of schedule 2144 was the relationship the server failed to check.

Now the instructive part. There was a *fifth* sub-resource on the same schedule, `/api/schedules/2144/employees`, and when I requested it as the same non-member I got back `401` with a message to the effect of "manager role required". Somebody had remembered to guard that one route. One sibling under the same parent object was gated, and four were wide open. That is not a policy, that is a checklist that someone did not finish. It is the single clearest piece of evidence that authorization on this app was applied per handler, by hand, and whoever wrote the handlers simply forgot four of them.

The guard was at the front door, not on the rooms

Login gate
valid token?
enforced
401 without a token,
every route.

/detail -> 200 open

/grid -> 200 open

/rows -> 200 open

/shifts -> 200 open

/employees -> 401 gated
One sibling was guarded.
The gate was per handler,
and four were forgotten.

Authentication was enforced everywhere. Object and function authorization were enforced on one route out of five under the same parent. That inconsistency is the fingerprint of per-handler access control.

## The pattern connecting them

Step back from the five and the shape is obvious. Every request passed through a real authentication filter, which is why a missing or invalid token reliably produced `401`. After that filter, the request landed in a handler that ran business logic against the data layer with almost nothing in between. Authorization, where it existed at all, was a line of code inside an individual handler that somebody remembered to write. Where it did not exist, the handler just did the work.

That is authorization enforced in the wrong place. The real gate on this product was the user interface: the portal only rendered the buttons an employee was allowed to press, and the API assumed nobody would ever send a request the UI had not offered. But the UI is a suggestion. The API is the contract, and it is the only thing an attacker talks to. A control that lives in the client, or in the menu that decides which routes get called, is a decoration. The `/employees` sub-resource being gated while its four siblings were open is the proof that the real checks were applied by hand, one handler at a time. Humans forget handlers. Policies do not. That is why this belongs in middleware, not in a handler.

## What this actually costs you

It is easy to file these as "info leak" and "access control" and move on. Here is what they mean in the language of the business running on the software.

**Staff privacy.** The directory finding alone exposes every employee's name, work contact, place in the org chart, supervisor, and whether they are an administrator, to anyone who can authenticate, including an account with zero permissions. The export finding does the same for rosters, enumerable one schedule identifier at a time. This is a data protection problem before it is anything else.

**Schedule integrity and payroll.** The shift write finding lets any authenticated user change paid hours on any schedule, and paid hours feed payroll and cost reporting. Inflating them is fraud with a quiet, plausible paper trail. The coverage write finding lets anyone rewrite the rules a schedule is validated against, so you can make an understaffed week look compliant or a compliant week look broken.

**Operational disruption.** The shift delete is a soft delete with no product level restore. Any authenticated user can remove shift definitions from any team, and the team cannot put them back through the app. On a system whose entire job is to tell people when and where to show up, silent deletion of schedule content is not a theoretical risk, it is the outage.

Every one of these is reachable by a single rank and file login, and several by a login with no permissions at all. There is no privilege to escalate and no exploit chain to build. The cost of entry is one valid account, and in a tenant with dozens or thousands of employees, that is not a scarce resource.

## Mapping it: OWASP API1 and API5, CWE-639, CWE-862, CWE-863

For the report, and for anyone triaging similar findings, here is how the five land against the standard references.

- **Export IDOR and non-member schedule read** are **OWASP API1:2023, Broken Object Level Authorization**. The server trusts an object identifier in the request without checking the caller's relationship to that object. The closest CWE is **CWE-639, Authorization Bypass Through User-Controlled Key**.
- **Shift write, coverage write, and the directory read** are **OWASP API5:2023, Broken Function Level Authorization**. A caller invokes a privileged function their role or permission set does not entitle them to. Where the function has no check at all, as with the directory and the open shift routes, that is **CWE-862, Missing Authorization**. Where a check exists but reaches the wrong conclusion, that is **CWE-863, Incorrect Authorization**.
- The coverage finding is a hybrid of both classes, because a non-manager performed a manager function (API5) against a schedule it had no relationship to (API1).

All five are **High severity** broken authorization issues. None requires more than one authenticated identity, and the function level cases do not even require a permission: network reachable, low complexity, low or no privileges, no user interaction, with real impact on confidentiality and, for the write cases, integrity and availability.

## How to test authorization without guessing

This class of bug does not show up in a scanner report, and it does not show up by clicking around as one user, because as one user everything looks fine. You find it by being deliberate, with a method that is boring on purpose.

**Build a role matrix first.** Provision one real account at each tier the product sells: an administrator, a manager, a plain employee, and, the one people skip, an account with no permissions at all. The zero-permission account is your sharpest instrument, because anything it can do is unambiguously a function level failure. Write down, for each sensitive operation, who is supposed to be able to perform it. That table is your oracle.

**Do two-account object swaps.** For every object the app creates, have account A create it and capture its identifier, then try to read, modify, and delete that exact identifier using account B's token, where B has no relationship to the object. A proper server answers `403` before it touches the data. If you get `200`, or a `201` like I did on the coverage create, you have BOLA. Do it for every verb, not just the one the UI offers. Developers routinely guard the read and forget the delete.

**Tamper the method, not just the identifier.** A route can be gated for `GET` and open for `PUT`, or the reverse, so walk every verb a route accepts. The sibling-route lesson applies here too: if `/thing/detail` is gated, do not assume `/thing/grid` and `/thing/rows` are. Per-handler authorization fails per handler.

**Learn the difference between a 400, a 401, and a 403.** A `400` means the server parsed you and did not like the body, it is not an access decision. A `401` means authentication is missing or invalid. A `403` is the only response that proves the server asked whether you were allowed and decided you were not. If you can get a sensitive operation to return `200` or `201` and you never saw a `403` on the way, the authorization check is not merely wrong, it is absent. Confirm writes actually persisted, too, by reading the object back as the owner. An echo is not a change.

## Root cause

The root cause is a single architectural decision made by omission: authorization was never given a home. Authentication had one, a filter that ran on every request and reliably produced `401`. Authorization was left to the individual handlers and to the user interface, which meant it existed wherever a developer happened to remember it and was absent everywhere else. The tenant isolation held, probably because it was baked into the data access layer and applied to every query, which is exactly the pattern the rest of the authorization needed and did not get.

There is no villain here and no exotic bug. It is the most common serious finding in modern APIs because the framework will not catch it for you. A SQL injection or an XSS has a sink you can scan for. A missing authorization check is the absence of code, and absence does not light up a linter. You find it only by asking, on every endpoint, the question the app forgot to ask.

## How to fix it

The fixes stack, and the order matters. Doing only the last one leaves you exactly as exposed as before.

**Centralize authorization.** Move the decision out of the handlers and into a policy layer the framework evaluates before the handler runs, the same way authentication already works. Every mainstream stack has a first class way to do this. The goal is that reaching a handler at all is proof that authorization already passed, so a developer writing a new endpoint gets the check by default instead of having to remember it.

**Enforce object ownership and membership at the data layer.** For anything addressed by an identifier, the server must resolve the caller's relationship to that object from the signed token and confirm it before acting. "Is this schedule one the caller manages or is a member of" has to be a query the code actually runs, not an assumption inherited from the UI. The tenant scoping already did this correctly for cross-tenant isolation. Extend the same discipline down to the schedule and object level.

**Deny by default.** The policy's resting state should be denial. A route with no explicit policy should refuse, loudly, rather than fall through to the handler. If the directory and the four open schedule routes had lived in a deny-by-default world, forgetting to annotate them would have produced a `403` and a bug report, not a breach. Make the forgetful case the safe case.

**Check every verb.** Authorization on the read does not cover the write, and a guard on create does not cover update or delete. Apply the policy to the whole surface of a resource, and add regression tests that drive a non-member and a zero-permission account at every verb of every sensitive route and assert `403`. The sibling that was gated while its four siblings were open is the exact failure a per-resource policy plus these tests would have caught before it ever shipped.

**At a glance.**

- **Coverage write BFLA and BOLA** on `POST /api/schedules/{id}/coverage` and `PUT` or `DELETE /api/coverage/{id}`. High. OWASP API5 and API1. CWE-862 and CWE-863.
- **Shift write BFLA** on `PUT` and `DELETE /api/shifts/{id}`, succeeding even for a zero-permission account. High. OWASP API5. CWE-862.
- **Schedule export IDOR** on `GET /api/schedules/{id}/export`. High. OWASP API1. CWE-639.
- **Directory read BFLA** on `GET /api/employees`. High. OWASP API5. CWE-862.
- **Non-member schedule read BOLA** on `GET /api/schedules/{id}/detail`, `/grid`, `/rows`, and `/shifts`. High. OWASP API1. CWE-639.

The one line to remember
Authentication proves who is knocking. Authorization decides which doors open, and it has to decide on the server, on every request, for the action and for the object. If your API never returns a 403 on a sensitive operation, it is not strict. It is not asking.

### Does your API check what the caller is allowed to touch, or only that they logged in?

Imperonlabs provisions a real account at every role, including one with no permissions, and drives your endpoints the way an insider would: every object, every verb, every sibling route. We find the checks that live in the UI instead of the server, before someone else does. Book a scoping call and we will map where your authorization only looks like authorization.

[Request an assessment](https://imperonlabs.com/#contact)
