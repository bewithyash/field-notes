---
title: "HTTP Request Smuggling: A Practical Field Guide"
date: August 2026
source: https://blog.imperonlabs.com/blog/request-smuggling-2026.html
---

Web · Methodology

# HTTP Request Smuggling: A Practical Field Guide

By Imperonlabs Research · August 2026 · 17 min read

Request smuggling is one of the last bug classes where a single request can compromise other users without ever touching their session. It works because two machines on the same connection disagree about where your request ends. The front end thinks it stops here. The back end thinks it stops there. The bytes in between become the opening of somebody else's request.

Discipline note. Never confirm smuggling by breaking a shared connection on production. You can wedge the request queue for real users. Everything below is built to be provable against your own traffic and a benign marker, with timing probes preferred over anything destructive.

## The disagreement at the heart of it

HTTP/1.1 gives you two ways to say how long a message body is: `Content-Length`, which states a byte count, and `Transfer-Encoding: chunked`, which streams sized chunks until a zero-length terminator. The spec says that when both appear, `Transfer-Encoding` wins and, honestly, that a well-behaved server should reject the ambiguity outright. Real deployments do neither consistently. When the front-end proxy and the back-end server resolve that ambiguity differently, you get smuggling, and it comes in three classic shapes.

- **CL.TE**: the front end honours `Content-Length`, the back end honours `Transfer-Encoding`.
- **TE.CL**: the reverse, front end chunked, back end counting bytes.
- **TE.TE**: both understand chunked, but one can be tricked into ignoring the header by obfuscating it, with a leading space, a tab, or a duplicate declaration, so effectively only one parser applies it.

In every case the trick is the same. You send bytes the front end counts as body, so it forwards them as part of your request, but the back end has already decided your request ended earlier. Those leftover bytes now sit at the front of the connection, waiting to be glued onto whatever request arrives next.

## Detect it without hurting anyone

The safe way in is timing, not breakage. You craft a request that, if the back end parses it the vulnerable way, forces it to wait for bytes that never arrive, and you measure the delay. A clean server answers instantly. A vulnerable one hangs for your timeout. You have confirmed the disagreement without ever corrupting a queued request. A classic CL.TE timing probe looks like this:

```
POST / HTTP/1.1
Host: target
Content-Length: 4
Transfer-Encoding: chunked

1
A
X
```

The front end sees `Content-Length: 4` and forwards four bytes. The back end sees chunked, reads the `1`-byte chunk, then blocks waiting for the next chunk that your four bytes never completed. A consistent, significant delay is your signal. Once timing says a host is worth pursuing, you confirm with a self-contained payload that only affects your own next request: smuggle a prefix that flips a harmless routing decision or echoes an observable marker header back to you. If your own follow-up request comes back wearing the marker, the smuggle landed, and you proved it on your own traffic.

Who counts, who chunks

CL.TE
front end reads Content-Length
forwards the whole body
back end reads Transfer-Encoding
stops at chunk 0, leftover queued
leftover = start of next request

TE.CL
front end reads Transfer-Encoding
forwards chunked stream
back end reads Content-Length
stops at N bytes, rest queued
leftover = start of next request

Two parsers, one connection, opposite rules. The bytes one forwards and the other ignores become the prefix of the next victim's request.

## What is worth chasing in 2026

Plain CL.TE and TE.CL still exist, but the interesting surface has moved to the protocol seams. These are the variants I prioritise now.

**HTTP/2 downgrade smuggling (H2.CL and H2.TE).** Many edges speak HTTP/2 to the client and rewrite to HTTP/1.1 for the origin. If that rewrite naively reconstructs `Content-Length` or trusts a smuggled length, the downgrade itself introduces the disagreement even though HTTP/2 was supposed to be immune to this class. The bug is not in either protocol, it is in the translation.

**CL.0 and its H2 cousin.** Some back ends ignore the body entirely on certain endpoints (static files, redirects, some error paths), treating every request there as zero-length regardless of headers. Point the technique at one of those endpoints and the body you sent becomes a free-standing smuggled request.

**CRLF injection across the downgrade.** An HTTP/2 pseudo-header or header value that carries a hidden carriage-return line-feed is harmless inside HTTP/2, but when the edge downgrades it to HTTP/1.1 that CRLF splits the request in two. Header smuggling becomes request splitting on the far side of the translation.

## The impact ladder, and how to prove each rung safely

### Access control bypass

The smuggled prefix reaches an internal path the front-end proxy is supposed to block but the origin happily serves. You prove this against your own connection: smuggle a request for the restricted path and observe your own follow-up receiving its response. Provable, benign, and clean.

### Cache poisoning

Combine request splitting with an unkeyed input and you can make the cache store a response under a key that other visitors will hit. The safe demonstration is to poison a benign, non-shared marker path and show it served back, proving the mechanism without polluting a page real users depend on.

### Traffic interception

The high-severity rung: a smuggled prefix that appends the next victim's request onto a storage or echo endpoint, harvesting their tokens and PII. You do not perform this against real users. You demonstrate it structurally: show that your own follow-up request gets captured by your prefix, and explain the chain, without ever collecting a real user's data.

## Running it responsibly

Rate-limit yourself, prefer timing detection over anything that touches the shared queue, never park a payload that could capture live traffic, and coordinate your test window with the program before you go near a production edge. Smuggling is powerful precisely because it crosses into other users' connections, which is exactly why the proof has to stay on your own.

The one line to remember
Smuggling is a parser disagreement, not a payload. Find the disagreement with timing, prove it on your own traffic with a benign marker, and describe the reach to real users without ever touching them.

### Does your edge and origin agree on where a request ends?

Imperonlabs tests the seams between your proxies, your HTTP/2 downgrade, and your origin, safely, and reports the ones that disagree. Book a scoping call.

[Request an assessment](https://imperonlabs.com/#contact)
