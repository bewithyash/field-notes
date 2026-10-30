---
title: "The Fastest Account Takeover Is the One Where You Never Log In"
date: September 2026
source: https://blog.imperonlabs.com/blog/unauthenticated-account-takeover-fintech-api.html
---

Web · API · Broken Auth

# The Fastest Account Takeover Is the One Where You Never Log In

By Yash Pandya · September 2026 · 15 min read

Most account takeovers are a grind. You phish a credential, or you brute a reset code, or you chain three bugs and a lucky race to borrow someone's session for ninety seconds. On this engagement I did none of that. I sent one request, to one endpoint, with no token and no cookie, and the server set a user's password to a value of my choosing and replied `200 {"data":true}`. The fastest account takeover is the one where you never had to log in, and a fintech handed me exactly that on its most sensitive controller.

**Scope note.** This is a sanitized account of a real engagement on an earned-wage-access / salary-advance fintech. The company, product, and domain here are inventions. I use `FlexPay`, the app `FlexPay Wallet`, and `api.example.com`, and I have renamed routes to neutral equivalents. The technical mechanics, the HTTP shapes, the chain, and the severity are exactly as found. Every destructive action was demonstrated only against a disposable test account I created. Nothing here points at a live system.

Let me set the stage, because the context is what turns a missing checkbox into a career-defining incident. FlexPay is an earned-wage-access product. Employees of enrolled companies connect their payroll, and the app lets them draw part of their earned salary before payday. In plain terms, it is a wallet that both moves money and holds the kind of identity data a criminal would frame and hang on a wall. When I pull a target like this apart, I am not thinking about XSS alerts popping. I am thinking about who can touch the money and who can touch the people.

## The target: a wallet that moves money and holds identities

FlexPay Wallet is a thin mobile and web client sitting on top of a single REST backend. That architecture is common and sensible, but it has a consequence worth stating plainly: the client is decoration, and the backend is the entire security boundary. Every meaningful decision, who you are, what you may see, and whether you may move money, happens behind the API. If the backend gets authorization wrong, no amount of polish on the app saves it. The app is a steering wheel. The API is the car.

The threat model for a product like this has two assets stacked on top of each other, and both are regulated. The first is money: salary-advance balances, request flows, and the eligibility logic that decides how much a user can draw. The second is identity: full names, emails, phone numbers, dates of birth, government ID numbers, bank account numbers, and a full salary breakdown per person. Either asset alone would make this a serious target. Together they make it the sort of system where a single broken control is not a bug report, it is a breach notification.

So the question I carry into the first hour is blunt. Of all the controls this team built, which one only looks like a control? Teams rarely fail by doing nothing. They fail by doing the right thing almost everywhere and leaving one door off the alarm system.

## Mapping the API by reading the client, not the login page

I did not start at the login screen. Login screens are where applications want you to start, which is a decent reason not to. I started by pulling the client apart to recover the full list of endpoints it talks to, because the route the UI never shows you is often the route nobody remembered to protect.

The client gave up its whole API map without much of a fight. There was no real obfuscation, so the endpoint paths, model names, and route templates read out of the binary like a table of contents. I catalogued everything into one list and then did the single most useful thing in API testing: I ignored the user interface entirely and started asking each endpoint, directly and anonymously, whether it would answer a stranger.

This is the part people skip. They test the flows the app performs, in the order the app performs them, with a session the app gave them. That tests the happy path and little else. I want the unhappy path: what happens when I call the account-management routes with no `Authorization` header at all. So I walked the list with a stripped request, no token, no cookie, and watched the status codes.

Most of the sensitive routes behaved. A direct object lookup like `GET /account/{id}` answered with a clean `401 Unauthorized`. The transaction routes that create and read salary-advance requests answered with `401` too. Good signs. An authentication layer existed, it was wired into the framework, and on those routes it was doing its job. That matters for the story, because it means what came next was not a platform with no auth. It was a platform with auth applied one route at a time, and a hand that got tired.

Then I hit the account directory. I sent a search request to `POST /account/directory` with an empty JSON body, expecting the familiar `401`, and instead I got `200` and a page of real people.

## The endpoint that answered a stranger

Here is the request, sanitized, that should have been rejected and was not:

```
POST /account/directory HTTP/1.1
Host: api.example.com
Content-Type: application/json

{}
```

And the response, with the values replaced by representative examples:

```
HTTP/1.1 200 OK
{"statusCode":200,"data":{"count":85,"data":[
  {"id":"00073a58-....","name":"Example Employee",
   "email":"[email protected]","iban":"XX00EXAMPLE0000000000000",
   "phone":"000000000","govId":"0000000000","birthDate":"1987-12-07",
   "basicSalary":914,"otherAllowances":1002,
   "bank":{"id":"6","name":"Example Bank"},
   "companies":[{"id":"480a14bb-....","name":"Example Co"}]},
  ... ]}}
```

Read what is in that payload, because this is not a profile page. For every enrolled employee in the system, an anonymous caller receives the full name, email, phone, date of birth, government ID number, bank account number, and a line-by-line salary breakdown. The endpoint ignored unknown filter fields and accepted a keyword, which means it is not just a dump, it is a lookup. You can ask it for a specific person by name or phone and it will oblige. Eighty-five real individuals sat behind that empty request. I capped evidence handling to the default page and did not page through the rest, because the point was proven on request one.

On its own, that is a catastrophic PII and financial-data leak, a breach of exactly the kind of regulated data this product exists to protect. But a directory read is a disclosure bug. What makes it the first link of a takeover chain is a field in that response you might skim past: `id`. Every record carries the server's internal user identifier. The directory did not just leak the data. It leaked the primary keys.

Because the next endpoint I tried wanted exactly one input, and the directory had just handed me a copy for every user on the platform.

## The takeover chain, one request at a time

The account controller had a route for setting a user's password by their identifier. In a sane design this is an administrative function, locked behind an admin session, used by support staff to help a stuck user. Here is what it looked like when I called it with no credentials at all:

```
POST /account/reset HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"userId":"8c08e29f-....","newPassword":"NewPass!2026","confirmPassword":"NewPass!2026"}
```

```
HTTP/1.1 200 OK
{"statusCode":200,"data":true}
```

No authentication. No current-password check. No emailed reset code, no one-time token, no proof whatsoever that the caller is the account owner or an administrator. The endpoint takes a target user identifier and a new password and it does the thing. That `userId` is the value the directory leaks for everyone, so this is not a reset for an account I control. It is a reset for any account on the platform, chosen from the directory I already dumped.

I ran this only against a disposable account I had created for the purpose, confirmed the server accepted the new password, and then logged in with it through the normal client login flow. The login returned `200` and proceeded into the session. The reset produces a working credential. That is the whole ballgame.

There was one small wrinkle, and it is worth walking because it shows how a broken-auth cluster self-heals around its own gaps. The reset call succeeds for any identifier, including accounts that are currently inactive, but a successful login additionally requires the account to be active. So if your chosen target is disabled, you are briefly stuck. Briefly, because the same controller also exposed an account-activation route, and it had the same missing gate:

```
POST /account/activate?userId=00000000-0000-0000-0000-000000000000 HTTP/1.1
Host: api.example.com
```

```
HTTP/1.1 400 Bad Request
{"statusCode":400,"message":"NoData","data":null}
```

I sent that against a non-existent all-zero identifier on purpose, as a boundary test, so that nothing real was touched. The response is the tell. A route with an authentication gate returns `401` to an anonymous caller before it ever looks at the body, which is exactly what the sibling routes did. This route skipped straight past authentication, processed my request, looked for the user, and only then complained that there was no such record. The gate is not there. A real identifier would have been activated. So the full chain, from a cold start with zero credentials, reads like this:

From zero credentials to a working login as anyone

1 Read the directory
unauth dump leaks every
user id + PII

2 Activate if needed
unauth state change
on the target id

3 Reset the password
unauth set, no code,
by user id

4 Log in as them
new password works;
full session, any role

Three unauthenticated calls and a normal login. No phishing, no brute force, no victim interaction. The directory supplies the identifier that the other two routes consume.

Step one gives me the identifier for anyone, including the highest-privilege administrator on the platform, whose record sits in the same directory as everyone else. Step two, used only when the target is inactive, flips the account on. Step three sets a password I know. Step four is just logging in, the way any user would, with a password I chose. There is no race window, no stolen cookie, no social engineering. The attacker is anonymous at every step except the last, and at the last step the attacker is the victim.

## What the attacker walks away with

It is tempting to file this as "password reset bug" and move on, so let me spell out what possession of an arbitrary account on this platform actually means, because the two stacked assets from the threat model both fall at once.

Take over an employee account and you are inside their wallet. You see their salary-advance history and eligibility, and you are positioned on the money-movement flows as them. Take over a company-side administrator and you inherit a tenant's worth of employees, their payroll relationships, and the controls that onboard and manage users. Take over the platform super-administrator, whose identifier is in the very directory that step one dumps, and the distinction between tenants stops mattering, because you are now above all of them.

And this is before you spend the directory dump itself. Government ID numbers plus bank account numbers plus full names and dates of birth is a turnkey identity-theft kit, and it is not for one unlucky user, it is for the entire enrolled population in one unauthenticated request. The password reset then lets an attacker rebind the credentials on those same identities, which is the part that keeps regulators up at night: it is not merely that an attacker can read who you are, it is that an attacker can become you inside a financial product and the real you is locked out of the account that holds your salary.

That is the difference between a disclosure finding and a takeover finding. One tells an attacker about your users. The other hands them the users.

## Root cause: authentication bolted on per route

None of this is exotic. There is no clever parser trick, no cryptographic flaw, no timing oracle. The root cause is the most ordinary failure in web security, which is precisely why it is so common and so lethal: authentication was applied per route, by hand, and three of the account controller's routes did not get it.

Remember the sibling routes that correctly returned `401`. Those are not an accident of a different codebase. They are the same controller, in the same application, behind the same framework, which means the mechanism to require authentication was present, understood, and in use elsewhere in the very same file. Someone wrote the gate and attached it route by route. On the directory read, the reset, and the activate routes, the attachment is simply missing. The authentication model was opt-in, and these endpoints never opted in.

In vulnerability-classification terms this is CWE-306, Missing Authentication for a Critical Function, sitting under the broader CWE-287, Improper Authentication. The password-reset specifics also land squarely in CWE-640, Weak Password Recovery Mechanism, because a reset that accepts an arbitrary target identifier with no proof of ownership is a recovery flow with the ownership check deleted. And there is an authorization dimension chained in here too: even an authenticated version of these routes would need a server-side ownership and role check, because accepting a client-supplied `userId` and acting on it without confirming the caller may manage that object is textbook broken object-level authorization, CWE-639. The platform got the full house of access-control failures: it skipped authentication, and the endpoints were never designed to verify ownership either.

The quiet lesson is that per-route authentication is a design that fails silently. A missing gate throws no error. The endpoint works perfectly. It serves its intended function flawlessly to the wrong people. Nothing in a normal test, a normal deploy, or a normal day reveals that the route is open, because open looks identical to working. That is the trap of opt-in security: the default is dangerous and the danger is invisible.

## Why the scanners shrugged and a human didn't

A reasonable question from a security team that owns scanners is: why didn't our tools catch this? They had scanners. The scanners were green. The answer is in how this bug is shaped.

An automated scanner looking at `POST /account/reset` sees an endpoint that returns `200` for a well-formed request. That is a success response. Scanners are built to flag errors, anomalies, reflections, injections, and known signatures. They are not built to hold an opinion about whether a `200` was morally entitled to happen. There is no payload here, no malformed input, no error to pattern-match. The request is boring and the response is boring. The only thing wrong is that the caller should not have been allowed, and "should not have been allowed" is a statement about business logic and intended privilege, not about syntax. No signature encodes it.

Worse, a scanner that crawls the application as an authenticated user will never see these routes at all, because the UI never navigates to the administrative reset. And a scanner crawling unauthenticated has no way to know that the clean `200` from the directory was supposed to be a `401`, because it has no model of what the directory is or who should see it. The tool lacks the one thing that caught this: an understanding that this is a sensitive function, so an anonymous `200` is not a pass, it is the whole finding.

What caught it was dull, manual work. Recover the complete endpoint map from the client instead of trusting the UI. Call every sensitive route anonymously and compare the status codes against each other. Notice that siblings in the same controller disagree about whether they need a token, and treat that disagreement as the loudest alarm in the building. The `401` on one route is what makes the `200` on its neighbor indefensible, and only a human comparing intent across routes reads it that way. Tools find bugs that look like bugs. This one looked like a feature.

## Blast radius: everyone

I try to be precise about blast radius rather than dramatic, so here it is in flat terms. The directory read affects every enrolled user, because it returns the whole population in one call. The password reset affects every account, because the only input it needs is an identifier the directory just leaked for everyone, and it draws no distinction between an employee, a company administrator, and the platform super-administrator. The activation route removes the one condition, an inactive account, that might otherwise have slowed a takeover down. There is no tenant boundary protecting one company's users from this, because the anonymous caller is outside all tenants and above all of them.

So the honest blast radius is not "some users under some conditions." It is every user, every tenant, and every administrator, from the public internet, with no credentials and no interaction from the victim. In an earned-wage-access product that is a total, unauthenticated compromise of a system that holds regulated financial identity data and sits on money movement. There is not a smaller way to say it.

## Fixing it: deny by default

The patch for the immediate bug is a one-liner per route: attach the authentication requirement the sibling routes already use, add an administrative authorization policy to the reset and activate functions, and enforce a server-side check that the caller is permitted to act on the target account rather than trusting a client-supplied identifier. Then route self-service password reset exclusively through a code-verified flow, so a reset always proves ownership. And because these routes have been reachable from the internet, every account should be treated as potentially compromised: force credential rotation and invalidate sessions platform-wide as part of remediation, not after someone asks.

But patching three routes fixes three routes. It does not fix the thing that produced three open routes, which is the per-route, opt-in design. The real remediation is architectural, and it is one idea: deny by default.

Flip the model so that authentication is required everywhere unless a route explicitly, visibly, and deliberately opts out. In a framework that means a global authentication filter or middleware at the top of the pipeline, so that a newly added endpoint is locked the moment it exists and a developer has to take a conscious, reviewable action to make it public. The handful of genuinely public routes, a health check, a login endpoint, become the exceptions you can count on one hand and audit in a minute, instead of the security of the whole surface depending on nobody ever forgetting an annotation. Pair that with server-side object-level authorization as a standing rule, so identity comes from the session and never from a field in the request body. Then add the monitoring that would have made this loud: alert on unauthenticated `200`s to sensitive controllers, and on password resets that are not preceded by a verified recovery flow.

The difference between opt-in and deny-by-default is the difference between a system that is secure until someone forgets, and a system that is secure until someone deliberately decides otherwise. Only one of those survives contact with a real team shipping real features on a real deadline.

## At a glance

- **Severity:** Critical.
- **CVSS v3.1:** 9.8 for the unauthenticated account takeover (AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H). The unauthenticated directory read and activation routes score in the 9.1 range on their own and are rated Critical in business context because the exposed data is regulated financial identity data and the defects chain: the directory supplies the identifiers that make the reset a whole-platform takeover.
- **CWE:** CWE-306 (Missing Authentication for a Critical Function) and CWE-287 (Improper Authentication), with CWE-640 (Weak Password Recovery Mechanism) on the reset and CWE-639 (broken object-level authorization) on the identifier-driven routes.
- **Reach:** Unauthenticated, network, no user interaction, every user and every administrator across every tenant.
- **Root cause:** Per-route, opt-in authentication. The gate existed and worked on sibling routes and was never attached to these.

The one line to remember
If authentication is something each route has to remember to ask for, then security is one forgotten line away from a total takeover. Make the default deny, and make public a decision someone has to sign for.

### Do you actually know which of your endpoints answer a stranger?

Imperonlabs recovers your real API surface from the client, calls every sensitive route the way an anonymous attacker would, and reads the disagreements between your own routes the way a breach would. If your authentication is applied per route, we will find the route that was forgotten before someone else does. Book a scoping call.

[Request an assessment](https://imperonlabs.com/#contact)
