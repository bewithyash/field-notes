---
title: "Hunting RCE in Modern Stacks"
date: August 2026
source: https://blog.imperonlabs.com/blog/rce-hunting-modern-stacks.html
---

Web · Methodology

# Hunting RCE in Modern Stacks

By Imperonlabs Research · August 2026 · 14 min read

The days of typing a semicolon into an input and getting a shell are mostly gone. Direct OS command injection still exists, but the modern remote-code-execution surface has moved up the stack into the abstractions we build on: the template engine, the deserializer, the package manager, the file parser. The code still runs. It just runs somewhere less obvious. This is where I look.

Scope note. Every payload below is a benign proof primitive: an arithmetic marker, a DNS callback, a timed sleep. Prove execution, then stop. You never need a reverse shell on someone else's box to demonstrate RCE, and reaching for one is how a clean finding turns into an incident.

When I hunt RCE now I do not go looking for a command line. I go looking for the places where the application hands my input to something powerful and trusts it to be data. Four sink classes cover the overwhelming majority of what I find. Learn to recognise each one and the same instinct carries across languages and frameworks.

## Server-side template injection

Template engines are everywhere: email and notification bodies, PDF and report generation, customizable dashboards, white-label theming, "insert your name here" personalization. The trouble starts when a template feature is exposed to a low-privilege user, because a template language is a programming language wearing a costume.

The probe is famous for a reason. Send an arithmetic expression in the engine's syntax and see if it evaluates. If a field reflects `49` back after you submit `{{7*7}}`, that field is not rendering your text, it is executing it. From there the path depends on the engine: Jinja hands you Python object traversal, FreeMarker exposes Java, ERB is straight Ruby. The critical realisation is that the vulnerable input is often not labelled as a template at all. It is an email subject, a display name, an invoice note, a filename in a generated document.

```
# Detection markers, per engine. Look for the product, not the literal.
{{7*7}}          -> 49     Jinja2 / Twig / Nunjucks
${7*7}           -> 49     FreeMarker / Thymeleaf-ish
<%= 7*7 %>       -> 49     ERB (Ruby)
#{7*7}           -> 49     Ruby / Slim
{7*7}            -> 49     some minimal engines

# Confirm execution out-of-band before touching the object graph:
{{ self.__init__.__globals__.__builtins__ }}    # Jinja recon, read-only
# then a DNS/HTTP callback primitive to prove code runs. Stop there.
```

Four sink classes, one instinct: input reaching power

Template injection
Look where: notification, PDF, theming. Prove: {{7\*7}} evaluates, then a callback.

Insecure deserialization
Look where: cookies, caches, queues. Prove: a gadget that pings out-of-band.

Dependency and build chain
Look where: internal package names, postinstall. Prove: a DNS beacon from install.

File-processing pipeline
Look where: image, doc, archive handlers. Prove: remote fetch or entity resolution.

The pattern is identical across all four: untrusted input reaches a component that treats it as instructions instead of data. Prove it with a benign primitive and the finding writes itself.

## Insecure deserialization

Any time an application stores structured state where the client can touch it (a serialized object in a cookie, a cache entry, a message on a queue, a hidden form field) ask what happens when that blob comes back in. If the server deserializes it into live objects without validating it, the mere act of deserializing untrusted input is the vulnerability. This spans ecosystems: Java's native serialization and its gadget chains, .NET's BinaryFormatter and friends, Python's pickle, and unsafe object hydration in Node. You do not need to write a gadget chain from scratch, the tooling and known chains are mature. Your job is to find the entry point and prove reachability.

The tell is the format. A base64 blob that decodes to `rO0` is Java. A leading `AAEAAAD` pattern is .NET. Python pickle has its own opcodes. Once you spot a serialized object crossing a trust boundary, the question is not whether it is dangerous, it is whether the server hydrates it before checking it.

## Dependency and build-chain RCE

Increasingly the code that runs on the target is not code the target wrote. Dependency confusion is the clean example: if an app installs an internal package by a name that is not claimed on the public registry, publishing that name publicly can get your package pulled into their build, and a package's install scripts run at install time. That is code execution inside their pipeline, not their runtime, which is often even more valuable.

Poisoned postinstall scripts, typosquatted dependencies, and simply outdated libraries with known RCE all live in the same neighbourhood. I fingerprint the exact versions of everything the app ships (lockfiles, response headers, static asset hashes, error pages) and cross-reference against known advisories. An outdated deserialization or template library with a public exploit is RCE you can often demonstrate without touching the app's own logic at all.

## File-processing pipelines

Upload a file and something parses it, and parsers are where memory-unsafe and feature-rich code meets attacker input. Image libraries that resolve remote resources or shell out to delegates. Document parsers that process external entities. Archive extractors vulnerable to path traversal that write files outside the intended directory (the Zip Slip class). Office and PDF pipelines that fetch remote content referenced inside the document. The proof primitive here is usually out-of-band: craft an input that makes the server fetch a URL you control or resolve an external entity, and a single inbound request confirms the parser is doing your bidding.

## Prove it, then stop

The methodology underneath all four is the same. Inventory every component that touches untrusted input: template renderers, deserializers, dependency resolvers, file parsers. Fingerprint exact versions so you know which known issues are even in play. Then demonstrate execution with the smallest, most benign primitive that removes all doubt. A DNS or HTTP callback from the target's infrastructure. A measurable sleep that proves timing control. An arithmetic result that could only come from evaluation. One out-of-band ping from a server that should never have made it is a complete, undeniable demonstration of RCE, and it is a far better finding than a reverse shell you now have to explain to legal.

## How to approach RCE on any target

1. Map every component that consumes untrusted input: templates, deserializers, dependency resolution, file parsers.
2. Fingerprint exact versions and cross-reference known advisories before you throw a single payload.
3. Probe template fields with arithmetic markers, including fields that are not labelled as templates.
4. Find serialized objects crossing trust boundaries and check whether they are hydrated before validation.
5. Hunt internal package names, install scripts, and stale libraries in the build chain.
6. Feed parsers crafted files and watch for out-of-band fetches or entity resolution.
7. Prove execution with a benign primitive, capture the callback, and write it up. Do not escalate to a shell.

The one line to remember
Modern RCE is rarely a command in a shell. It is input reaching a template, a deserializer, a package, or a parser that mistakes it for instructions. Find the abstraction that trusts you, prove it with one benign ping, and stop.

### How much of your code that runs in production did you actually write?

Imperonlabs hunts RCE where it really hides in modern stacks: template engines, deserialization sinks, the dependency and build chain, and file-processing pipelines. Book a scoping call and we will map the execution surface you cannot see.

[Request an assessment](https://imperonlabs.com/#contact)
