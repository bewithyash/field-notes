---
title: "Testing Mobile Apps: A Working Methodology"
date: August 2026
source: https://blog.imperonlabs.com/blog/mobile-app-methodology.html
---

Mobile · Methodology

# Testing Mobile Apps: A Working Methodology

By Imperonlabs Research · August 2026 · 13 min read

A mobile app is a client you do not control, running on a device you do not own, talking to a backend that too often trusts it. Almost every serious mobile finding comes from that one sentence. The client is not the vulnerability, it is the map. The vulnerability is on the server, in the check it assumed the client already made. Here is how I work the three phases so I spend my time where the impact is.

Scope note. Test your own accounts against your own installs on devices you control. Frida and objection are for confirming server behaviour, not for tampering with other people's sessions. The high-value findings all reproduce with a single account and a proxy.

## Phase one: static analysis maps the surface, it does not hunt secrets

I unpack the APK or IPA first, but not to grep for a magic hardcoded key. Junior testers burn days chasing an analytics token that unlocks nothing. The point of static analysis is to build the map: what does this app talk to, and what does it assume?

Four things I pull out every time. Every endpoint, API host, and third-party service the app contacts, so I know the full backend surface before I send a request. The credentials that actually matter, meaning signing keys, cloud storage credentials, and long-lived tokens, not the trivial third-party analytics identifiers. The client-side security assumptions: the authentication checks, premium and feature gates, and validation logic that a rebuilt or instrumented client could simply skip. And the platform-specific weaknesses: exported Android components (activities, services, content providers, broadcast receivers) reachable by other apps, insecure storage flags, cleartext-traffic allowances, and weak deep-link handling.

By the end of phase one I do not have a vulnerability, I have a list of server endpoints and a list of things the client checks that the server had better re-check. That list drives everything that follows.

The client checks. The question is whether the server does too.

Client (attacker-controlled)

hides the "upgrade" button
sets is\_premium = false
disables the withdraw action
validates the input format
All of this can be bypassed.

Server (must re-check everything)

is this user actually premium?
is this user authorized to withdraw?
does this account own this object?
is the input valid on the backend?
If any answer is "the app checked" -> bug.

Every client-side gate is a hint about a server-side check that might be missing. Rebuild the request without the gate and see if the backend still says yes.

## Phase two: the network layer, where 80 percent of the value lives

If I could only do one phase, this is the one. A mobile app is a REST or GraphQL client wearing a UI, and the moment I can see and modify its traffic, I am testing the API as an untrusted client, which is exactly what a real attacker is.

The gate to get in is certificate pinning. Plenty of apps pin, and defeating the pin (patching the check, hooking the validation at runtime, or using a pinning-bypass module) is a routine step, not the finish line. Once traffic is visible in the proxy, the real work starts.

I test the API as what it is: an internet-facing service that must not trust its client. That means the full authorization sweep. Can I access or modify another user's objects by changing an identifier? Are role and tier restrictions enforced on the server, or only reflected in the app's UI? The recurring root cause, the one that pays over and over, is that the server trusts the client to have already checked something, and it has not. The app grays out the premium feature, so the endpoint behind it never re-verifies entitlement. The app hides the admin call, so the admin call is unauthenticated.

Then the token lifecycle, because session handling on mobile is its own surface. How are tokens issued, how do they refresh, what scope do they carry, can they be revoked, and are they bound to a device or portable to anyone who captures them? A refresh token with no revocation and no device binding is a persistent account-takeover primitive.

```
# The core mobile test, in one request.
# App UI grays out "export all users" for a standard account.
# So does the server actually enforce it, or just the client?

GET /api/v2/admin/users/export HTTP/1.1
Host: api.example.test
Authorization: Bearer <standard-user-token>

# 200 with the export  -> server trusted the hidden UI. Critical.
# 403                  -> server re-checked. Working as intended.
# The app never showed this button. The endpoint did not care.
```

## Phase three: dynamic instrumentation confirms the rest

With Frida or objection on a rooted or jailbroken device I close the loop on the things the network layer cannot show me directly. I confirm the server does not re-validate a client-side gate by forcing the gate open in memory and watching whether the backend accepts the resulting action. I inspect sensitive data at rest across the storage mechanisms the app actually uses (preferences, keystores, local databases, cache files) to see what is stored in the clear that should not be. And I watch the system logs during sensitive flows for tokens, credentials, or personal data leaking where any app or anyone with device access could read them.

Instrumentation is powerful and it is also the easiest place to generate noise, so I keep it pointed at one question: does removing a client-side control change what the server allows? If it does, the server is the finding. If it does not, the missing control is at most a defence-in-depth note.

## Prioritise server impact, not client control

The fastest way to write a weak mobile report is to list every missing client-side protection: no root detection, no obfuscation, no pinning, debuggable build. On their own these are hardening gaps, not vulnerabilities, because the threat model already assumes the attacker owns the client. Separate the genuine findings from that noise by asking one thing of every issue: what is the server-side impact? Authorization flaws that let one user reach another's data or actions, and sensitive data exposure that a real attacker can actually retrieve, are the findings that matter. A missing client-side control matters only when its absence lets you reach a server that should have said no.

## How to approach any mobile app

1. Unpack the binary to map endpoints, real credentials, client-side gates, and platform weaknesses. Do not chase trivial secrets.
2. Defeat certificate pinning and get the API traffic into a proxy. This is where most value lives.
3. Test the API as an untrusted client: full authorization sweep, object ownership, and tier and role enforcement.
4. Work the token lifecycle: issuance, refresh, scope, revocation, and device binding.
5. Use instrumentation to confirm the server re-checks client-side gates and to inspect data at rest and in logs.
6. Rank by server-side impact. Missing client protections are hardening notes unless they unlock the backend.

The one line to remember
The client is the map, not the target. Every gate the app enforces is a question about whether the server enforces it too, and the bug is always the check the backend assumed somebody else already made.

### Does your backend trust your mobile app more than it should?

Imperonlabs tests mobile apps the way an attacker does: as an untrusted client against an API that has to defend itself. Book a scoping call and we will find the checks your server left to the app.

[Request an assessment](https://imperonlabs.com/#contact)
