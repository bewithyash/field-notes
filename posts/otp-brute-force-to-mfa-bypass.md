---
title: "A Six-Digit Code Is a Million Doors: OTP Brute-Force to MFA Bypass"
date: October 2026
source: https://blog.imperonlabs.com/blog/otp-brute-force-to-mfa-bypass.html
---

Web · Auth · MFA Bypass

# A Six-Digit Code Is a Million Doors: OTP Brute-Force to MFA Bypass

By Yash Pandya · October 2026 · 14 min read

A one-time code is a hallway of doors. A six-digit code is a million of them, a four-digit code only ten thousand. People treat that number like a lock. It is not a lock. The number of doors never protected anything on its own. What protects you is that someone stops me after I have knocked on a handful of them. On the e-signature platform in this story, no one stopped me. So I knocked on all ten thousand doors, found the one that opened, and walked straight through the second factor into another person's account.

**Scope note.** This is a sanitized account of a real, authorized engagement against a digital-signature platform. The product, company, domains, routes, and values are invented. I use "InkSeal" on `sign.example.com` throughout. The mechanics, the arithmetic, the CWE, and the severity are reproduced exactly from the finding. There are no screenshots and nothing here points at a live system.

## The hook: a second factor that could not count

Most authentication bugs I write up are about something clever: a confused parser, a signature that was never checked, a trust boundary drawn in the wrong place. This one is almost embarrassing in how simple it is, which is exactly why it keeps shipping. A second factor built on a short numeric code has precisely one job beyond generating the code, and that job is to stop you from guessing. Counting failed attempts is not a feature of the control. It *is* the control. Remove it and you are left with a theater prop that looks like multi-factor authentication and behaves like a turnstile with no attendant.

InkSeal, the platform I was testing, is a digital-signature service. People use it to put legally meaningful signatures on documents. The second factor sat right in the login flow, which is the highest-value place it could possibly sit, and it had no attempt counter at all. Not a weak one. Not a generous one. None. I will walk through exactly where it sat, what it was supposed to defend, the arithmetic of why the missing control is the whole ballgame, and how the fix is less about the code and more about the counter.

## The target, and where the OTP sits in the flow

InkSeal is a single-page web app talking to an API behind a gateway. Login is deliberately two-step, which is the correct instinct. The first step takes an email and a password. The second step takes a one-time code delivered out of band, over SMS or email, and only when both steps succeed do you receive a session token. Cleaned up and renamed, the two calls are these:

```
POST /api/auth/login     { email, password }        -> triggers a one-time code, returns a login reference
POST /api/otp/verify     { email, code, reference }  -> validates the code, returns an access_token
```

The code is **four digits**. Four digits means the set of all possible codes is `0000` through `9999`, which is exactly ten thousand values. Hold onto that number, because the entire story is a fight between that number and how many times the server will let me be wrong.

The important structural detail, and the one that quietly decides the outcome, is what the code was bound to. The code was validated against the *account*, keyed by the email and the login reference. It was not bound to a device, a browser session, or the network client that requested it. That means the guesses did not have to come from the same place the code was sent to. They could come from anywhere, from any number of connections, in any order, as fast as I cared to send them. The second factor was a value floating in front of an account, waiting for anyone to name it correctly.

## The threat model: what the code is actually guarding

Before touching the gap, it is worth being honest about what the second factor is for, because that frames the severity. The first factor is the password. Passwords leak. They are reused across sites, dumped in breach corpora, phished, and sprayed. Any threat model written after about 2012 assumes the password will eventually be known to someone who should not know it. The second factor exists precisely for the world where the first factor has already failed. It is the backstop.

So the relevant attacker is not some genius. It is someone who already has a victim's email and password, from a reused-credential dump or a quick phish, and is now standing in front of the backstop. On an e-signature platform, what is behind that backstop is not just an inbox. It is the ability to view documents meant to be signed, to impersonate the account holder, and, depending on the product's rules, to approve or apply a signature in that person's name. The code is the last thing standing between a leaked password and a forged legal signature. That is the stake, and it is why a weakness here is not a hygiene issue. It is a direct path to the worst outcome the product has.

## The control as designed, on paper

Give the design its due, because the shape was right. Two factors. A code delivered out of band, so possession of the password alone is not enough. A short-lived code, so a stolen or shoulder-surfed value does not live forever. A login reference tying the verification back to the attempt that started it. If you sketched this on a whiteboard it would pass review. A reviewer skimming the flow would tick the "MFA present" box and move on.

This is the trap I see again and again. The presence of a control and the strength of a control are different measurements, and audits love to conflate them. A header can be present and permissive. A policy can exist and allow everything. And a second factor can be fully wired up, sending real codes to real phones, while doing nothing to stop someone who simply tries every code. The flow had all the right boxes. It was missing the one invisible thing that makes the boxes matter.

## The gap: the control that could not say "enough"

I did the least imaginative test in the book. I authenticated with a valid password to trigger a code, then I started submitting wrong codes to the verify endpoint and watched what the server did. Here is what it did: nothing. It kept answering.

Specifically, across a run of twenty-five consecutive wrong codes sent back to back, every single response was identical. Same status, same body, same timing. There was:

- **No rate limit.** The verify endpoint never returned a `429`, never slowed down, never inserted a delay. Request number twenty-five came back as fast as request number one.
- **No account lockout.** Repeated failures against one account did not freeze it, did not require a re-authentication, did not invalidate the pending login. The code stayed guessable for its whole life.
- **No attempt counter per code.** A single issued code could absorb an unlimited number of guesses. The server never decided that nineteen wrong answers about one code meant someone was guessing.
- **No CAPTCHA, no step-up, no friction of any kind** after repeated failures.
- **No binding to the client.** As noted, the code answered to the account, not to the device or session that requested it, so guesses could be fanned out across many connections with no penalty.

The failure response looked like this, and it looked like this every time:

```
HTTP/1.1 200 OK
Content-Type: application/json

{ "access_token": null, "token_type": null, "error": "invalid_request", "message": "Invalid Otp" }
```

A correct code, by contrast, returned a real `access_token`. So the oracle was perfect and binary: wrong looks exactly one way, right looks exactly another, and nothing in between ever changes no matter how many times I am wrong. That is the ideal condition for brute force. I do not need a side channel, a timing difference, or any subtlety. I need a loop and a little patience, and the patience turns out to be measured in minutes.

## The arithmetic, which is the whole argument

Security people sometimes wave at "brute force" without doing the multiplication, and the multiplication is the entire case. Let me do it properly, because the numbers are what turn a shrug into a High.

### Keyspace and expected guesses

A four-digit code has a keyspace of `10^4 = 10,000`. Against one issued code that is fixed and valid, my guesses are draws without replacement from that set. The worst case to guarantee a hit is 10,000 guesses. The average case, if the real code is uniformly random, is about 5,000 guesses, because on average the target sits in the middle of the order I happen to try. Success probability after `k` distinct guesses is simply `k / 10,000`. After 1,000 guesses I have a 10% chance; after 5,000, a coin flip; by 10,000 it is a certainty.

### Guesses become seconds

Now convert guesses into wall-clock time, which is where the missing rate limit bites. Suppose I send requests at a polite, deliberately-throttled-on-my-side rate so as not to stress the target:

```
rate        worst case (10,000)      average case (5,000)
10 req/s    1,000 s  ~= 16.7 min     500 s  ~= 8.3 min
20 req/s      500 s  ~=  8.3 min     250 s  ~= 4.2 min
50 req/s      200 s  ~=  3.3 min     100 s  ~= 1.7 min
100 req/s     100 s  ~=  1.7 min      50 s  ~= 0.8 min
```

Those are not exotic rates. Twenty requests a second is a single thread with no cleverness. And because the code answered to the account rather than to a session, I could also fan the work across several connections in parallel and collapse the worst case to well under a minute without ever tripping a per-connection limit, since there was no limit of any kind to trip. The point is not the exact second. The point is that the entire keyspace of this control falls inside a coffee break.

### Where TTL comes in, and why it is not a rescue

The one thing that could have saved the design without a counter is a short code lifetime, the TTL. If the code expires quickly, maybe I cannot get through the keyspace before it dies. So let us ask what rate I would need to cover all 10,000 values inside a given window. The requirement is simply keyspace divided by the TTL in seconds:

```
TTL        rate needed to cover all 10,000 codes
10 min     10,000 / 600  ~= 16.7 req/s
5 min      10,000 / 300  ~= 33.3 req/s
2 min      10,000 / 120  ~= 83.3 req/s
60 s       10,000 / 60   ~= 166.7 req/s
```

Read that table and the problem is obvious. Even a tight two-minute TTL only demands about 83 requests a second to sweep the whole space, and a one-minute TTL demands under 170. Those are trivial numbers for an unthrottled endpoint. A short TTL feels like a control because it has a comforting unit of time attached, but without a rate limit it just sets a speed I need to hit, and the speed is laughably reachable. TTL and attempt-capping are not substitutes. TTL limits how long a given code is worth guessing. The attempt cap limits how many guesses you get inside that window. You need both, and of the two, the cap is the one doing the real work.

### The regeneration trick, and why a per-code cap alone still loses

Here is the part people miss, and it is the reason the fix has to be designed carefully rather than just bolted on. Imagine the team adds the obvious patch: a cap of, say, five wrong guesses per issued code, after which that code is burned. Good instinct. But if I can freely request a *new* code, by re-running the login step, and that regeneration neither counts against the account nor locks anything, then the per-code cap becomes a speed bump I drive over repeatedly.

The math shifts but does not save you. With a cap of `m` guesses per code and unlimited regeneration, each new code is a fresh uniformly random value, and I get `m` shots at it. The chance I hit a given code is `m / 10,000`. Over `R` regenerated codes, my probability of success is:

```
P(success) = 1 - (1 - m/10,000)^R
```

Plug in the "obvious patch" of `m = 5`. To reach a coin-flip chance I need `R = ln(0.5) / ln(1 - 5/10,000)`, which is about **1,386 regenerations**. For 99% confidence it is about **9,200 regenerations**. If requesting a code is free and unthrottled, nine thousand requests is again a matter of minutes. The per-code cap, on its own, bought almost nothing.

There is a mirror-image version of the same trick. Suppose instead the server locked each code hard to one guess, but let me fix my guess and regenerate forever. I pick `0000` and keep asking for new codes, hoping the server eventually generates `0000`. Each regeneration is an independent 1-in-10,000 draw, so my probability after `N` codes is `1 - (1 - 1/10,000)^N`. That reaches a coin flip at about **6,931 regenerations** and 99% at about **46,000**. Slower, but still finite and still cheap when nothing throttles the request side.

The lesson the arithmetic hands you is blunt. The real strength of this control is not the length of the code. It is the smaller of two numbers: how many guesses you get per code, and how many codes you can force the system to issue. If either number is unbounded, the keyspace is cosmetic. On InkSeal both numbers were effectively infinite, which is why the worst case was "a few minutes" and not "never."

## The loop, framed responsibly

I test these responsibly, which in practice means I cap my own request rate, stop the instant I have proven the control is absent rather than riding it all the way to a real account's token, and I never target a value I am not authorized to reach. During the engagement I confirmed the absence of throttling with a small, bounded run and then stopped. For illustration only, here is the shape of the loop against the invented `sign.example.com`, written to be read, not run:

```
# ILLUSTRATIVE ONLY. Rate-limited on the client side, run against an invented host,
# and stopped as soon as the control is shown to be absent.

ref=$(get_login_reference "[email protected]")   # from POST /api/auth/login, triggers one code

for code in $(seq -w 0000 9999); do
    resp=$(curl -s -X POST https://sign.example.com/api/otp/verify \
        -H 'Content-Type: application/json' \
        -d "{\"email\":\"[email protected]\",\"code\":\"$code\",\"reference\":\"$ref\"}")

    if ! echo "$resp" | grep -q '"access_token":null'; then
        echo "code $code accepted: $resp"
        break
    fi
    sleep 0.05           # my own throttle, not the server's. the server has none.
done
```

The thing to notice is how boring this is. There is no payload craft, no encoding, no race, no parser abuse. It is `seq` and `curl`. The sophistication of the attack is zero, which is the most damning thing I can say about the control, because the only thing that was ever supposed to stand between a leaked password and a victim's signature was the server's willingness to say "enough," and it never said it.

## What it bypasses, and the impact

Walk the chain end to end. I start with a password, which the threat model already assumes an attacker can obtain. Normally the second factor stops me cold, because I do not have the victim's phone or inbox and cannot read the code that was sent there. But I do not need to read it. I guess it, in minutes, because nothing counts my failures. The verify endpoint hands me a real `access_token` for an account I have no legitimate access to.

At that point the second factor has been fully bypassed and I am authenticated as the victim. On a digital-signature platform that is account takeover with the sharpest possible edge: the account's reason for existing is to produce signatures that mean something. Depending on the product's signing rules, the next actions range from reading confidential documents staged for signature, to impersonating the account holder in the workflow, to approving or applying a signature in their name. The control that failed was not guarding a comments section. It was guarding the integrity of a legal act.

## The standards, and why this is a clean High

The primary mapping is **CWE-307: Improper Restriction of Excessive Authentication Attempts**. That is the textbook definition of what happened: the system placed no meaningful restriction on how many times an attacker could try to authenticate the second factor. It sits alongside **CWE-287: Improper Authentication**, because the net effect is that authentication can be satisfied by someone who should not be able to satisfy it, and it is the brute-force class squarely (CWE-307 is the brute-force-restriction node in the tree).

I scored it **High**, CVSS 3.1 base **7.5**, vector `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`. A word on the two choices people argue about. `PR:N` models no prior privilege on the system itself; the attacker brings a password obtained out of band, from a dump or a phish, which is exactly the world the second factor is built to survive, so I do not credit the control for the attacker needing the first factor. `AC:L` because, as the arithmetic shows, there is no special condition to engineer and no race to win: a plain loop against an unthrottled endpoint succeeds inside the code's lifetime every time. Confidentiality and integrity are both High because the outcome is full account takeover on a platform whose integrity is its entire product. Availability is untouched, hence `A:N`. If your program weights the business impact of forged signatures, this is the kind of finding that argues its way upward, not downward.

## How to fix it, so the counter is the control

The fix is not one setting. It is a small set of limits that only work together, because, as the regeneration math showed, capping one path just pushes the attacker onto another. Do all of these:

- **Cap attempts per code.** Allow a handful of wrong guesses against a single issued code, then burn that code permanently. Five is a reasonable number. This closes the "hammer all 10,000 against one code" path.
- **Cap attempts and regenerations per account.** This is the one most designs forget. Count failed verifications and code requests against the account and the login reference, not just against a single code, and stop issuing fresh codes after a small total. This is what closes the regeneration trick, where unlimited new codes defeat a per-code cap.
- **Cap per source as a backstop.** Rate-limit by IP and client as defense in depth, understanding that a distributed attacker evades it, which is exactly why it cannot be your only cap.
- **Back off and lock out.** Apply exponential backoff on repeated failures and, past a threshold, invalidate the pending login entirely so the attacker must start over from the password step. A lockout that forces re-authentication resets the attacker's progress to zero.
- **Keep the TTL short, but never alone.** A short code lifetime is good hygiene and shrinks the window, but as the table showed it is only a rate requirement, not a wall. Pair it with the caps above.
- **Invalidate on regeneration.** Requesting a new code must atomically kill the old one, so an attacker cannot keep several live codes in flight and multiply their shots.
- **Lengthen the code.** Move to six digits. It multiplies the keyspace by a hundred, to `1,000,000`, which turns a few-minute sweep into hours and makes the TTL requirement genuinely demanding. Treat this as a cost multiplier, not a fix: a six-digit code with no counter still falls to a patient, fast, regenerating attacker. Length buys time; the counter buys security.
- **Bind the code to the request context.** Tie the code to the device or session that requested it so that guesses from unrelated clients are rejected outright, removing the attacker's freedom to fan out.
- **Monitor and alert on the pattern.** A burst of failed OTP verifications against one account is one of the cleanest attack signatures there is. Detecting and responding to it is the operational half of the fix, and it is nearly free to add.

**At a glance.**
  
**Finding:** Login one-time-code verification with no rate limit, no lockout, and no attempt counter, enabling brute-force of the second factor.
  
**Where:** `POST /api/otp/verify` on a digital-signature platform's two-step login.
  
**Keyspace:** four-digit code, `10,000` values. Observed: 25 consecutive wrong guesses, identical `200` responses, zero throttling.
  
**Time to break:** roughly 1 to 8 minutes at 20 to 100 requests per second, well inside any reasonable code lifetime.
  
**Impact:** complete MFA bypass to account takeover, and on a signing platform, unauthorized signatures in the victim's name.
  
**CWE:** CWE-307, with CWE-287. **Severity:** High, CVSS 3.1 7.5, `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`.
  
**Fix:** caps per code and per account and per source, backoff and lockout, short TTL, invalidate on regeneration, longer codes, device binding, and alerting.

## The thing to carry away

A one-time code protects nothing because of how many values it has. It protects you because something is counting. The keyspace is a speed limit; the counter is the wall. InkSeal built the whole second factor correctly, generated real codes, delivered them out of band, gave them a login reference, and then forgot to count, and that single omission reduced ten thousand doors to a few minutes of knocking. Every control that looks like a lock is only a lock if someone stops you after a few wrong tries. When you are reviewing your own MFA, do not ask whether the code is strong. Ask what happens on the twentieth wrong guess, and the two-hundredth, and whether asking for a fresh code resets the count. If the answer is "nothing happens," you do not have a second factor. You have a speed bump with a good story.

The one line to remember
The security of a one-time code is not its length, it is the counter behind it. If nothing stops the twentieth wrong guess, or the thousandth fresh code, the keyspace is decoration and the second factor is already open.

### Do you know what your MFA does on the twentieth wrong code?

Imperonlabs tests authentication the way an attacker who already has the password does: counting your counters, probing your lockouts, and chaining the gap from a leaked credential to a real takeover. Book a scoping call and we will find the control that only looks like a control.

[Request an assessment](https://imperonlabs.com/#contact)
