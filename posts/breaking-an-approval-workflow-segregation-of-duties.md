---
title: "The 403 That Lied: Breaking an Approval Workflow Without a Second Identity"
date: September 2026
source: https://blog.imperonlabs.com/blog/breaking-an-approval-workflow-segregation-of-duties.html
---

Web · Business Logic · Authorization

# The 403 That Lied: Breaking an Approval Workflow Without a Second Identity

By Yash Pandya · September 2026 · 16 min read

The first thing this application told me was the truth. I asked it to approve an offer I was not the approver of, and it said no, cleanly, with a 403 and a sensible error. Most of the time that is where a boring report ends. The authorization is enforced, write a note praising the team, move on. I stayed, because a 403 only answers the question you asked it. I had asked the wrong question. The right one was not "can I approve this offer" but "can I make myself the person who is allowed to."

**Scope note.** This is a sanitized account of a real engagement on a multi-tenant HR and applicant-tracking SaaS. The product is fictionalized here as "Harbor ATS" on `ats.example.com`, procedure names are renamed to neutral equivalents, and identifiers are examples. The CVSS, CWE, and mechanics are reported exactly as found. Nothing here points at a live system.

There is a particular kind of finding that does not show up in a scanner and does not show up in a diff of the HTTP responses, because every individual request in the chain returns a perfectly legal 200. It only exists when you step back and look at who did what across eight calls, and notice that the answer to "who" is the same name eight times. This is that finding. It came in two acts, one for each role that could pull it off, and the lesson both acts teach is the same: strong per-endpoint access control is not the same thing as a sound workflow, and the gap between them is exactly big enough to drive a binding compensation offer through.

## The target, the roles, and the approval model

Harbor ATS is a tenant-isolated hiring platform. Companies sign up, invite their people, and run the full recruitment pipeline inside it: post an opening, collect candidates, move them through stages, and at the end produce an offer. The offer is the crown jewel of the whole product, because an offer is not a draft email. In this platform an offer that reaches the `sent` state is a binding compensation document delivered to a candidate, with a role title, numbers, and the company's name on it. That is the action everything else exists to protect.

The tenant I was testing had the roles you would expect, and I had credentials for each so I could test the boundaries honestly:

- **Employee.** A plain seat. Can see things, cannot run the pipeline.
- **Recruiter / HR.** Authors applications and offers, drives candidates through the funnel.
- **Hiring manager.** Owns openings, is the person offers are usually routed to for sign-off.
- **Admin.** The elevated tenant role.

The approval model sitting on top of the offer was genuinely thoughtful, which is why it is worth describing. Each opening carries an *approval rule*. The rule says whether approval is required at all (a `waived` flag), and if it is required, it lists one or more *layers*, each layer naming a set of member user IDs and a quorum. When a recruiter authors an offer and submits it, the offer enters `in_approval` and sits in the pending queue of whoever the rule names. One of those named people calls the decision procedure, the offer flips to `approved`, and only then can it be sent. On paper, this is four-eyes done right. The author proposes, a distinct named approver disposes.

I want to be fair to the engineers here before I take their workflow apart, because they got the hard part right. The object-level check on the decision endpoint was real, as I was about to confirm the unpleasant way.

## Why the raw BFLA was correctly blocked

The obvious attack on an approval workflow is the direct one. Find an offer that is sitting in approval, waiting on someone else, and just call the decision endpoint on it yourself. This is broken function-level authorization in its most literal form: the approve action should be reachable only by the named approver, so can a non-approver reach it? I logged in as the HR seat, found an offer whose rule named the hiring manager as approver, and sent the decision directly.

```
POST /api/offers/decideApproval HTTP/1.1
Host: ats.example.com
Content-Type: application/json
Cookie: session=<HR_SESSION>

{"json":{"offerId":"of_7b6f...not-mine","decision":"approve"}}

HTTP/1.1 403 Forbidden
{"code":"FORBIDDEN","status":403,
 "message":"You are not an approver for this offer."}
```

A clean 403. Not a 500, not a silent success, not a 200 with an error buried in the body. The server knew exactly who I was, looked at the offer's approval snapshot, saw that my user ID was not in the approver set, and refused. I repeated the same call across other offers I had no business deciding and got the same refusal every time. The function-level authorization on the decision action was, I will say it again because it matters, correct. If the report had been "I tried to approve someone else's offer and could not," it would have been a one-line note under "controls that held."

Here is the thing a clean 403 does to an inexperienced tester: it closes the case. The decision endpoint checks membership in the approver set, membership did not include me, access denied, done. But that sentence contains an assumption I had not yet earned the right to make. It assumes the approver set is not something I control. The 403 tells you that the gate is locked. It tells you nothing about who holds the key to the lock, or whether, awkwardly, that is also me.

## Act 1: HR rewrites the rule and signs its own paperwork

So I asked the question the 403 could not answer. The approval rule lives on the opening. Who is allowed to edit the approval rule? I did not have to guess, because the UI exposed a settings panel for exactly this, and behind it sat a procedure. I renamed it `setApprovalRule` for this writeup; the real one does the same job. It takes an opening ID, the `waived` flag, and the full list of layers with their member user IDs. It overwrites the rule wholesale.

The recruiter/HR seat could call it. That, by itself, is defensible. Recruiters configure openings, and an approval chain is part of configuring an opening. The problem is what the recruiter is also allowed to be: the author of the offer. Nothing separated "the person who decides the rules of the approval" from "the person whose work the approval is meant to check." They were the same seat. So I did the only interesting thing left to do. I named myself the approver.

First, read the current rule to see what I was overwriting. The opening I used routed approval to the hiring manager, call them `usr_manager`:

```
POST /api/ats/getApprovalRule
Cookie: session=<HR_SESSION>
{"json":{"openingId":"op_0984..."}}

HTTP/1.1 200
{"waived":false,
 "layers":[{"name":"Manager","order":1,"quorum":"any",
            "memberUserIds":["usr_manager"]}]}
```

Then I overwrote it. One layer, one member, and the member is me, the HR user `usr_hr`:

```
POST /api/ats/setApprovalRule
Cookie: session=<HR_SESSION>
{"json":{"openingId":"op_0984...","waived":false,
  "layers":[{"name":"Review","order":1,"quorum":"any",
             "memberUserIds":["usr_hr"]}]}}

HTTP/1.1 200
{"waived":false,
 "layers":[{"name":"Review","order":1,"quorum":"any",
            "memberUserIds":["usr_hr"]}]}
```

The server accepted it without a second glance. The designated approver, the hiring manager, had just been silently removed from their own opening by someone whose work they were supposed to be checking, and no alert fired, no second identity signed off on the change to the approval chain, and nothing in the offer's eventual audit trail would obviously show that the gate had been quietly moved to stand in front of nobody.

From there it was the ordinary pipeline, run start to finish by the one seat. Create an application, author the offer against the template, submit it for approval:

```
# author the offer
POST /api/ats/createOffer
Cookie: session=<HR_SESSION>
{"json":{"applicationId":"ap_377f...","templateId":"tpl_ee7d...",
  "variableValues":{"roleTitle":"Senior Engineer","candidateName":"Example Candidate"}}}
-> 200 {"offerId":"of_3b09..."}

# submit it into the approval queue
POST /api/ats/submitOffer
Cookie: session=<HR_SESSION>
{"json":{"applicationId":"ap_377f...","offerId":"of_3b09..."}}
-> 200 {"status":"in_approval"}
```

Now the offer was sitting in `in_approval`, exactly as the design intended, waiting on an approver. I checked my own pending queue, and there it was, because I was the approver now:

```
POST /api/offers/listPendingApprovals
Cookie: session=<HR_SESSION>
{"json":{}}
-> 200 [{"offerId":"of_3b09...","status":"in_approval", ... }]
```

And then the same decision endpoint that had so correctly told me 403 twenty minutes earlier told me 200, because this time the server's honest membership check found exactly what I had put there:

```
POST /api/offers/decideApproval
Cookie: session=<HR_SESSION>
{"json":{"offerId":"of_3b09...","decision":"approve"}}
-> 200 {"offerStatus":"approved"}
```

One seat authored the offer, rewrote the rule to appoint itself, and approved its own work. Segregation of duties on the highest-value action in the product, defeated with zero second identity. And the decision endpoint never did anything wrong. It checked membership and found me, because I am the one who decided who the member was.

For completeness I tested the lazier variant, because workflow bugs usually have a lazier variant and it is worth knowing it exists. Instead of naming myself the approver, I just set `waived:true` with an empty layer list:

```
POST /api/ats/setApprovalRule
Cookie: session=<HR_SESSION>
{"json":{"openingId":"op_0984...","waived":true,"layers":[]}}
-> 200 {"waived":true,"layers":[]}
```

Offers created under that opening came back with the "requires approval" flag flipped to false. The approval step was not satisfied, it was deleted. No approver to name, no decision to make, nothing to self-approve, because the control the offer was supposed to pass through no longer existed. The same seat that benefits from skipping the check is the seat allowed to delete the check. That is the whole bug in one sentence, and I had two ways to trigger it.

I scored Act 1 as **High**. CVSS 3.1 `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N`, which lands at **7.1**. Low privileges required, because any ordinary HR seat has this, no user interaction, and a high integrity impact because the integrity of a binding compensation document is precisely what gets forged. CWE-841, improper enforcement of behavioral workflow, with a flavor of CWE-639, because the approver identity is an attacker-controlled key.

## Act 2: the manager runs the whole factory

Act 1 needs an existing opening and an application to hang the offer on. That is a mild constraint. A paranoid defender might hope it limits the blast radius to recruiters working within requisitions someone else set up. It does not, and Act 2 is where that hope dies, because the hiring-manager role has a wider reach, and that reach closes the last gap.

A hiring manager can create openings. When a manager creates an opening, it can name itself as the opening's hiring manager, which makes it the owner and, critically, the party allowed to configure that opening's approval rule. So a manager does not need a pre-existing requisition handed to it by an admin. It can manufacture the entire pipeline from an empty tenant: the opening, the candidate, the application, the offer, the approval, and the delivery, all under one session, with no other human in the loop at any step. I ran it end to end.

```
# 1. manager opens its own vacancy and names itself the manager
POST /api/ats/createOpening
Cookie: session=<MGR_SESSION>
{"json":{"name":"Self-Provisioned Opening","hiringManagerUserId":"usr_mgr"}}
-> 200 {"id":"op_c7ad...","status":"open","hiringManagerUserId":"usr_mgr"}

# 2. manager writes the approval rule to name itself sole approver
POST /api/ats/setApprovalRule
Cookie: session=<MGR_SESSION>
{"json":{"openingId":"op_c7ad...","waived":false,
  "layers":[{"name":"Review","order":1,"quorum":"any",
             "memberUserIds":["usr_mgr"]}]}}
-> 200

# 3. manager creates a candidate lead on its own opening
POST /api/ats/createLead
Cookie: session=<MGR_SESSION>
{"json":{"openingId":"op_c7ad...","name":"Example Lead",
  "email":"[email protected]","source":"sourcing"}}
-> 200 {"leadId":"ld_4d31..."}

# 4. manager promotes the lead to a real application
POST /api/ats/promoteCandidate
Cookie: session=<MGR_SESSION>
{"json":{"leadId":"ld_4d31..."}}
-> 200 {"status":"promoted","applicationId":"ap_5000..."}

# 5. manager authors the binding offer
POST /api/ats/createOffer
Cookie: session=<MGR_SESSION>
{"json":{"applicationId":"ap_5000...","templateId":"tpl_ee7d...",
  "variableValues":{"roleTitle":"Self-Provisioned Opening","candidateName":"Example Lead"}}}
-> 200 {"offerId":"of_53a1..."}

# 6. manager submits it for approval
POST /api/ats/submitOffer
Cookie: session=<MGR_SESSION>
{"json":{"applicationId":"ap_5000...","offerId":"of_53a1..."}}
-> 200 {"status":"in_approval"}

# 7. manager approves its own offer
POST /api/offers/decideApproval
Cookie: session=<MGR_SESSION>
{"json":{"offerId":"of_53a1...","decision":"approve"}}
-> 200 {"offerStatus":"approved"}

# 8. manager sends the binding offer to the candidate
POST /api/ats/sendOffer
Cookie: session=<MGR_SESSION>
{"json":{"applicationId":"ap_5000...","offerId":"of_53a1...",
  "subject":"Your offer","body":"Please find your offer attached."}}
-> 200
```

Eight calls, one session, no second person. I then read the offer back to confirm the final state was not some draft or half-committed artifact, and the server was admirably honest about what had just happened:

```
POST /api/ats/listOffers
Cookie: session=<MGR_SESSION>
{"json":{"applicationId":"ap_5000..."}}
-> 200 [{"id":"of_53a1...","status":"sent",
         "requiresApproval":true,"approvalConfigured":true, ... }]
```

Read that response slowly, because it is the finding in miniature. The offer's status is `sent`. It is binding and it is out. And `requiresApproval` is `true`: the offer *did* require approval, the control was not waived, the four-eyes rule was nominally in force. The server is telling you, in the same breath, that approval was required and that it was satisfied, by the author, alone. The workflow did not fail to require a second signature. It required one and then accepted the first person's as the second.

## What this is not: the negative controls

It would be easy to misread this as "the permissions are a mess." They are not, and the parts that held are what prove the bug is a workflow problem rather than a missing check. I spent real time trying to make it a boring permissions bug and failing, which is the most convincing evidence I can offer that it is not one.

- An **employee** calling `createOpening` got a clean **403**. Employees cannot open vacancies. The role boundary is enforced.
- An **employee** calling `setApprovalRule` to waive approval got a clean **403**. Employees cannot touch the approval flow. Enforced.
- A **manager** calling `setApprovalRule` on an opening it does *not* own got a clean **403**. The manager's reach is scoped to openings it owns. Enforced.
- And the one I opened with: `decideApproval` on an offer whose snapshot names a different approver, **403**. The decision endpoint checks the approver set. Enforced.

Every individual gate works. The employee cannot start the chain because the employee cannot create openings or rules. The manager cannot reach into an opening it does not own. The decision endpoint will not let you decide an offer whose approver set excludes you. If any one of those checks were missing, this would be a straightforward broken-access-control finding and I would have written it in five minutes. None of them is missing. The vulnerability lives in the one relationship nobody put a check on: the relationship between the author of a thing and the configurer of that thing's approval. Those two capabilities land in the same role, and that role can set the second to point at the first.

## Why this is segregation of duties, not an IDOR

This distinction is worth being precise about, because it changes the fix and it changes how you argue the severity to a team that is, understandably, proud of their 403s.

An IDOR, a classic insecure direct object reference, is when you access an object that is not yours by changing an identifier the server fails to check against your identity. If I could have passed `offerId` of someone else's offer into the decision endpoint and had it approved, that would be an IDOR, and it is exactly what the 403 prevented. The server does check the object against my identity. There is no missing ownership check on the decision action.

What I did instead was never touch an object that was not mine. Every offer I approved was legitimately, according to the server's own records, an offer I was authorized to approve, because I had legitimately configured myself as its approver through a procedure I was legitimately allowed to call. There is no step in the chain where I referenced an object I did not own or impersonated an identity I did not have. I simply occupied two roles that the workflow assumed would be occupied by two different people. That is the definition of a segregation-of-duties failure, and it maps to **CWE-841**, improper enforcement of behavioral workflow: the application enforces the steps of the workflow but not the invariant that gives the workflow its meaning. The attacker-controlled approver identity gives it the secondary flavor of **CWE-639**.

The practical consequence of that classification: you cannot fix this on the decision endpoint by tightening the ownership check, because the ownership check is already correct. The fix has to live in the invariant, one layer up from any single request.

## Business impact: rogue offers and a worthless audit trail

Strip away the HTTP and this is a fraud-and-repudiation problem wearing a web-app costume. The offer is a binding compensation document. A single recruiter or manager seat, which in a real organization is not one person but dozens, can issue fully-approved, sent offers with numbers of their choosing, to candidates of their choosing, with no other human aware it happened. A rogue or compromised insider can hire their friend or inflate the terms on a real offer, and at every step the system will report that the required approval was obtained.

The second-order damage is to the audit trail, and this should worry a compliance team more than the forged offer itself. The purpose of an approval workflow is non-repudiation: when something goes wrong, you point at the record and say two named people signed off. After this bug, that record is a lie you cannot detect from inside it. The offer says it required approval and got it. It does not say the approver was the author, because the approver-was-the-author invariant is exactly the thing nobody checked, so nobody recorded its violation. An audit trail that cannot flag its own circumvention is worse than none, because it manufactures false confidence.

## Root cause: no second-identity invariant, and a self-editable rule

The root cause is a single missing invariant, stated in one sentence: *the identity that authors or submits an offer must not also be the identity that approves it, and the identity that approves an offer must not be the identity that got to choose who the approver is.* The system enforced the shape of the workflow, offer then submit then approve then send, with great care. It never enforced the property that gives that shape a point, which is that distinct humans occupy the distinct roles. A workflow that lets one identity occupy every seat is not a workflow, it is a single-player game with extra steps.

Two design decisions combine to produce it. The first is that the approval rule is self-editable by the same role that authors offers, with no separation between authoring the flow and being subject to it. The second is that there is no self-approval guard anywhere, so even when the rule is left alone, the author being listed among the approvers is accepted rather than rejected. Either decision alone would be survivable. Together they let the author rewrite the rule to name themselves and then act on it, which is both halves of the bug reinforcing each other.

## Remediation

The fixes stack, from the narrow and immediate to the structural, and I recommended all of them.

1. **Enforce a self-approval guard on the decision, server-side.** The decision endpoint must reject a decision when the deciding identity is the offer's author or submitter, even if that identity is legitimately in the approver set. This is the smallest patch and it closes the direct self-approve, independent of how the rule got set. It is a disjointness check: author set and deciding identity must not intersect.
2. **Separate rule authoring from offer authoring.** Configuring an opening's approval rule should require a role distinct from the role that produces offers under it, or, at minimum, the approver set a rule names must be disjoint from the opening's hiring manager and from the offer's author when they are the same seat. One person should not be able to both set the rule and be its only subject.
3. **Make `waived:true` an elevated, rare, and loud action.** Waiving approval entirely should not be reachable from an ordinary recruiter or manager seat. Gate it behind admin privilege, and treat every use of it as a security event, not a configuration preference.
4. **Make approval rules immutable per offer, and audited.** Snapshot the approval rule at the moment an offer enters approval, and record every change to a rule as a first-class audit event with the actor, the before, and the after. If the gate moves, the log must say who moved it and when, so the circumvention is visible from inside the trail rather than invisible to it.
5. **Enforce genuine four-eyes as an invariant, not a shape.** The requirement is not "a decision happened." It is "a decision happened, made by an identity distinct from everyone who produced or configured the thing being decided." Write that as a server-side invariant checked at decision time, and write a regression test that runs the full single-identity chain from Act 2 and asserts it ends in rejection, not `sent`.

## How to approach this class of bug

1. When an endpoint returns an honest 403, do not close the case. Ask what controls the thing it checked against. The gate was locked; find out who holds the key.
2. For any approval or review workflow, enumerate the required roles, then ask whether a single identity can acquire every one of them. If it can, the workflow has no spine regardless of how good the per-step authz is.
3. Treat configuration that governs a check as more dangerous than the check itself. The right to edit the approval rule outranks the right to approve.
4. Read the state-reflecting responses literally. `requiresApproval:true` next to `status:sent`, decided by the author, is the whole finding stated by the server in its own words.
5. Prove it is not a permissions bug by trying to make it one and failing. The negative controls that hold are what elevate a workflow finding from "they missed a check" to "their model is unsound."

The one line to remember
Per-endpoint authorization answers "can this identity do this action." It never answers "should these actions be done by the same identity." A workflow whose integrity depends on distinct people must enforce distinctness as an invariant, because a 403 will happily guard a door that the attacker was allowed to move.

## At a glance

- **Class:** Segregation-of-duties bypass in an approval workflow (not an IDOR; object-level authz was enforced).
- **CWE:** CWE-841 (Improper Enforcement of Behavioral Workflow) with CWE-639 (authorization via attacker-controlled key).
- **Severity:** High. CVSS 3.1 `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N` = 7.1, both acts.
- **Act 1 (HR):** A recruiter seat rewrites an opening's approval rule to name itself sole approver, or sets `waived:true`, then authors, submits, and self-approves a binding offer.
- **Act 2 (Manager):** A hiring-manager seat self-provisions the entire pipeline (opening, rule, lead, application, offer, submit, self-approve, send) and lands an offer in `sent` with zero second identity.
- **What held:** Direct decision on another's offer (403), employees creating openings (403), employees waiving approval (403), managers configuring openings they do not own (403).
- **Fix:** Server-side self-approval guard; separate rule authoring from offer authoring; elevate and audit waiver; snapshot and audit rules; enforce a distinct-approver invariant with a regression test.

## Closing

The engineers on this platform were not careless. They built real per-endpoint authorization, they scoped roles correctly, and when I knocked on the decision endpoint it told me no in exactly the way a secure endpoint should. Everything they chose to check, they checked well. The bug was in the one thing they did not think to make a check at all: the assumption, baked silently into the design, that the person authoring an offer and the person approving it would naturally be two different people. Authorization asks whether you are allowed to perform an action. Segregation of duties asks whether you are allowed to perform all of them. Those are different questions, and this application only ever answered the first one. The authz was perfect, and the workflow still had no spine.

### Does your approval workflow enforce distinct people, or just distinct steps?

Imperonlabs tests the business logic a scanner cannot see: who can occupy which roles, which configuration governs which checks, and whether one identity can quietly become the whole committee. Book a scoping call and we will map where your approvals have a spine and where they only have a shape.

[Request an assessment](https://imperonlabs.com/#contact)
