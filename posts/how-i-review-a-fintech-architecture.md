---
title: "How I Actually Review a Fintech Architecture"
date: August 2026
source: https://blog.imperonlabs.com/blog/how-i-review-a-fintech-architecture.html
---

Architecture · Fintech

# How I Actually Review a Fintech Architecture

By Imperonlabs Research · August 2026 · 15 min read

A penetration test asks what breaks when you send a packet. An architecture review asks a harder question: where does this system break before a single packet is sent? For fintech that question is not academic. The assets are money, the money movement is irreversible, and a missing authorization check does not leak a record. It moves funds.

Scope note. The reference platform below is a composite drawn from real reviews. Component names are generic on purpose. The point is the questions you ask at each boundary, not the vendor logo on the box.

Most reviews I inherit start life as a slide with rounded rectangles and arrows that mean whatever the person drawing them wanted them to mean. That diagram is where the client thinks the system is. My first job is to redraw it as something an attacker would recognise: a data flow, with every place data crosses from one owner or privilege level to another marked in red. Those red lines are trust boundaries, and almost every real finding I have ever written was a missing or wrong control on a boundary crossing.

So the method is simple to state and slow to execute. Map the flows. Mark the boundaries. Interrogate the control on each one. Rank what is missing by how much money it moves and how easy it is to reach.

## Redraw the picture as data, not boxes

I take the client's architecture diagram and rebuild it around a single question per arrow: what data is moving here, who owns it on each end, and what privilege level does it sit at. The moment an arrow crosses from a tenant to the shared platform, from the edge to an internal service, from an unauthenticated caller to an authenticated one, or from your control plane to a third party, that arrow is a boundary. I annotate it and move on. By the end I am not looking at a system, I am looking at a list of crossings, and each crossing owes me an answer about the control that guards it.

## The reference platform I keep in my head

Across AWS, GCP and OCI I keep running into the same payment platform wearing different clothes. Customer and partner APIs at the front. An OIDC identity service. A payments authorization service. A ledger. A message bus that is some flavour of Kafka. A cache that is some flavour of Redis. A relational database. And a fistful of external integrations for processors and KYC. The managed service names change. The architectural questions do not. That is the single most useful thing to internalise: the cloud is a detail. Whether the money-moving topic sits on managed Kafka, Pub/Sub or OCI Streaming, the question is identical. Who is allowed to produce to it?

The cloud is a detail. The boundary questions are not.

AWS
GCP
OCI

MSK

Pub/Sub

OCI Streaming
Who can produce to the money-moving topic?

ElastiCache

Memorystore

OCI Cache
What does the app trust as true because the cache said so?
Change the vendor, keep the question. That is the whole trick.

Three clouds, one platform. The findings live in the questions, and the questions do not move.

## Walking the components, one boundary at a time

### Edge and gateway

The first thing I hunt for is the service that quietly grew a public IP outside the WAF, because the WAF is only a control if traffic cannot go around it. I check that TLS termination at the edge does not drop to plaintext on the internal hop, and that the partner and customer APIs do not share a header-based trust that one can forge to impersonate the other. A gateway that trusts an `X-Internal-Auth` header set by "the edge" is a gateway that trusts anyone who can reach it directly.

### Identity, OAuth and OIDC

This is where fintech either holds or falls over. My checklist per service:

- Every service validates the token in full. Signature, `iss`, `aud`, and expiry, on every hop, not just at the gateway.
- Public clients use Authorization Code with PKCE, not implicit flows or embedded secrets.
- Scopes are enforced downstream. A read-balance token must not be able to move funds because the payments service never checked the scope.
- Refresh tokens rotate, and logout actually revokes them.
- Machine identities use service-to-service auth with least privilege, not one shared god credential.
- Multi-tenant claims are verified against the resource. A token that says partner X but acts on partner Y's accounts is cross-tenant fund movement, and it is the finding I look for first.

### The message bus

Kafka and its cousins are where the money-moving events flow, and they are frequently deployed with the assumption that the network is the security. I check for authentication (mTLS or SASL) and per-topic ACLs, and specifically who can produce to the high-value topics. I look at whether payloads are encrypted, what the retention policy exposes to a later reader, whether consumers are idempotent, and whether a replayed message re-triggers a transfer.

### The cache

The interesting question with Redis is not whether it is encrypted at rest. It is what the application logic treats as ground truth because the cache returned it. I map that surface, then check whether locks and idempotency keys survive across the operations that depend on them, whether a TTL can expire mid-operation and reopen a race, and whether Lua scripting is exposed to anyone who should not have it. An evicted idempotency key is a duplicate payment waiting to happen.

### Database and ledger

Access control matters more than encryption at rest here, because the threat is a service reaching data it should not, not a stolen disk. I want per-service least-privilege credentials rather than one shared user, a ledger that is genuinely append-only and double-entry rather than a mutable balances table, backups and replicas that are guarded as tightly as the primary, and no reporting tool quietly concatenating SQL on the side.

### Third-party integrations

The outbound boundary is the one people forget, and it is where standing access accumulates. My checks:

- Inbound webhooks are verified by signature over the raw body, before parsing.
- The system re-fetches authoritative data from the processor rather than trusting the amount in the webhook payload.
- Outbound calls are signed and their responses validated.
- Every piece of standing access is inventoried: VPNs, API keys, IAM roles that a partner or vendor holds into your environment.
- Integration secrets live in a vault, not in an environment variable committed two years ago.

## Then, and only then, STRIDE

Once the flows and boundaries are mapped, I apply STRIDE to each crossing and immediately throw away the categories that do not represent a realistic threat to this system. The output is not a grid. It is a ranked list, scored by impact times reachability, that reads like this:

```
CRITICAL  Multi-tenant token confusion moves funds across partners
CRITICAL  Forged or replayed webhook books an unpaid order as settled
HIGH      Any authenticated service can produce to the payout topic
HIGH      Idempotency key eviction reopens a duplicate-transfer race
HIGH      Shared database user gives every service full-table access
HIGH      Gateway trusts a spoofable X-Internal-Auth header
MEDIUM    Origin reachable directly, bypassing the WAF
```

Notice that not one of those is "the diagram is wrong." Every line is a specific, boundary-localised threat with a dollar consequence attached. That is the deliverable a fintech team can actually act on.

The one line to remember
A fintech architecture review is not a diagram critique. It is a ranked list of boundary crossings where a missing control moves money, sorted by how much and how easily. Everything else is scenery.

### Do you know which boundary moves your money?

Imperonlabs reviews payment platforms the way an attacker reads them: as data flows and trust boundaries, ending in a ranked list your engineers can ship against. Book a scoping call.

[Request a review](https://imperonlabs.com/#contact)
