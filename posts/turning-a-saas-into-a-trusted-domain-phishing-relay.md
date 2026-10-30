---
title: "Turning a SaaS Into a Trusted-Domain Phishing Relay"
date: October 2026
source: https://blog.imperonlabs.com/blog/turning-a-saas-into-a-trusted-domain-phishing-relay.html
---

Web · Abuse of Functionality · Phishing

# Turning a SaaS Into a Trusted-Domain Phishing Relay

By Yash Pandya · October 2026 · 14 min read

Most of my favorite findings are not exploits. They are features. On this engagement I did not defeat a control, decode a token, or smuggle a payload past a parser. I asked the product to send an email, which is the one thing it was built to do, and it sent exactly the email I asked for: from the company's own trusted, DKIM-signed domain, to a recipient I chose off the street, with my subject, my body, and eventually my raw HTML. I did not need an exploit. I needed the product to do its job, pointed at the wrong person.

**Scope note.** This is a sanitized account of a real engagement on a multi-tenant B2B SaaS that sends email on behalf of its customers. The company, product, and vertical here are invented. "TalentForge" and every `example` domain are stand-ins. HTTP mechanics, status codes, CWE references, and severity are faithful to the real finding; names, addresses, and payloads are illustrative.

The target, which I will call TalentForge, is a hosted platform that runs a business process end to end and emails the humans involved at each step. Think of any SaaS that sends mail on your behalf: invoicing tools, applicant systems, contract and e-signature flows, support desks. The shape is always the same. The platform owns a sending domain, warms it, aligns it with SPF and DKIM, publishes a DMARC record, and then dispatches transactional messages that land softly in inboxes because the whole internet has learned to trust that domain. That deliverability is an asset built over months. It is also, if you expose the send button to the wrong actor with the wrong amount of control, a loaded weapon pointed outward.

The authorization model was, to its credit, strong. I spent a long time trying to read other tenants' data, escalate roles, and smuggle objects I was not allowed to touch, and I failed at all of it. Object-level and function-level access control held. So I stopped asking "what am I not allowed to do" and started asking the more productive question: "what am I allowed to do, and what happens if I do it to the wrong target." That is where this lived.

## The feature, and who gets to pull the trigger

TalentForge has a document workflow. A mid-privileged role, which I will call a manager seat, can run a short pipeline: create a prospective contact, promote that contact into an active record, attach a document to that record, submit the document for approval, and then send it out by email to the contact. In normal use the contact is a real person the manager is legitimately corresponding with, the document is a real document, and the email that goes out is a polite, branded notification telling the recipient that something is waiting for them.

Two facts about this pipeline mattered more than the pipeline itself.

First, the manager seat could provision the entire chain by itself. In a well-separated workflow, the person who creates a record is not the person who approves the outbound action, and the recipient is not something either of them types freely. Here, one manager seat held every primitive: it could open its own item of work, set or waive the approval rule on that item, create the document, submit it, approve its own submission, and send. No second human, no second identity, no out-of-band verification. The segregation of duties that the UI implied was not enforced on the API. (That self-approval chain is its own finding; here it matters only because it means a single low-trust actor can stand up the whole sending apparatus unassisted.)

Second, and this is the heart of it, the recipient address was never validated against anything. When the manager creates the contact, the contact's email is a free-text field. There is no requirement that it belong to the tenant, to a verified candidate, to a domain the company controls, or to anyone who has ever agreed to be contacted. You type an address. The platform believes you. Later, when you send, that address is where the mail goes, from the trusted domain, signed and aligned.

So the ingredients were all on the counter: a trusted sending identity, a send action reachable by a mid-privileged seat, a recipient field under my full control, and (as I will get to) content fields under my full control too. Nobody had to leave a door unlocked. The feature was the door, and it opened outward onto the entire internet.

## The request that sends the mail

The platform speaks a typed, JSON-over-HTTP RPC style. Procedures have names and take a JSON envelope; nothing exotic. After walking the pipeline (create contact, promote, attach document, submit, self-approve) I arrived at the send call. Sanitized to `example` domains and neutral procedure names, the request that actually dispatches mail looks like this:

```
POST /api/workflow/sendDocument HTTP/1.1
Host: app.talentforge.example
Content-Type: application/json
Cookie: __session=<valid session for a manager seat>

{"json": {
  "recordId":   "01a0-...-record",
  "documentId": "01a0-...-document",
  "subject":    "Action required: verify your account",
  "body":       "Please confirm at https://login.evil.example/ to continue."
}}
```

And the response, with no drama at all:

```
HTTP/1.1 200 OK
Content-Type: application/json

{"json": {"documentId": "01a0-...-document"}}
```

Two hundred. The document's status flipped to `sent` and a `sentAt` timestamp appeared on it. From the platform's point of view, a routine transactional email had just gone out, correctly, to the address on file. From mine, a message with my subject and my body had just been handed to a trusted mail service bound for whatever address I had typed into a field three steps earlier. The server did not ask whether `login.evil.example` was a place anyone should be sending customers, because asking that is not its job. Its job is to send the document, and it did its job.

## Three levers, all mine

A phishing message is only as good as the attacker's control over three things: who it reaches, what it claims to be, and what it actually says. I tested each lever independently so I could report precisely how far each one bent, rather than waving at "the email is attacker-controlled."

**The recipient.** This was the lever I expected to be clamped and it was not clamped at all. I set the contact's email to a synthetic external address that had nothing to do with the tenant. No allow-list check, no "this domain is not associated with your account" error, no verification step where the recipient has to click a link to confirm they want mail. I could have put any address there: a competitor's finance team, a customer's whole distribution list one at a time, a journalist, a regulator, my own aunt. The platform treats the recipient as data the operator supplies and never as something to be authorized. For a tool that sends from a trusted domain, that is the entire ballgame. The value of the attack is a direct function of how little the recipient field is constrained, and here it was constrained by nothing.

**The subject.** Free text, reflected verbatim into the outgoing message's `Subject` header. I set it to things a real phish uses: urgency ("Action required"), impersonation of a security team, a fake ticket or reference number. All passed through untouched. The subject is the first thing a human reads and the thing a tired person acts on, and I owned it completely.

**The body.** Also free text, and this is where the finding stops being theoretical. The body went into the message as sent. I could write a complete, self-contained lure: a pretext, a reason to act now, and a link to infrastructure I control. On its own, with plain-text body, that is already a working phishing message wearing the company's domain as a costume. As I will cover in a moment, the body did not stay plain text for long.

I want to be clear about what makes this different from typing a nasty message into a webmail compose window. In webmail, the three levers are mine too, but the fourth thing, the sending identity, is not. I send as me, from my unremarkable address, across infrastructure with no reputation and no alignment to the brand I am imitating. Here the first three levers are mine and the fourth is borrowed from an organization the recipient already trusts. That borrowing is the whole prize, and it deserves its own section.

## Why a trusted domain is the entire point

Modern email abuse is not blocked by cleverness; it is blocked by reputation and by authentication. Understanding why this finding is dangerous means understanding the three letters that stand between an attacker and an inbox.

**SPF** lets a domain publish which mail servers are allowed to send on its behalf. When a receiving server sees mail claiming to be from `talentforge.example`, it checks whether the sending IP is on that domain's authorized list. Spoofers fail this. The platform passes it natively, because the platform genuinely is an authorized sender for the domain.

**DKIM** adds a cryptographic signature. The sending infrastructure signs the message with a private key, and the receiver verifies it against a public key in the domain's DNS. A forged message cannot produce a valid signature. A message the platform actually sent carries a real, valid signature, because the platform holds the key. My phishing body rode out under that signature like a stowaway in a diplomatic bag.

**DMARC** ties the two together and tells receivers what to do when they fail, and crucially it checks *alignment*: it is not enough for SPF or DKIM to pass for some unrelated domain, they have to pass for the domain in the visible `From` address. This is exactly the check that kills ordinary spoofing, where an attacker slaps a trusted `From` on mail sent from unaligned infrastructure. My message aligns perfectly, because the `From` domain and the signing domain and the authorized sender are all the same real, trusted entity. SPF pass, DKIM pass, DMARC pass, aligned on all counts.

So picture the message landing. It is authenticated. It is aligned. It comes from a domain the recipient has received legitimate mail from before, one their mail provider already delivers to the inbox rather than the spam folder, one their security awareness training has probably taught them to treat as safe. There is no red "external sender" banner, because to the receiving system it is not a spoof, it is the real TalentForge. Every signal a careful human or a tuned filter uses to catch phishing says "this is fine." The only thing wrong is the content, and the content is the one thing the platform does not police.

This is the uncomfortable asymmetry at the center of the report. An ordinary attacker spends enormous effort trying to look like a trusted sender and usually fails the authentication checks. I skipped all of that by letting the trusted sender compose and send the message for me. The platform's months of domain warming became my delivery guarantee.

## Force multipliers: HTML and headers

A plain-text lure from a trusted domain is already a High-impact problem. Two neighboring findings on the same engagement turned "already bad" into "shaped to spec," and because I always report chains rather than isolated bugs, they belong here.

**Arbitrary HTML, via the template sink.** The platform let a user author the document template that the outbound email is rendered from, and that template path accepted raw HTML and executed it without the escaping applied everywhere else in the product. I have written that stored-XSS-in-a-template finding up on its own, so I will keep it short here: the practical consequence for *this* finding is that the body I send is not limited to plain text. I can build a pixel-perfect HTML email. Real logo, real brand colors, a convincing button, a link whose visible text says one thing and whose `href` says another, the whole visual grammar of a legitimate notification. The content lever is no longer "some text an alert reader might find odd." It is a faithful forgery of the company's own transactional email, sent by the company's own mail service. The link does not even have to look suspicious, because everything around it is real.

**Attacker-influenced headers, via CRLF.** The send procedure also accepted carriage-return and line-feed sequences inside the subject and body fields without stripping them. Injecting `\r\n` into a field that is later composed into a message is the classic header-injection primitive (CWE-93): a newline can end the current header and begin a new one, opening the door to inserting or influencing headers like `Reply-To`, additional recipients, or MIME structure. I am deliberately careful about this one. The API accepted the CRLF and returned success, but I could not confirm from the outside that the downstream mail composition actually honored the injected headers at the SMTP layer, because I had no read access to the mail logs and modern mail libraries often sanitize this on the way out. So I report it as what it is: a confirmed input-acceptance issue and a plausible header-injection vector, rated conservatively on its own, and noted here as a potential amplifier rather than a proven one. A `Reply-To` quietly pointing at attacker infrastructure, riding inside an otherwise legitimate aligned message, would be a particularly nasty amplifier if it fired.

One send button, four levers, a message indistinguishable from the real thing

Trusted identity
from the platform's own
DKIM-signed domain

Recipient
free-text, no allow-list,
no verification

Subject + body
attacker text, verbatim
into the message

+ HTML template
pixel-perfect forgery
of a real notification

+ CRLF headers
possible Reply-To,
extra recipients (caveat)

spf=pass dkim=pass dmarc=pass → inbox

The platform supplies the one thing a real attacker cannot forge, the aligned identity. The attacker supplies everything else.

## What this buys an attacker in the real world

It is easy to file "can send email" under harmless. It stops feeling harmless the moment you write out the scenarios, so I did, because a severity argument that cannot name a victim is just an opinion.

**Business email compromise.** This is the expensive one. BEC works by convincing someone in finance to move money or change payment details, and it lives or dies on the credibility of the sender. A message from a vendor's genuine, authenticated domain, telling a customer's accounts-payable team that banking details have changed, is the holy grail of BEC, and normally the hardest thing to fake. Here I could send it from inside the trusted domain, to any external finance contact I could name, with an HTML body dressed as a routine account notice. The recipient's mail filter waves it through. Their training says the domain is safe. The invoice gets paid to the wrong account.

**Credential phishing against the platform's own users.** Who is most likely to trust a message from TalentForge? TalentForge's own customers. I can send them a perfectly aligned "your session has expired, please re-authenticate" email from the real platform domain, linking to a login page I host that mirrors the real one. Because the mail is legitimately from the platform, it sails past the exact defenses that would stop an external spoof, and it harvests credentials to the very platform that sent it. The tool becomes the delivery mechanism for attacks against its own user base.

**Fake offers, contracts, and notices.** Depending on what the platform sends for its customers, the trusted channel can carry a convincing fake version of whatever it normally carries: a fraudulent offer, a bogus "sign this document" request pointing at a credential-harvesting page, a fake policy notice. The recipient already believes mail from this domain is real and actionable, and that belief is precisely what makes the fake work.

In every case the attacker's cost is near zero (a mid-privileged seat, which on a self-service SaaS may be a trial account) and the victim's defenses are neutralized not by evasion but by endorsement. The platform vouches for the message, and that is a thing no spam filter knows how to argue with.

## Why I rated this High, not Low, and where I held back

The automated harness that first flagged this drafted it as a Medium, and I understand the instinct. There is no memory corruption, no data exfiltration, no broken crypto. "The app sent an email" sounds like a feature working as designed, which is exactly the trap. Abuse-of-functionality findings get under-rated because the individual request is boring. The impact is not in the request; it is in who can make it and what it is pointed at.

I finalized this at High, and the reasoning is additive. A send action merely reachable by a mid-privileged seat is a Low. Add a recipient field with no allow-list and no verification and you have an open relay scoped to one trusted domain, a Medium. Add fully attacker-controlled subject and body and you have a working phishing message wearing the company's identity, a High. Add arbitrary HTML through the template path and you have a pixel-perfect forgery of the company's own mail. The CWE mapping reflects the fusion: CWE-79 for content injection into the message, CWE-451 for interface-level spoofing of the sender's identity, on top of a classic abuse-of-functionality mail-relay pattern. The severity is not any one of those; it is what they become when a single low-trust actor holds all of them at once.

And here is the caveat I put in the report in the same breath, because leaving it out would be dishonest: I confirmed the send fired. I saw the HTTP 200, the `sent` status, and the `sentAt` timestamp, which is how the platform itself records a successful dispatch. What I could not do was pull the outbound mail logs and watch the message leave the building, because my account could not read delivery logs (the status endpoint returned a 403) and outbound email was, by arrangement, disabled during the engagement, which is the right way to test a sending feature without spraying strangers. So I rested the finding on the platform's own success signals and on the mechanics being unambiguous, and I said plainly that end-to-end delivery to a live external mailbox was not directly observed. That caveat lowers my certainty about one link in the chain. It does not lower the severity of the design, because the design is the finding: a trusted channel wired to a low-trust actor with no recipient or content controls is dangerous whether or not my specific test message cleared the last hop.

## Root cause: a trusted channel exposed to a low-trust actor

Strip away the specifics and the root cause is a single sentence. A high-reputation, authenticated sending channel was exposed to a low-trust actor with no constraint on recipient and no constraint on content. Every sub-problem is a facet of that.

- No recipient allow-list or verification, so the "to" address is whoever the operator names.
- No content controls, so the subject and body (and via the template path, the HTML) are whatever the operator writes.
- No separation between authoring a message and dispatching it, and no second identity on the send, so one seat does the whole thing.
- Transactional identity and user-authored content share a channel, so the brand's trust is lent to text the brand never reviewed.

None of those is a coding bug in the narrow sense. The code does what it was told. The flaw is in the trust boundary: the platform treats "send a transactional notification" and "let a user mail arbitrary content to an arbitrary stranger from our trusted domain" as the same operation, when they are categorically different risks that happen to share an implementation.

## How I would fix it

The fixes stack, and I ranked them by how directly they cut the attack rather than by how easy they are to ship.

**Constrain the recipient.** This is the single highest-value control and the one that was entirely missing. Outbound mail should only go to addresses that have earned the right to receive it: a verified contact tied to a legitimate record the sender is actually authorized for, an address inside a domain the tenant controls, or an address that has completed a confirmed-opt-in step before the platform will mail it. The moment the recipient cannot be an arbitrary stranger, the relay stops being a relay. Everything else is damage reduction on top of this.

**Separate transactional identity from user-authored content.** This is the deep fix. Mail that carries the company's full trusted identity should carry only content the company controls: fixed templates, with user data inserted into named, escaped fields, never free-form subject and body and never raw HTML. If users genuinely need to send free-form content, send it from a visibly distinct, lower-trust identity (a subdomain or sender that recipients and filters do not treat as first-party), so that attacker-authored text does not inherit the reputation of the transactional domain. Do not let the brand sign for words the brand never read.

**Restrict content.** Treat subject and body as data, not as mail-composition language. Strip or reject CRLF before anything is composed into a header (this closes the header-injection vector cleanly and cheaply). Disallow raw HTML in user-authored fields; render through a template system that escapes by default, and if rich content is required, pass it through a strict allow-list sanitizer rather than letting it through verbatim.

**Add approval, rate, and velocity controls.** A single seat should not be able to self-provision a record, approve its own outbound action, and send, all without a second identity. Enforce the segregation of duties on the API, not just in the UI. Rate-limit sends per seat and per tenant, and alert on anomalies: a burst of sends to never-before-seen external domains is exactly the signature of this abuse.

**Monitor the thing you are lending out.** Make outbound dispatch auditable to the people who own the domain's reputation. Log recipient, sender seat, template, and a content fingerprint, and make those logs readable, which would also have let me confirm the last hop during the test. You cannot defend a channel you cannot see.

The one line to remember
If your product sends email from a trusted, authenticated domain, the recipient field and the content fields are security controls. An unconstrained send button is an open relay wearing your reputation.

## At a glance

- **Class.** Abuse of functionality / platform-mediated phishing relay. CWE-79 (content injected into the message), CWE-451 (sender and interface spoofing), mail-relay abuse pattern.
- **Who can do it.** Any mid-privileged seat, which on a self-service platform can mean a trial account.
- **What they control.** Recipient (free text, no allow-list), subject, body, and via the template path arbitrary HTML; via CRLF a plausible header-injection vector (reported conservatively).
- **What the platform supplies.** The one thing an outside attacker cannot forge: a verified, SPF and DKIM aligned, DMARC-passing sending identity with warmed deliverability.
- **Observed.** HTTP 200, document status `sent`, `sentAt` set. End-to-end delivery not directly observed (mail logs not readable; outbound disabled during testing), and the report says so.
- **Severity.** High. Elevated from a Medium harness draft because it fuses a trusted-domain send with fully attacker-controlled recipient and content.
- **Root cause.** A high-trust channel exposed to a low-trust actor with no recipient or content controls and no separation between authoring and sending.

## Closing thought

The findings that frighten defenders most are rarely the ones with a clever payload. They are the ones where the attacker never did anything the system considers wrong. I filled in a form, clicked send, and the platform leaned its entire hard-won reputation behind a message I wrote to a stranger I chose. There was nothing to patch in the sense of a broken line of code. What was broken was the assumption that the person pressing the send button deserved to borrow the company's identity to do it.

When you build a product that speaks in your customers' voice, every field that shapes that speech is part of your attack surface. The recipient list is a security control. The subject line is a security control. The body is a security control. Treat them like one, or someone will treat them like a gift.

### Does your product send email on someone else's behalf?

Imperonlabs tests the features, not just the flaws. We map every channel where your platform lends its identity to user-supplied content, then show you where that trust can be pointed at the wrong person. Book a scoping call and we will find your send buttons before someone else does.

[Request an assessment](https://imperonlabs.com/#contact)
