---
title: "SSRF Is Never the Finding, Where It Goes Is the Finding"
date: August 2026
source: https://blog.imperonlabs.com/blog/ssrf-that-matters.html
---

Web · Deep Dive

# SSRF Is Never the Finding. Where It Goes Is the Finding.

By Imperonlabs Research · August 2026 · 16 min read

A DNS callback in your interaction listener proves one thing. It proves the server can be aimed. That is a capability, not an impact, and a mature triage team knows the difference. The report that pays is the one that shows what the server reached after you aimed it.

Scope note. Everything below uses example domains and lab targets. If you want a payload catalogue to test against, the PayloadsAllTheThings SSRF list is the reference to keep open in a second tab.

Most SSRF write ups stop at the callback. The screenshot shows a request landing on the tester's collaborator, the title says "SSRF", and the impact section says "an attacker can make the server send requests." True, and almost worthless. The server sends requests all day. The question that decides severity is whether one of those requests touches something the internet was never supposed to touch.

So treat the callback as the start of the work, not the end of it. The rest of this note is how I turn a fetch into a finding.

## First, find every place the server fetches something

The obvious sinks are the ones that look like a URL. A webhook field, an avatar or image import by link, a PDF or screenshot generator, a link preview, an "import from URL" feature, a document converter. Test every one of them, because each fetcher can behave differently.

The sinks that actually pay are the ones that do not look like a URL at all. XML parsers that resolve external entities. A PDF library that loads remote images referenced inside the document. A markdown renderer that fetches embedded assets. A file upload that gets processed by ImageMagick or ffmpeg, both of which will happily reach out over the network if you feed them the right container. An SSO flow that fetches a metadata document from a URL you influence. When you inventory the app, do not ask "where can I paste a link." Ask "where does the server go get something on my behalf," because that set is much larger.

## Learn the shape of the fetcher before you attack it

Before you throw payloads, spend five minutes fingerprinting the thing. Three questions decide everything that follows.

Does it follow redirects? Point it at a host you control that answers with a 302 to an internal address. If the server follows, your allowlist bypass is often just a redirect away.

Which schemes does it accept? Try `http`, then `https`, then the ones that matter: `gopher`, `dict`, `file`, `ftp`. A fetcher that speaks gopher is a fetcher that can talk to a raw TCP service, and that is the difference between reading a page and writing to a database.

What comes back? Full response, a thumbnail, a status code, a timing difference, or nothing at all. This tells you whether you are working a full read SSRF or a blind one, and it changes your whole plan.

Escalation, in order of what it is worth

1 Cloud metadata
credentials, tokens, user data. Instant crown jewel.

2 Internal services
admin panels, dashboards, unauthenticated APIs on the LAN.

3 Raw TCP via gopher
write to Redis, Memcached, unauthenticated Postgres.

4 Reach only
a bare callback. Real, but the weakest version of the bug.

Work top down. The same SSRF is a Low or a Critical depending on how far up this ladder you can climb.

## Escalation, in order of what it is worth

### Cloud metadata first, always

If the target runs in a cloud, the metadata endpoint is the fastest path from callback to crown jewel. On AWS it lives at `169.254.169.254`, and on an IMDSv1 instance a plain GET to the security credentials path hands you temporary keys. GCP, Azure and Alibaba all expose their own metadata services on the same link local address with slightly different paths and, in GCP's case, a required header. Try each provider's layout, because a fetcher that adds custom headers for you can sometimes satisfy the header requirement that is supposed to protect the endpoint.

IMDSv2 raises the bar by requiring a PUT to fetch a token first, which a naive GET only SSRF cannot do. That is exactly why the redirect and gopher questions from earlier matter. A fetcher that follows redirects or speaks a request smuggling friendly scheme can sometimes still get there.

### Internal services next

No metadata, or a hardened one? Turn the server into a scanner. Sweep `127.0.0.1` and the private ranges, watch for timing and status differences, and map what answers. Internal admin panels, monitoring dashboards, message queues and service APIs are routinely deployed with no authentication because "only the internal network can reach them." Your SSRF is now on the internal network.

### Gopher into Redis, the example that changes minds

This is the one that moves a report from interesting to urgent. If the fetcher speaks gopher and an unauthenticated Redis is listening internally, you are no longer reading, you are writing. Redis accepts inline commands, so you encode a sequence of commands into a gopher URL, carriage returns and line feeds and all, and the server delivers them to Redis for you. From there the classic paths are writing a cron job, dropping a webshell into a known web root, or poisoning a cache. The point for the report is simple. You did not just reach an internal service. You executed commands against it.

## Beating the filter without hating the developer

Blocklists that try to ban "internal" addresses are a losing game, and knowing why helps you bypass them cleanly. The same address can be written as decimal, as hex, as octal, with an IPv6 mapping, or hidden behind a hostname whose DNS you control. The userinfo trick, where `http://[email protected]/` parses as a request to the metadata service while looking like the allowed host, still catches parsers that split on the wrong character. DNS rebinding beats a validator that resolves a name once, approves it, and then fetches it a second time after your record has flipped to an internal IP. And parser differentials, where the validator and the HTTP client disagree about where the host in a URL actually ends, are the gift that keeps giving. You are not attacking the developer here. You are attacking the gap between two libraries that both think they parse URLs correctly.

## Blind SSRF is still worth your time

No response body does not mean no impact. A timing oracle tells you open ports from closed ones. An error based oracle leaks a few bytes of an internal response inside a parser error. And plenty of blind SSRF still reaches a metadata endpoint or an internal write target where the effect does not need to come back to you to be real. Writing a cron job through a blind gopher payload works whether or not you ever see the response.

## Writing it up so it survives triage

The report that gets paid at the right severity tells a story in one breath. Here is the input, here is the sink, here is the internal thing it reached, and here is what that let me do. Show the metadata response with the credential redacted. Show the Redis write landing. Do not make the triager imagine the impact, hand it to them. A callback screenshot asks them to trust you. A retrieved credential does not.

## How to fix SSRF properly

Allowlist the destinations, do not blocklist them. Resolve the hostname yourself, validate the resolved IP against the allowlist, and then connect to that exact IP so a rebind cannot slip in between the check and the fetch. Disable the schemes you do not need, so no gopher, no file, no dict. Strip and refuse redirects unless the feature truly requires them. And put the fetcher on a network segment that cannot reach the metadata endpoint or the internal control plane in the first place, because defence in depth is what saves you when the parsing check fails.

## How to approach SSRF on any target

1. Inventory every sink, the URL shaped ones and the hidden ones inside parsers and file processors.
2. Fingerprint the fetcher: redirects, schemes, and what comes back.
3. Go for cloud metadata first if the target is in a cloud.
4. If not, sweep internal ranges and map what answers.
5. If a raw TCP service is reachable and gopher works, escalate from read to write.
6. Write it up as a chain, from input to concrete impact, with the evidence in hand.

The one line to remember
Nobody pays for a server that can send a request. They pay for the credential, the internal write, or the command you reached with it. Chase the destination, not the callback.

### Is your attack surface hiding a fetch you have not mapped?

Imperonlabs runs offensive assessments that chase findings to real impact, not to a callback. Book a scoping call and we will show you where your servers go on your behalf.

[Request an assessment](https://imperonlabs.com/#contact)
