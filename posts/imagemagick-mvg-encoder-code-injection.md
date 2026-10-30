---
title: "When a Drawing Option Becomes a Drawing Command: GHSA-qq8x-fq3x-43h4"
date: October 2026
source: https://blog.imperonlabs.com/blog/imagemagick-mvg-encoder-code-injection.html
---

ImageMagick · Injection · MVG

# When a Drawing Option Becomes a Drawing Command

By Yash Pandya · October 2026 · GHSA-qq8x-fq3x-43h4 · 12 min read

MVG (Magick Vector Graphics) is ImageMagick's own little vector drawing language. `push graphic-context`, `fill`, `rectangle`, `text`, `image`: a program in MVG is a sequence of text commands, and ImageMagick's drawing engine executes them. The MVG encoder builds that program by stitching your drawing options into text. The problem this advisory describes is the oldest one in the book, wearing a graphics costume: when an option value is placed into the MVG stream without being neutralized, it can stop behaving as data and start behaving as another command. The attacker's "color" is no longer a color. It is a new line of the program.

Disclosure note. GHSA-qq8x-fq3x-43h4 is a fixed, published ImageMagick advisory credited to me (`bewithyash`), patched in 7.1.2-33 and 6.9.13-58. No CVE was assigned. I want to be upfront about sourcing: unlike my other ImageMagick write-ups this one has no public proof-of-concept or named function in the advisory, and I am not going to invent one. What follows explains the vulnerability *class* the advisory identifies, namely code injection in the MVG encoder via drawing options, with the mechanism described in general terms and every version and rating quoted directly from the published advisory. Where I bring in historical context, I label it as background and date it, so you can always tell what this advisory fixed from what MVG has done to people over the years.

## What MVG is, and why it is a language and not a config

Most image formats are data: a header, some pixels, maybe some metadata. A PNG decoder reads bytes into a pixel buffer and that is the end of its ambitions. MVG is a different kind of thing. It is a textual drawing program that ImageMagick interprets, in the same spirit as SVG, and ImageMagick treats it as a first-class coder: it can read MVG as an input format and it produces MVG internally as the serialized form of a drawing. A small MVG program looks like this:

```
push graphic-context
  viewbox 0 0 200 100
  fill 'blue'
  rectangle 10,10 90,90
  fill 'black'
  text 20,60 'hello'
pop graphic-context
```

Each line is an operator (`fill`, `rectangle`, `text`, `image`, `push`, `pop`) followed by operands. The drawing engine walks the list, mutating drawing state and emitting pixels as it goes. When you run something like `magick -fill blue -draw 'rectangle 10,10 90,90' out.png`, ImageMagick is assembling MVG drawing operations from your command-line options and feeding them to that engine. The MVG encoder is the component that serializes drawing state and operations into the textual command stream the engine consumes.

The reason "it is a language" matters is the same reason SQL injection and template injection matter. Any time a program builds a line of a command language by concatenating a value into it, the boundary between "this part is an instruction" and "this part is a parameter" is enforced by syntax: quotes, delimiters, line breaks, whitespace. If a value can contain those syntactic characters and they are passed through literally, the value can close its own field early and append whatever it likes after. It becomes code in the host language. That host language here is MVG, and MVG has operators that do considerably more than draw shapes. It can set and composite fonts, pull in and overlay other images with the `image` operator, read external resources, and reference the filesystem. That is precisely why injecting commands into it is more than a cosmetic concern: the grammar you are breaking out of has reach.

## How ImageMagick parses and renders MVG

It helps to be concrete about the paths by which a string you did not write ends up being executed as MVG. There are several, and they are easy to conflate.

First, the `-draw` option. Its argument *is* MVG. When an application builds a draw string by interpolating user-supplied geometry, text, or colours into a primitive like `text 20,60 '...'`, the user's value lands directly inside an MVG command. Second, the higher-level drawing options such as `-fill`, `-stroke`, `-font`, `-pointsize`, `-annotate`, and `-caption`. These feel like simple switches, but under the hood they contribute operands to MVG operators that the encoder assembles. A caption string is not painted by magic; it becomes a `text` (or label) operation in the drawing stream. Third, MVG as an input file. ImageMagick decides how to handle a file by content and by explicit format hints, so a file whose bytes are MVG, or a filename carrying the explicit `mvg:` prefix, is parsed as a drawing program rather than as inert pixels. Fourth, format coercion through coders and delegates: ImageMagick is a format multiplexer, and a request to render one format can route through MVG internally depending on the operation.

The thread running through all four is that drawing options and drawing text are not sandboxed values. They are fragments of a program, and whoever controls a fragment is, to the extent of the grammar, writing program text. The encoder's job is to make sure a fragment that is supposed to be a leaf value (a colour, a label, a filename) cannot grow branches.

## Why MVG has been a powerful injection surface, historically

This next part is background, not this advisory, and it is a decade old. I include it because it explains why anyone who works on ImageMagick treats the MVG and MSL coders with suspicion, and why "it is only a drawing option" is not a reassuring sentence.

In 2016, a family of ImageMagick vulnerabilities that the press named **ImageTragick** (headlined by **CVE-2016-3714**, with siblings through CVE-2016-3718) showed what the format-plus-delegate machinery could do in the wrong hands. The core issue there was different in mechanism from this advisory: it was insufficient filtering of filenames passed into *delegate* commands, where a delegate is an external program ImageMagick shells out to, configured in `delegates.xml` and invoked through a `system()`-style call. Because a parameter expanded into that command line was not properly neutralized, a crafted input could inject shell metacharacters and reach command execution. The affected coders named at the time included EPHEMERAL, HTTPS, MVG, MSL, TEXT, SHOW, WIN, and PLT, and that family was fixed in ImageMagick 6.9.3-10 and 7.0.1-1.

What made it memorable was the toolkit of side effects that rode along with it, documented publicly at the time: server-side request forgery through the `https:` and `url:` pseudo-protocols, file deletion through `ephemeral:`, file writing or moving through `msl:`, and file reading through the `label:` and `text:` pseudo-protocols. An attacker could rename an exploit file with an innocent image extension, because ImageMagick identifies formats by content ("magic bytes") rather than trusting the extension, and any service that auto-thumbnailed an upload would process it. That is the lineage MVG sits in. To be clear about the distinction: ImageTragick was primarily delegate and pseudo-protocol command injection reaching the operating system, whereas GHSA-qq8x-fq3x-43h4 is injection *within the MVG drawing language itself* through the encoder, and it is rated far lower. The reason the history is relevant is not that this bug is as severe (it is not), but that MVG is a language with operators that touch external content, so breaking its grammar is never purely decorative.

## The mechanism, in general terms

Per the advisory, the defect is that drawing options are inadequately sanitized before the MVG encoder processes them, enabling code injection. Concretely, the shape of such a bug is: an option value that is expected to be a single token (a colour, a font name, a label, a filename) is written into the MVG stream without escaping the characters MVG treats as structure. A value carrying an unescaped quote or separator terminates the field it was supposed to fill, and the trailing text is then parsed by the drawing engine as additional MVG operators and operands rather than as an inert string.

Here is an *illustrative* sketch of that class. This is not the advisory's proof-of-concept, which was not published, but the canonical pattern of an unescaped-value injection, so the idea is concrete:

```
# Intended: a value is just data inside a quoted field.
fill 'blue'

# Class pattern: a value containing MVG syntax closes its own field early
# and the remainder is interpreted as further drawing commands rather than
# as an inert string.
fill 'blue' <additional MVG operators> 'rest'
#          ^ the value ended the quote here; everything after is now "code"
```

If you have seen SQL injection, this is the same move. There, a value like `' OR '1'='1` works because the single quote it carries closes the string literal the query was building, and the characters after it are read as SQL rather than as a name. Here the "query" is a line of MVG, the "string literal" is a quoted operand, and the characters after the stray quote are read as drawing operators. The surface differs, the bug is identical in spirit. The defender's error in both cases is the same: building a sentence in a structured language by pasting in a value, and trusting the value to stay inside the lines.

The fix for this class is always the same in spirit, and the advisory's CWE assignment points right at it: neutralize the input before it is emitted. Treat option values as opaque data, escape or reject the characters that carry MVG structure, and never let a value supply its own delimiters. ImageMagick classified the weakness as **CWE-79** (Improper Neutralization of Input During Web Page Generation, the "cross-site scripting" CWE). That label can mislead, so it is worth unpacking: CWE-79 is not fundamentally about browsers or HTML, it is about a program building output in a structured sink without neutralizing input that can change that output's structure. The project chose it here because the failure is an output-neutralization failure in a markup-and-command context, and MVG is exactly such a context. The lesson the CWE encodes, neutralize at the point of output, is precisely the lesson here.

## How it could be reached, and the realistic impact

The attack vector is control over the drawing options that reach the encoder. That is anything that lets an attacker influence the values ImageMagick draws with: a service that accepts caption text, colours, fonts, or geometry from a user and renders them, and there are more of those than people realize. Dynamic Open Graph and social-card image generation, watermarking pipelines, badge and certificate and ticket generators, avatar and nameplate renderers, and any "type your text, get a picture" feature all funnel user strings into drawing operations. If one of those strings reaches a draw primitive or a drawing option that the encoder serializes, the user is contributing to the MVG program. The attacker controls one or more option values; the consequence described by the advisory is that those values are interpreted as MVG commands instead of data.

I will be careful about impact because the advisory is, too. This is *code injection into the MVG drawing language* (the ability to get attacker text executed as drawing operations), not demonstrated operating-system command execution, and the published rating reflects a limited outcome. ImageMagick scored it **Low, CVSS 3.6** with the vector `CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:C/C:L/I:N/A:N`. Read it metric by metric: `AV:L` local attack vector, `AC:L` low complexity, `PR:N` no privileges required, `UI:R` user interaction required, `S:C` scope changed, and an impact of `C:L/I:N/A:N`, a limited confidentiality impact only, with no integrity or availability impact claimed. The one metric worth dwelling on is **Scope: Changed** (`S:C`). That is the advisory saying the injected MVG can affect resources beyond the encoder's own security scope, which is consistent with MVG operators that can pull in or composite other content. The scope change is the reason even a "low" rating here is a real finding and not a curiosity: the injected commands do not stay neatly inside the box the encoder was reasoning about. What it is *not* is a remote-code-execution claim, and the confidentiality-only, no-integrity, no-availability impact and the local vector are the advisory's own words on where the ceiling sits. I am not going to dress it up past that.

## Affected versions and the fix

Per the advisory the issue affects ImageMagick before **7.1.2-33** on the 7.x line and before **6.9.13-58** on the still-maintained 6.x line, and is fixed in those two releases respectively. The remediation follows the CWE-79 classification: properly neutralize drawing-option input before the MVG encoder emits it, so a value can no longer break out of its field and contribute commands to the drawing stream. The advisory was published on 10 October 2026, is credited to me as `bewithyash`, and carries no CVE identifier. The patch landing in both the 7.x and the maintained 6.x line matters in practice, because a large amount of production ImageMagick is still 6.x by way of distribution packages, and "we are on 6, so it does not apply" is not true here.

## Defensive guidance

Upgrading is the real fix: move to 7.1.2-33 or later, or 6.9.13-58 or later, so the encoder neutralizes drawing options itself. Everything below is defense in depth that you should want in place regardless of this one advisory, because it narrows the entire MVG and delegate attack surface that the 2016 history made famous.

The strongest lever ImageMagick gives you is `policy.xml`, its security policy file. It lets you revoke rights from specific coders by pattern. If your service never legitimately needs to read or render MVG or MSL (most do not), deny them outright. The relevant directives live inside the `<policymap>` stanza and look like this:

```
<policymap>
  <policy domain="coder" rights="none" pattern="MVG" />
  <policy domain="coder" rights="none" pattern="MSL" />
  <policy domain="coder" rights="none" pattern="LABEL" />
  <!-- the classic ImageTragick set, denied as a group -->
  <policy domain="coder" rights="none" pattern="EPHEMERAL" />
  <policy domain="coder" rights="none" pattern="URL" />
  <policy domain="coder" rights="none" pattern="HTTPS" />
  <policy domain="coder" rights="none" pattern="HTTP" />
  <!-- block indirect/filename reads via the @ prefix -->
  <policy domain="path" rights="none" pattern="@*" />
</policymap>
```

A `domain="coder"` policy with `rights="none"` removes ImageMagick's permission to use the named coder at all, so even a file that is really MVG inside, or an explicit `mvg:` request, is refused rather than interpreted. The `domain="path"` entry with pattern `@*` blocks the indirect-file syntax that lets a value smuggle in a filename to read. If you do need MVG or MSL for a legitimate workflow, confine it: run ImageMagick as an unprivileged user with no outbound network access and a read-only, minimal filesystem view, so that even a successful injection has nothing interesting to reach. That containment is what blunts the `S:C` scope change in practice.

At the application layer, the rule is the one this whole series keeps arriving at: treat user input as data, never as a fragment of a command language. Do not build `-draw` strings or drawing options by string-concatenating user text. Validate colours against a strict allowlist or a known color syntax, validate geometry as numbers, bound the length of caption and label text, and reject or strictly encode quotes, newlines, and other MVG structural characters before anything reaches the drawing options. Where you can, pass text through an interface that keeps it as a bound value rather than letting it become part of the program text. And identify formats deliberately: do not let a file's claimed extension decide how it is processed when content and explicit format prefixes are what ImageMagick actually trusts.

## The honest summary

I would rather under-claim a real bug than over-claim it with details I cannot source. The public advisory establishes the component (the MVG encoder), the class (code injection via inadequately sanitized drawing options), the weakness (CWE-79), the rating (Low, CVSS 3.6, Scope: Changed), the affected and fixed versions (before 7.1.2-33 and before 6.9.13-58, fixed in those two releases), the publication date (10 October 2026), and the credit. It does *not* publish a specific vulnerable function, file, line, or proof-of-concept, so I have described the mechanism at the level the advisory supports and flagged the illustrative example as exactly that. The ImageTragick material is dated background from 2016 and a different mechanism, included to show why MVG earns its suspicion, not to borrow its severity. The through-line connecting this to everything else in the series is the oldest rule in injection security: the moment you build a sentence in some language by pasting a value into it, that value has to be neutralized first, whether the language is SQL, a template, a shell, or ImageMagick's own MVG.

The one line to remember
MVG is a language, not a config file. Any option value concatenated into it unescaped can stop being a parameter and start being a command: the graphics-pipeline version of injection you already know.

### If your service renders user text into images, it speaks a drawing language.

Imperonlabs reviews the places where user input becomes instructions: template engines, query builders, and command languages like MVG hiding inside your image pipeline. We find the missing neutralization step before someone else does.

[Request an assessment](https://imperonlabs.com/#contact)
