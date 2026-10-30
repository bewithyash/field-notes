---
title: "Retesting a Fixed Program: Verifying the Patch, Finding the Gap"
date: August 2026
source: https://blog.imperonlabs.com/blog/retesting-a-fixed-program.html
---

Methodology

# Retesting a Fixed Program: Verifying the Patch, Finding the Gap

By Imperonlabs Research · August 2026 · 7 min read

Every finding from the first engagement was closed. Verified, reproduced, gone. And the retest still produced the highest-severity issue of the whole program. That is not a contradiction. A retest is not just a checklist that confirms fixes held. The pattern of the fixes is itself intelligence, and reading it told us exactly where the next bug had to be.

Scope note. Generalized from a real retest engagement. Endpoints, roles and record types are described in the abstract. What matters here is the method, not the target.

## The prior findings, verified closed

Three issues from the original assessment came back for validation, and all three held up under retest.

- **Insecure local storage, moved to the keystore.** Session tokens that once sat in plain application preferences were now in the encrypted platform secure store. We checked the old cleartext location. It was empty.
- **IDOR on delete, now `401`.** An endpoint that had allowed cross-user deletion now demanded the right role and rejected the request. Testing with controlled accounts, unauthorized deletions were refused every time.
- **Broken message authorization, now enforced.** A manager-messaging feature that used to leak across organizational boundaries now applied role checks and denied access from roles that had no business reaching it.

Clean sweep. If a retest were only about confirming closures, this is where the report would end, three green checkmarks and a handshake. But the fixes were telling us something, and it was worth listening to.

## What the fix pattern revealed

Line the three remediations up next to each other and the shared characteristic jumps out. Every one of them added an authorization check to a state-changing operation. The delete got a check. The message send got a check. The storage write moved to protected storage. The team had clearly done a pass over their write paths and hardened them, methodically and well.

Which raised the obvious question: what about the reads? A fix pattern that concentrates entirely on writes is a strong signal that the reads were never part of the same conversation. So we tested them, from a low-privilege account, and found authorization enforced on writes but absent on reads. Ordinary users could enumerate records well beyond their access level. A directory-disclosure vulnerability, and it outranked everything from the original engagement.

A retest has three phases, not one

1 Confirm each prior fix
controlled accounts, minimal repro, state plainly whether closure holds.

2 Test the fix's neighborhood
the patched DELETE has GET and PUT siblings. Exercise all of them.

3 Read the shape of the remediation
all fixes on writes? the unchecked class is reads. Go find it.

Confirming closures is phase one. The highest-severity finding usually comes from phases two and three, where the fixes point.

## The methodology

Every retest we run follows the same three phases.

1. **Confirm each prior fix.** Use controlled accounts and the minimal reproduction steps, and state plainly whether the closure holds. No hedging. A fix is confirmed or it is not.
2. **Test the fix's neighborhood.** A patched `DELETE` lives next to a `GET` and a `PUT` on the same resource. If only the delete was hardened, the siblings are prime suspects. Exercise the whole family, not just the endpoint on the ticket.
3. **Read the shape of the remediation.** How the team fixed things tells you how they think, and what they are likely to have missed. All fixes clustered on writes means the read paths were never in scope for the developers, which makes them squarely in scope for you.

## The lesson

The manner of a fix is intelligence about an organization's blind spots. This team's write-side protections were solid, genuinely well done. But the total absence of read-side authorization, which their own fix pattern advertised, became the retest's highest-severity finding. A retest that stops at confirming closures leaves the most valuable finding on the table. Confirm the patch, then read what the patch tells you about everything the team did not think to touch.

The one line to remember
A fix is a confession. When every patch lands on the write paths, the read paths are where the next finding is already waiting. Confirm the closure, then hunt the shape of it.

### Is your retest confirming fixes, or finding the next gap?

Imperonlabs verifies every closure and reads the remediation pattern to surface the class of bug the fixes left behind. Book a scoping call and get a retest that earns its place.

[Request an assessment](https://imperonlabs.com/#contact)
