# Field Notes — Security Research

Offensive-security research, vulnerability disclosures and anonymized case studies by
**[Yash Pandya](https://github.com/bewithyash)** / **[Imperon Labs](https://imperonlabs.com)**.

Full archive (rendered): **[blog.imperonlabs.com](https://blog.imperonlabs.com)** · markdown sources live in [`/posts`](./posts).

> All client case studies are anonymized. Vulnerability disclosures are published after fix / coordinated timelines.

## Vulnerability research & disclosures (CVE / GHSA)
- **A Signed Int That Wrapped the Allocator — CVE-2026-104016** (Jackson/protobuf string-length overflow) · [post](https://blog.imperonlabs.com/jackson-protobuf-string-length-overflow-cve-2026-104016.html)
- **The Same Overflow, One Function Over — CVE-2026-110747** (protobuf map-key overflow) · [post](https://blog.imperonlabs.com/jackson-protobuf-map-key-overflow-cve-2026-110747.html)
- **Morphing a Grayscale Frame Into an RGBA One** (ImageMagick heap overflow — GHSA-x48p-6655-8qgv) · [post](https://blog.imperonlabs.com/imagemagick-morph-heap-overflow-channel-map.html)
- **A Single White Pixel That Read Memory It Shouldn't** (ImageMagick OOB read — GHSA-5222-mhv9-26rg) · [post](https://blog.imperonlabs.com/imagemagick-minimum-bounding-box-oob-read.html)
- **The Overflow the Allocator Could Not See** (ImageMagick 32-bit overflow — GHSA-7438-vh5v-v52h) · [post](https://blog.imperonlabs.com/imagemagick-convex-hull-32bit-integer-overflow.html)
- **When a Drawing Option Becomes a Drawing Command** (ImageMagick MVG code injection — GHSA-qq8x-fq3x-43h4) · [post](https://blog.imperonlabs.com/imagemagick-mvg-encoder-code-injection.html)

## Access control & authorization
- **The Server Knew Who I Was. It Never Asked Whether the Data Was Mine.** (cross-tenant BOLA) · [post](https://blog.imperonlabs.com/cross-tenant-bola-multitenant-fintech.html)
- **A Login Is Not a Permission** (broken authorization, scheduling API) · [post](https://blog.imperonlabs.com/broken-authorization-workforce-scheduling-api.html)
- **BFLA: How a Read-Only Employee Account Exposed an Entire Workforce** · [post](https://blog.imperonlabs.com/bfla-employee-read-entire-directory.html)
- **BOLA: The Print Button That Skipped the Bouncer** · [post](https://blog.imperonlabs.com/bola-the-print-button-that-skipped-the-bouncer.html)
- **Breaking an Approval Workflow: Perfect Authorization, No Spine** (segregation of duties) · [post](https://blog.imperonlabs.com/breaking-an-approval-workflow-segregation-of-duties.html)

## Authentication & account takeover
- **The Fastest Account Takeover Is the One Where You Never Log In** (unauth ATO, fintech API) · [post](https://blog.imperonlabs.com/unauthenticated-account-takeover-fintech-api.html)
- **A Four-Digit Code Is Ten Thousand Doors** (OTP brute-force → MFA bypass) · [post](https://blog.imperonlabs.com/otp-brute-force-to-mfa-bypass.html)

## Injection & client-side
- **Stored XSS to Admin Account Takeover: One Raw-HTML Block Was All It Took** (CVSS 9.0) · [post](https://blog.imperonlabs.com/stored-xss-raw-html-template-to-admin-takeover.html)
- **Stored XSS to ATO: A Support Widget Handed Me Every Parent's Account** · [post](https://blog.imperonlabs.com/stored-xss-to-account-takeover-support-widget.html)
- **A Note From an Employee, a Cookie From the Boss** (stored XSS via notes) · [post](https://blog.imperonlabs.com/a-note-from-an-employee-a-cookie-from-the-boss.html)
- **SSRF Is Never the Finding. Where It Goes Is the Finding.** · [post](https://blog.imperonlabs.com/ssrf-that-matters.html)
- **Turning a SaaS Into a Trusted-Domain Phishing Relay** · [post](https://blog.imperonlabs.com/turning-a-saas-into-a-trusted-domain-phishing-relay.html)

## Infra, mobile & misc findings
- **From a Locked-Down Citrix Desktop to a Domain Shell** · [post](https://blog.imperonlabs.com/from-a-locked-down-citrix-desktop-to-a-domain-shell.html)
- **Auditing a React Native App: What the Binary Gave Up** · [post](https://blog.imperonlabs.com/auditing-react-native-binary.html)
- **The Privilege Escalation That Wasn't** (honest negative) · [post](https://blog.imperonlabs.com/privilege-escalation-that-wasnt.html)
- **Retesting a Fixed Program: Verifying the Patch, Finding the Gap** · [post](https://blog.imperonlabs.com/retesting-a-fixed-program.html)

## Methodology & field guides
- **How I Actually Review a Fintech Architecture** · [post](https://blog.imperonlabs.com/how-i-review-a-fintech-architecture.html)
- **Business Logic Flaws: The Bugs Scanners Never Find** · [post](https://blog.imperonlabs.com/business-logic-flaws.html)
- **HTTP Request Smuggling: A Practical Field Guide** · [post](https://blog.imperonlabs.com/request-smuggling-2026.html)
- **Hunting RCE in Modern Stacks** · [post](https://blog.imperonlabs.com/rce-hunting-modern-stacks.html)
- **Testing Mobile Apps: A Working Methodology** · [post](https://blog.imperonlabs.com/mobile-app-methodology.html)
- **Fuzzing That Finds Bugs, Not Just Crashes** · [post](https://blog.imperonlabs.com/fuzzing-that-finds-bugs.html)
- **Threat Modeling That Ships** · [post](https://blog.imperonlabs.com/threat-modeling-that-ships.html)
- **Securing Fintech and Payment Platforms: An Offensive Checklist** · [post](https://blog.imperonlabs.com/securing-fintech-payments.html)

---
<sub>© Imperonlabs LLC · research & disclosure: <code>security@imperonlabs.com</code></sub>
