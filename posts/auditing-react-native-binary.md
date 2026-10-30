---
title: "Auditing a React Native App: What the Binary Gave Up"
date: August 2026
source: https://blog.imperonlabs.com/blog/auditing-react-native-binary.html
---

Mobile

# Auditing a React Native App: What the Binary Gave Up

By Imperonlabs Research · August 2026 · 8 min read

People treat a mobile binary like a safe to be cracked. It is closer to a map that folds out and tells you where everything is. Ship a React Native app and you ship the JavaScript that runs it, and with it every base URL and endpoint the app will ever call. The audit that matters is not the one that breaks the binary. It is the one that follows the binary to the API.

Scope note. Generalized from a real engagement against a production React Native app built on Hermes, tested on both Android and iOS. Findings ranged from informational to medium, and the medium only existed as a chain. Names and identifiers withheld.

## The static pass is mapping, not magic

A React Native app compiles its JavaScript to Hermes bytecode and drops it in the package as `index.android.bundle`. That file is the app's brain, and it comes home with you the moment you pull the APK or IPA. Disassemble the bytecode and you recover the logic, and more importantly the API surface: every base URL, every route, every field name the client knows about, laid out in constants and string tables.

That is the point of the static pass. Not to find the one magic secret hardcoded in the bundle, though those turn up too, but to draw the map so the dynamic pass knows exactly where to go. Alongside the surface, three platform-hygiene issues surfaced on the Android side:

- `allowBackup="true"` in the manifest, which lets application data be pulled off an unprotected device over ADB backup.
- Certificate pinning present, but a claim that needs to be checked at runtime rather than trusted from the manifest.
- A secure-storage wrapper whose configuration needed a closer look before anyone called it secure.

None of those is a headline on its own. They are notes in the margin of the map, and they matter later.

## The dynamic pass, where the API is the target

With the surface mapped, we ran the app on an instrumented device behind an intercepting proxy. Pinning was in the way, so we removed it. Not with a toolkit of scripts, just a single Frida hook that forces the trust evaluation to return success. Once the app trusts our proxy, the traffic is ours, and the real work begins.

```
Java.perform(function () {
  var TMF = Java.use("javax.net.ssl.TrustManagerFactory");
  // hook the platform trust check and force it to pass,
  // so the pinned client accepts our proxy certificate
  // ...install an all-trusting TrustManager, reset SSLContext...
});
```

From there the audit is an API audit that happens to have a mobile front end. We worked object-level authorization, function-level authorization, business logic, and token handling against the exact endpoints the static pass had already named. The client stopped being the target and became the lens.

A word on the pinning itself. Its absence, or its trivial bypass, is not by itself a high-severity finding. A determined attacker analyzing their own device was always going to see their own traffic. Pinning raises the cost of casual interception, it does not make the client a trust boundary. Reporting a one-line bypass as a critical would be dishonest, and triage teams know it.

The funnel: from binary to the finding that matters

Static
disassemble Hermes, map base URLs, routes, storage, manifest.

Access
one Frida hook defeats pinning, proxy sees plaintext.

API
BOLA, BFLA, logic, tokens on the mapped surface.

Honest severity
rate what you proved, not what you could scare with.

Static maps the surface, dynamic reaches the API, and honest severity keeps the report worth reading. Skip any layer and the audit gets thinner.

## The secure-storage flag that mattered

Back to that margin note. The secure-storage wrapper was initialized with device authentication not required. In plain terms, the keys that protect stored session material were reachable without the user re-authenticating. On its own, a configuration nit. Combined with `allowBackup="true"`, it becomes a chain: physical access to an unlocked device leads through backup to extracted session material, with no biometric or PIN gate standing in the way. That compounding is the whole reason this rated medium in context. Not one catastrophic flaw, but two ordinary ones that hand off to each other.

## What we deliberately did not inflate

A mobile report earns its credibility by what it refuses to overclaim. We held the line on three tempting non-findings:

- A pinning bypass with no sensitive data behind it. Interesting, not impactful.
- A root or jailbreak detection bypass that the server does not rely on anyway. Defeating a check that guards nothing is not a finding.
- An `allowBackup` setting when there is no sensitive local data to back up. A hardening note, not a vulnerability.

Each of those becomes real only when you can demonstrate the exposure it enables. Until then it stays a recommendation, clearly labeled as one.

## How to fix it

- Set `allowBackup="false"` and exclude sensitive files from any backup rules that remain.
- Configure secure storage to require device authentication before keys are released, so a locked device is actually a locked device.
- Keep pinning, but treat its bypass as expected. Never let a client-side check that users can disable be the thing standing between an attacker and your data. Enforce on the server.

## The lesson

A mobile audit is a funnel. Static analysis maps the surface, dynamic analysis reaches the API, and honest severity is what makes the whole report trusted. Platform-hygiene issues are multipliers, not headlines. They turn a medium into a high when they chain, and they stay informational when they do not. The headline findings almost always live in the API the binary quietly reveals, which is why the binary is best understood as the first page of the map, not the destination.

The one line to remember
The binary is not the target, it is the map. Follow it to the API, chain the hygiene issues honestly, and rate what you proved rather than what you could imply.

### Does your mobile app hand attackers a map of your API?

Imperonlabs disassembles the binary, reaches the endpoints behind it, and rates every finding by what we can actually demonstrate. Book a scoping call and we will show you what your app gives up.

[Request an assessment](https://imperonlabs.com/#contact)
