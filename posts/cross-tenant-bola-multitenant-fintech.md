---
title: "The Server Knew Who I Was, Not Whose Data It Was: Cross-Tenant BOLA in a Payroll Fintech"
date: October 2026
source: https://blog.imperonlabs.com/blog/cross-tenant-bola-multitenant-fintech.html
---

API · BOLA

# The Server Knew Who I Was. It Never Asked Whose Data It Was.

By Yash Pandya · October 2026 · 14 min read

The login worked. The token was signed, unexpired, and mine. Every request I sent carried it, and the server validated it on the way in, every single time. The platform was certain who I was. What it never once checked was whether the record I asked for had anything to do with me. So I asked for someone else's, from a different company entirely, and it handed it over with a cheerful `200 OK`. This is the story of a bug that lives in the one-character gap between knowing you and trusting you.

**Scope note.** This is a sanitized account of a real engagement on a multi-tenant salary-advance and payroll fintech. The product is called **SalarySprint** here and lives at `pay.example.com`; both are invented stand-ins. Tenant names, identifiers, routes and field values are examples. The mechanics, HTTP behaviour, CWE mapping and severity are reproduced exactly from the report. Nothing here points at a live system.

SalarySprint is an earned-wage-access platform. Employers sign up as tenants, onboard their staff, and those staff request advances against salary they have already earned. Each advance moves through a short pipeline: a request is raised, a wallet balance is checked, credit is extended, and a bank payout is scheduled against the employee's account. The data that rides along is exactly the data you would least want to leak. Full legal names, dates of birth, bank account numbers, monthly salary broken down into its components, and a running ledger of who borrowed how much and when. It is regulated financial personal data, and the whole value proposition of the company rests on one promise that is never written on the marketing site: your company's numbers stay inside your company.

That promise is an authorization promise. And authorization, it turns out, was the one thing this API had quietly decided was optional.

## The target, and the shape of a multi-tenant system

A multi-tenant application is one backend serving many customers who must never see each other. The physical reality is a shared database, shared code, and shared infrastructure. The logical reality has to be a set of hard walls between tenants, enforced in software, because there is no wall in the hardware. Every table has a `tenant_id` or a `company_id` column doing the work that, in a single-tenant world, a separate database would have done for free. The isolation is entirely a property of your queries. If a query forgets to filter by tenant, there is nothing underneath to catch it. The row is right there, and the database will happily return it.

SalarySprint had this structure in its purest form. Tenants were companies. A company had employees, and employees had advance requests, wallets, credit records and payout details. Every one of those objects carried a `companyId` that said which tenant it belonged to. The API was a clean set of REST endpoints: a search endpoint per object type that took a filter and returned a page of results, and a by-id endpoint that returned a single object. In the abstract, a textbook design. The question an attacker brings to a textbook design is always the same. Where, precisely, does the tenant wall get enforced, and what happens if I lean on it?

I was given a client-provided administrator account to start, which is the normal way these engagements open. But an admin account is the wrong tool for testing isolation, because an admin is often supposed to see across scopes, so a cross-tenant read from an admin proves nothing. The only honest way to test a tenant boundary is to stand fully inside one tenant, with the lowest privilege the system offers, and then try to reach out. So the first thing I did was create a disposable, low-privilege HR-role account and bind it to a single tenant I will call **Acme Staffing**. Acme was a brand new company I had just made. It had no employees of its own worth speaking of and, critically, zero advance requests. Any advance data that account could see was, by definition, data that did not belong to it.

## Authentication is not authorization, and the difference is the whole bug

These two words get used as if they were one, and the conflation is responsible for a genuinely enormous share of real-world breaches, so it is worth being pedantic about them for a moment.

**Authentication** answers one question: who are you? It is the login, the password, the one-time code, the signed token the server hands back and that your client presents on every subsequent request. When authentication works, the server can say with confidence, this request is coming from the account belonging to this person. That is all it says. It is a statement about identity and nothing else.

**Authorization** answers a different question, and really two of them: are you allowed to perform this action, and are you allowed to touch this specific object? The first of those is function-level authorization, sometimes called role-based access control. Can an HR user call the admin-only endpoint that creates companies? The second is object-level authorization. You are allowed to read advance requests in general, fine, but is this particular advance request one of yours? The industry shorthand for a failure of that second check is Broken Object Level Authorization, or BOLA, and its older web-era name is Insecure Direct Object Reference, or IDOR. Same bug, two decades of two names.

The reason the distinction matters so much here is that SalarySprint got one half of authorization completely right and the other half completely wrong, and that split is what made the finding so clean. My low-privilege HR account was correctly refused every admin-only action. When I tried to create a company, add an account, extend credit, or hit any dashboard endpoint, the server came back with `401` and a polite note that I did not have permission. Function-level authorization was present, enforced, and working. If the whole system had been broken, the finding would have been noisy and arguable. Instead exactly one layer was missing, which made it precise and impossible to wave away. The server knew my role and respected its limits. It just never connected my identity to the ownership of the data I was requesting.

The distinction in one line
Authentication proves you are a valid user. Authorization proves this valid user may touch this exact object. A system can do the first flawlessly and skip the second entirely, and from the outside the login looks perfect right up until the data walks out.

## Finding the object identifier, and the two-account test

Object-level bugs begin with object identifiers, so the first job is to find out what the application uses to name a thing and whether that name is guessable, enumerable, or simply handed to you. I watched the traffic my own account generated and catalogued every identifier the client sent to the server. There were two shapes that mattered.

The first was a scope parameter on the search endpoints. Every list request carried a `tenantIds` array in its body, the set of companies whose records to return. The client was sending the scope. Read that sentence again, because it is the whole vulnerability in one observation. The thing that decides which tenant's data comes back was a value the client got to choose. The second shape was a plain object identifier on the by-id endpoints, a GUID in the path, as in `GET /api/v1/advances/{id}` and `GET /api/v1/users/{userId}`.

The only reliable way to test isolation is with two accounts and a swap. You need an account that owns a thing, so you know a real valid identifier, and a second account that owns nothing, so that any success is unambiguous. I already had Acme Staffing, my owns-nothing account. For the other side I used a separate tenant I will call **Globex Logistics**, which had real employees and a handful of genuine advance requests. The experiment writes itself. Take an identifier that belongs to Globex. Present it on a request authenticated as Acme. See what comes back. If the wall is real, the correct answer is an empty result or a `403 Forbidden`, because Acme has no business naming a Globex object. If the wall is missing, the data comes back, and the fact that the token belongs to a tenant that provably does not own the record removes every other explanation.

There is a quieter, even more damning variant to try first. On the search endpoint, instead of naming another tenant, I removed the scope entirely. I sent the `tenantIds` array empty. In a system that derives scope from the caller, an empty client-supplied scope should mean either nothing or, at most, your own things. In a system that trusts the client's scope and treats empty as no filter, it means everything. That is the test that tells you, in a single request, whether scope is a server decision or a client suggestion.

## The request that crossed the boundary

Here is the empty-scope search, authenticated as the HR account that belongs only to Acme and owns no advances at all. Values are sanitized and the host is an example.

```
# Acme owns zero advances. An empty tenant scope should return nothing.
curl -s -X POST 'https://pay.example.com/api/v1/advances/search' \
  -H 'Authorization: Bearer <token-for-HR-user-of-Acme>' \
  -H 'Content-Type: application/json' \
  --data '{"page":1,"size":50,"filters":[],"sort":{"field":"createdOn","order":"desc"},"tenantIds":[]}'

200 OK
{ "count": 48, "items": [ /* every tenant's advance requests */ ] }
```

Forty-eight records. Acme owns none of them. The empty array was read as no filter, and no filter across a shared table means the whole table. That is the entire platform's advance pipeline, returned to an account that created it minutes ago and owns nothing in it.

To make the cross-tenant claim airtight rather than merely alarming, I repeated it with explicit scopes, still as the HR user of Acme.

```
# Naming Acme's own tenant id, as Acme:
  "tenantIds": ["<acme-tenant-id>"]     → 200  count 0    (Acme really does own nothing)

# Naming Globex's tenant id, as Acme:
  "tenantIds": ["<globex-tenant-id>"]   → 200  count 2    (leaked)
    {
      "requestNumber": "ADV-0043",
      "iban":          "XX00 0000 0000 0000 0000 00",
      "userName":      "Example Employee of Globex",
      "tenantId":      "<globex-tenant-id>"
    }
```

Account A, authenticated and scoped to nothing, naming Account B's tenant and receiving Account B's financial records. Zero when I asked for my own, because I genuinely have none. Two when I asked for someone else's. There is no ambiguity left in that pair of responses. The server is scoping by the number I typed, not by the person I am.

The by-id endpoints told the same story in the most direct possible form, no filter semantics to argue about, just a single object fetched by its name.

```
# Read one of Globex's advance objects directly, as Acme:
GET /api/v1/advances/<globex-advance-id>
Authorization: Bearer <token-for-HR-user-of-Acme>
→ 200 OK, full record

# Read a Globex employee's account by user id, as Acme:
GET /api/v1/users/<globex-employee-id>
Authorization: Bearer <token-for-HR-user-of-Acme>
→ 200 OK  { name, email, iban, salary breakdown, dateOfBirth }
```

A valid token for one tenant, a known identifier for another, and the full employee record of a stranger's employee, name, email, bank account, and salary down to the component, returned without a second thought. The same pattern held across wallets and credit records. Every object type exposed the identical missing check, because they all shared the same data-access path.

One backend, two tenants, and a wall that was never built

Tenant A (Acme)
valid token, owns nothing

API checks the token
identity: confirmed

Data layer runs the query
filters by the id you sent

Tenant B rows (Globex)
IBANs, salaries, PII

Returned to Tenant A
ownership never checked

Missing step: intersect every query with the caller's own tenants
deny by default, derive scope from the token, not the body

The token is checked. The role is checked. The one check that never runs is whether the row the query is about to return belongs to the caller's tenant.

## Read versus write, and why the read is already a catastrophe

Everything above is a read. A read-only BOLA is sometimes dismissed as an information disclosure, which undersells it badly, and in a fintech it is close to the worst thing you can lose. What leaked was not a marketing newsletter. It was bank account numbers tied to named individuals, full salaries, dates of birth, and a complete map of who at every customer company is borrowing against their pay. That is identity-theft-grade material on one side and commercially devastating intelligence on the other. A competitor who wanted to know a rival's exact headcount, payroll cost, and financial stress could read it straight out of the API with a free trial account. For a platform whose single promise is tenant isolation, a total read-side isolation failure is not a lesser bug. It is the product failing at the one thing it exists to do.

Write is the darker cousin, and the same root cause produces it the moment a mutating endpoint shares the missing check. The identical pattern, authentication present and ownership absent, showed up on a state-changing endpoint elsewhere in the same assessment: a user in one tenant could trigger a password reset against a user in another tenant and the server accepted it with a `200`. When ownership is not enforced on the object you are modifying, BOLA stops being about reading secrets and becomes about rewriting other people's state. Read and write BOLA are the same hole seen from two sides. If the data-access layer does not know who owns a row on the way out, it does not know on the way in either, and whether a given endpoint leaks or corrupts is mostly an accident of which verb the developer wired to the unguarded query.

## Why this is BOLA: OWASP API1:2023, CWE-639 and CWE-863

This finding sits at the top of the OWASP API Security Top 10 as **API1:2023 Broken Object Level Authorization**, and it is first on that list for a reason. Object-level authorization is the single most common and most impactful API vulnerability in the wild, precisely because it is invisible to every test that uses one account. Your own data looks correct. The login works. The role checks work. The bug only appears when you ask whether the object you just received was ever yours to receive, and that is a question most test suites, and most code reviews, simply never pose.

Two CWE identifiers describe the mechanism with more precision. **CWE-639, Authorization Bypass Through User-Controlled Key**, is the exact shape of the search and by-id behaviour here: the key that selects the object, the `tenantIds` value and the GUID in the path, is user-controlled, and the server uses it to fetch without verifying the caller's relationship to what it selects. **CWE-863, Incorrect Authorization**, is the broader statement that an authorization check that should happen does not, or happens wrongly. Both are accurate. CWE-639 names the lever, CWE-863 names the missing lock.

On severity, I will use exactly what the report established rather than reverse a vector to fit a conclusion. The finding was rated **Critical**, CVSS 8.1, with the vector `AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:L/A:N`. That reads cleanly against the facts. It is reachable over the network with no special conditions, it needs only the low privilege of any ordinary authenticated tenant user, and it needs no victim interaction. The scope is changed, because a flaw in one tenant's security authority grants access to resources governed by another tenant's authority, which is the textbook definition of a scope change and is what makes a cross-tenant isolation break categorically worse than a same-tenant one. Confidentiality impact is high, because complete regulated financial PII for the entire platform is exposed. Integrity is low and availability none, reflecting that the primary demonstrated path here was a read; raise integrity the moment a write-side sibling like the cross-tenant reset is in scope. The number is not the point. The point is that the lowest-privilege account the system can mint reads the highest-sensitivity data every other customer holds, and no amount of CVSS arithmetic makes that anything other than a priority-one fix.

## How to test for this class systematically

BOLA hides from careless testing and surrenders instantly to disciplined testing. The discipline is a small, repeatable routine, and it is worth running against every object-bearing endpoint rather than spot-checking one or two.

1. **Always use two accounts.** One that owns real objects, so you have valid identifiers to replay, and one that owns nothing, so any success is unambiguous. A single account can never prove isolation, because you cannot tell a real permission from a missing check when every object you see is legitimately yours.
2. **Catalogue every identifier the client sends.** Path GUIDs, query parameters, and scope arrays in the body all count. Any value the client supplies that influences which object the server returns is a candidate key for BOLA. The scope array is the one people miss, because it does not look like an object reference, but a client-supplied scope is the purest form of this bug.
3. **Swap the id and diff the responses across accounts.** Present account B's identifier on a request authenticated as account A. A correct system answers `403` or an empty set. A broken one answers `200` with B's data. Diffing A-asking-for-B against B-asking-for-B tells you in one comparison whether the server scopes by caller or by input.
4. **Probe the empty and the wildcard scope.** Send the scope array empty, send it with every id you can discover, send it with ids you are guessing at. An empty scope that returns everything is a loud confession that scope is a client suggestion, not a server decision.
5. **Enumerate where identifiers are sequential or discoverable.** GUIDs slow enumeration but rarely stop it, because ids leak through other responses, exports, and references between objects. Treat any id you can obtain from one response as a key to try on every endpoint.
6. **Separate the two authorization questions deliberately.** Confirm function-level access control independently from object-level access control. A system can pass one and fail the other, as this one did, and testing them together hides exactly that split.

## Root cause: the ownership filter the data layer never applied

None of this is a clever exploit. There is nothing to bypass, no race, no parser trick. The exploit is simply asking plainly for data and not being told no. That makes the root cause equally plain, and it lives in exactly one place: the data-access layer that turns a request into a query.

On the search endpoints, the code built its query filter directly from the client's `tenantIds` array and ran it. There was no step that took the authenticated user, derived the set of tenants that user actually belongs to, and intersected the requested scope with the permitted scope. An empty request scope therefore became an empty SQL filter, which returned every row, and a request scope naming a foreign tenant became a filter for that foreign tenant, which the database obligingly matched. On the by-id endpoints the failure was even barer: the code looked the object up by its primary key and returned it, with no clause comparing the object's own `companyId` to the caller's tenants before serialization. In both cases the tenant boundary existed as a column in the database and as a concept in the developers' heads, and nowhere in the actual code path that fetched and returned data. The wall was documented. It was simply never built.

This is the characteristic signature of BOLA in a mature codebase. The authentication middleware is solid, the role checks are thoughtful, and then a developer writes a repository method that takes an id and returns a row, and forgets that in a multi-tenant system "return the row with this id" is a security decision, not a lookup. The check does not belong in the controller, where it is easy to forget on the next endpoint. It belongs in the data layer, where every query passes through whether the developer remembers it or not.

## Remediation that actually closes the class

Patching the five endpoints I named would be a mistake, because the next five endpoints will have the same hole. The fix has to target the pattern.

1. **Derive scope from the token, never from the body.** The authenticated identity, and the set of tenants it belongs to, is the only trustworthy source of scope. Compute the caller's permitted tenant set server-side on every request and ignore any client-supplied scope except as a narrowing filter inside what the caller is already allowed to see. Intersect; never trust.
2. **Enforce ownership on every single object fetch.** For any by-id read or write, after loading the object and before returning or mutating it, compare the object's owning tenant to the caller's permitted set and refuse if it is not a member. This is three lines, and it must be present on each endpoint, which is the argument for pushing it down rather than copying it up.
3. **Make queries tenant-scoped by construction.** Push the tenant filter into the data-access layer itself, a repository base class, a query interceptor, a row-level security policy in the database, or an ORM global filter keyed off request context, so that it is impossible to write a query that forgets it. A control the developer has to remember is a control that will eventually be forgotten. A control that applies itself cannot be.
4. **Deny by default.** The baseline answer to "may this caller see this object?" should be no, overridden only by an explicit, positive ownership match. Default-allow means a forgotten check leaks; default-deny means a forgotten check breaks loudly in testing, which is exactly where you want it to break.
5. **Consider indirect reference maps for high-sensitivity objects.** Rather than exposing a database identifier the client can replay against other endpoints, hand each client a per-session, per-user opaque handle that the server resolves back to the real object only within that user's scope. It is not a substitute for server-side authorization, and it must never be the only control, but it raises the cost of enumeration and keeps real identifiers off the wire.
6. **Add a cross-tenant regression test.** Seed two tenants, authenticate as one, request the other's objects by id and by empty scope, and assert a refusal. If that test is green, this exact bug cannot return quietly, and "quietly" is the only way BOLA ever returns.

## At a glance

**Finding:** Cross-tenant Broken Object Level Authorization (BOLA / IDOR) on authenticated search and by-id endpoints.

**Mechanism:** The server scopes queries by a client-supplied `tenantIds` array and by a path GUID, and never intersects either with the tenants the authenticated caller actually belongs to. Empty scope returns all tenants; a foreign id returns that tenant's object.

**Demonstration:** A low-privilege HR account owning zero advances, bound to Tenant A, read Tenant B's advance requests, wallets, credit records and full employee PII, by id and by naming B's tenant.

**Impact:** Complete multi-tenant isolation failure. Any authenticated user of any tenant reads every other tenant's regulated financial PII: names, bank account numbers, salaries, dates of birth. The same missing check on a mutating endpoint allowed a cross-tenant password reset, the write-side face of the identical flaw.

**Classification:** OWASP API1:2023 Broken Object Level Authorization; CWE-639 Authorization Bypass Through User-Controlled Key; CWE-863 Incorrect Authorization.

**Severity:** Critical, CVSS 8.1 (`AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:L/A:N`), per the engagement report.

**Fix:** Derive scope from the token, enforce ownership on every object fetch, push the tenant filter into the data-access layer, deny by default, and add a two-tenant regression test.

The one line to remember
A valid login tells the server who is asking. It says nothing about what the asker is allowed to have. In a multi-tenant system, authorize the object on every read and every write, in the data layer, deny by default, and never let the client choose its own scope.

### Can a free trial account at your platform read a paying customer's records?

Imperonlabs tests tenant isolation the only way that proves anything, with two accounts and an id swap, across every object your API exposes. We separate the checks that work from the ones that only look like they do, and we hand you a fix that closes the class, not the five endpoints we happened to find. Book a scoping call.

[Request an assessment](https://imperonlabs.com/#contact)
