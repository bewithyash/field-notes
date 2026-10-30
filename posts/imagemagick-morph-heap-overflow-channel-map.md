---
title: "Morphing a Grayscale Frame Into an RGBA One: GHSA-x48p-6655-8qgv"
date: October 2026
source: https://blog.imperonlabs.com/blog/imagemagick-morph-heap-overflow-channel-map.html
---

ImageMagick · Memory Safety · Fuzzing

# Morphing a Grayscale Frame Into an RGBA One

By Yash Pandya · October 2026 · GHSA-x48p-6655-8qgv · 22 min read

ImageMagick's `-morph` option builds the in-between frames of a crossfade: hand it two images and a tween count and it interpolates a smooth sequence from one to the other. To blend two pixels it walks their channels: red to red, green to green, alpha to alpha. The whole bug is a question of *whose* channel list it walks. The blend loop iterates the **source** frame's channel map but writes through those offsets into the **destination** frame's buffer, and when the two frames have different channel layouts, the source's offsets point past the end of the destination's pixel. For most pixels that silently corrupts the neighbour. For the last pixel of the last row it writes off the end of the heap allocation entirely, with an attacker-influenced value, past an attacker-sized block.

Disclosure note. GHSA-x48p-6655-8qgv is a fixed, published ImageMagick advisory credited to me (GitHub `bewithyash`), patched in 7.1.2-34. No CVE was assigned. I found it with a fuzzer aimed specifically at ImageMagick's multi-image list operations, a surface the standard single-image decoder harnesses never exercise, then reduced it to a four-line command line and confirmed the root cause against a patched build. The code locations, arithmetic, and sanitizer output below come from my advisory. The published severity is ImageMagick's, and where I give my own assessment I say so explicitly.

## At a glance

- **Advisory:** GHSA-x48p-6655-8qgv (ImageMagick GitHub Security Advisory, no CVE assigned).
- **Severity (published, ImageMagick):** Low. **CVSS 3.1 base 3.3**, vector `CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:N/A:L`.
- **Researcher self-assessment (mine, stated as such):** I argued High, **CVSS 3.1 7.1**, vector `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:H`, with an honest `C:N` because I did not demonstrate RCE; a service that morphs user-supplied sequences with no interaction reaches `UI:N` and about 8.1. The authoritative number remains ImageMagick's Low 3.3.
- **CWE:** CWE-787 (Out-of-bounds Write), secondary CWE-125 (the same loop also corrupts in-bounds neighbour pixels).
- **Affected function / file:** `MorphImages()` in `MagickCore/visual-effects.c`, specifically the channel-map blend loop at `visual-effects.c:1300-1319`. The ASan crash site is `SetPixelChannel` at `MagickCore/pixel-accessor.h:775`.
- **Affected versions:** every ImageMagick 7 release from commit `d2f2234b0` (2020-01-05, *"Separate -fx option from other visual effects methods"*) through `main` at the time of reporting. ImageMagick 6 is *not* affected. Both 32-bit and 64-bit, because this is index logic and not a pointer-width issue.
- **Patched version:** **7.1.2-34** (affected: `< 7.1.2-34`).
- **Reachability:** plain CLI `magick A B -morph N out`, and `MagickMorphImages()` through every binding. Default build, default policy, no size or resource-limit preconditions.

## Attack surface: where MorphImages runs, and why it sees attacker input

ImageMagick is the image-processing library behind an enormous amount of the internet's "upload a picture" machinery: thumbnailers, avatar resizers, document and media pipelines, asset processors. It is also a long, feature-rich C codebase that parses and transforms attacker-supplied images, often with attacker-influenced options, which is why it has been a perennial source of memory-safety findings. `MorphImages()` lives in `MagickCore/visual-effects.c` and is the implementation behind the `-morph` CLI option and the `MagickMorphImages()` API that every binding (MagickWand, PHP Imagick, Python Wand, RMagick) exposes. The ASan stack I captured runs `CLIListOperatorImages → MorphImages` straight from the plain `-morph` option, so there is no exotic code path involved.

What `-morph` actually does is produce fade transitions: given a list of frames, it inserts `N` interpolated tween frames between each adjacent pair, so a sequence of stills becomes a smooth crossfade. That is a staple of batch animation builders, GIF and video-thumbnail workflows, and any pipeline that stitches user images into a montage. Those are exactly the pipelines that accept more than one attacker-supplied image.

The one precondition that makes this different from a typical single-upload bug is that `-morph` operates on an image *list*: you need at least two frames, and the two adjacent frames have to have different channel layouts. That is a low bar. Two uploaded files of different types, or a single multi-frame container whose frames differ, is enough. A grayscale logo followed by a colour photo, a palette GIF frame next to an RGB frame, an RGB frame next to a CMYK-plus-alpha frame: all of these pair a narrow channel map with a wider one. Beyond the pairing there are no special requirements. Default build, default resource policy, any supported image formats, no size threshold, and the advisory confirms it on both 32-bit and 64-bit because the fault is in how an index is computed, not in how wide a pointer is. The grayscale-into-RGBA case I use throughout this post is simply the cleanest minimal pairing; it is representative, not special.

## The channel map, and how the write is indexed

In ImageMagick 7 a pixel is not a fixed RGBA struct; it is a run of channels described by a per-image *channel map*. The map records how many channels the pixel has and at what offset each one (red, green, blue, alpha, black, and so on) sits inside the pixel's run of quanta. A grayscale image has one channel; an RGBA image has four. The accessors `GetPixelChannels(image)`, `GetPixelChannelChannel(image, i)`, and `SetPixelChannel(image, channel, value, q)` all consult *that image's own* map to translate a logical channel into a concrete offset inside the pixel buffer. The map is the single source of truth for where a channel lives, so handing one function the wrong image's map silently relocates every write it performs.

Morph interpolates each adjacent pair of frames by resizing both to a common tween geometry. The *current* frame is resized and becomes the destination template, named `morph_images` in the code (the tween that gets appended to the output list). The *next* frame is resized to the same geometry and becomes the source, named `morph_image`. The blend then reads a pixel from the source and a pixel from the destination tween, mixes them, and stores the result back into the destination. Here is the loop, labelled `[1]` through `[4]` exactly as in my advisory, as it stood in the affected releases:

```
q=GetCacheViewAuthenticPixels(morph_view,0,y,morph_images->columns,1,exception);  /* destination buffer  */
...
for (x=0; x < (ssize_t) morph_images->columns; x++)
{
  for (i=0; i < (ssize_t) GetPixelChannels(morph_image); i++)          /* [1] SOURCE channel count  */
  {
    PixelChannel channel = GetPixelChannelChannel(morph_image,i);      /* [2] SOURCE channel map    */
    PixelTrait traits = GetPixelChannelTraits(morph_image,channel);
    PixelTrait morph_traits=GetPixelChannelTraits(morph_images,channel);
    if ((traits == UndefinedPixelTrait) || (morph_traits == UndefinedPixelTrait))
      continue;                                                        /* [3] only gate            */
    if ((morph_traits & CopyPixelTrait) != 0)
      {
        SetPixelChannel(morph_image,channel,p[i],q);                   /* [4] WRITE: q[map_×offset] */
        continue;
      }
    SetPixelChannel(morph_image,channel,ClampToQuantum(alpha*(double)
      GetPixelChannel(morph_images,channel,q)+beta*(double) p[i]),q);  /* [4] WRITE, blended value  */
  }
  p+=(ptrdiff_t) GetPixelChannels(morph_image);                        /* p: source stride          */
  q+=(ptrdiff_t) GetPixelChannels(morph_images);                       /* q: destination stride      */
}
```

Walk it line by line.

- **`[1]` loop bound:** `GetPixelChannels(morph_image)`. The iteration count is the number of channels in the *source* frame. For an RGBA source that is four.
- **`[2]` channel resolution:** `GetPixelChannelChannel(morph_image, i)`. The logical channel for iteration `i` is read from the *source's* map, and so is its offset when it is later used to index.
- **`[3]` the only gate:** the two `GetPixelChannelTraits` calls fetch the trait of this channel in the source and in the destination, and the loop skips the channel only if it is `UndefinedPixelTrait` in either frame. That is a pure presence check. It asks "does this channel exist in both?" and never asks "does its offset fit inside the destination's stride?"
- **`[4]` the write:** both the copy-trait fast path and the blended path call `SetPixelChannel(morph_image, channel, value, q)`. `SetPixelChannel(image, channel, quantum, pixels)` stores at `pixels[image->channel_map[channel].offset]`. The `image` argument is `morph_image`, the source, so the offset comes from the *source's* map. But the buffer being written, `q`, is the *destination* tween's buffer.
- **The strides at the bottom:** `p` advances by `GetPixelChannels(morph_image)` (source stride) and `q` advances by `GetPixelChannels(morph_images)` (destination stride). So `q` walks pixels spaced by the destination's channel count, while the offsets written into each of those pixels come from the source's map.

Read `[1]`, `[2]`, and `[4]` together and the defect is stark. The loop bound and the channel offsets both come from the source; the buffer and its per-pixel stride both come from the destination. When the source has more channels, or a wider map, than the destination tween, an offset such as 1 or 2 is perfectly valid for the source but exceeds the destination's stride of 1. The write at `[4]` then lands outside the destination pixel it was supposed to fill.

## Root cause: the one value the attacker shapes, and where the write goes

The attacker-shaped quantity is the source frame's channel map, specifically the offsets it assigns to channels that also pass the presence gate in the destination. It is shaped simply by choosing the source image's type: an RGBA source yields red/green/blue at offsets 0, 1, 2 (and alpha), a CMYK source yields four or five colour offsets, and so on. Pair any such source with a narrower destination tween and you have offsets that exceed the destination's stride.

Take the minimal case the advisory uses: a grayscale destination tween (one channel per pixel, stride 1) and an RGBA source (red, green, blue at offsets 0, 1, 2). Grayscale images in ImageMagick 7 still *define* red/green/blue traits, because those three alias the single gray channel, so the trait gate at `[3]` passes for all three. The loop therefore performs writes at `q[0]`, `q[1]`, and `q[2]` for a destination pixel whose real width is one channel:

- **Every non-final pixel:** `q[1]` and `q[2]` land in the *next* pixel's slot. In-bounds, but wrong. Silent corruption of the image, row after row. This is the secondary CWE-125 behaviour: the loop reads and clobbers neighbour slots it had no business touching.
- **The last pixel of the last row:** there is no next pixel. `q[1]` writes one quantum past the end of the destination's pixel-cache allocation and `q[2]` writes two past. That is the heap out-of-bounds write, CWE-787.

Expressed generally: the destination tween is `W_t × H_t` pixels at `S_d` channels each; the source carries channels at map offsets `O(c)`. A write fires for every channel that is defined in both maps, at `q[O(c)]` where `q` strides by `S_d`. Any channel with `O(c) >= S_d` overwrites a following slot, and at the final pixel it overflows the allocation by up to `(max O(c)) - S_d + 1` quanta. For the grayscale-plus-RGB pairing that is two quanta past the end (8 bytes on a Q16-HDRI build, 4 bytes on Q16 non-HDRI). Channels absent from the destination, such as alpha on a tween that has none, are caught by the trait gate and skipped, which is what caps the overflow extent rather than letting it run arbitrarily far.

Two further facts make this more than a crash. First, the written value is not noise. In the blended path it is `ClampToQuantum(alpha*(destination channel) + beta*(source pixel))` with `beta = (n+1)/(N+1)` for tween index `n` of `N`. As the tween count `N` grows, `beta` approaches 1, so the stored value is dominated by the attacker's own source-pixel channel; for `N = 15` it is already at least 93% the source value. The destination term is itself attacker-shaped, because the attacker supplies the destination frame's content. The bytes written off the end are therefore attacker-influenced, not random. Second, the victim allocation's size is attacker-chosen, because the pixel cache is allocated as `W_t × H_t × S_d × sizeof(Quantum)` and `W_t` and `H_t` interpolate between the two attacker-chosen frame geometries. A small, precisely placed, attacker-valued write past an attacker-sized, 64-byte-aligned heap block (from `OpenPixelCache → AcquireAlignedMemory`) is the classic adjacent-object corruption primitive.

## Full reproduction

I reproduced this on a 64-bit Q16-HDRI AddressSanitizer plus UndefinedBehaviorSanitizer CLI build of commit `e87c97d2c` (7.1.2-33 Beta), and built a second, independently patched binary carrying the fix below for the differential control. The build itself is a stock ImageMagick checkout configured with the sanitizers; nothing about the configuration is load-bearing, since the bug is index logic rather than width or policy.

The PoC is four lines. The first two synthesize the two frames; the third runs the morph. Both inputs are ordinary image files (PAM here; any format pair with differing channel maps works):

```
magick -size 8x8 xc:gray g.pam
magick -size 8x8 xc:gray -colorspace sRGB -type TrueColorAlpha b.pam
MAGICK_CONFIGURE_PATH=/nonexistent \
ASAN_OPTIONS=detect_leaks=0:allocator_may_return_null=1 \
magick g.pam b.pam -morph 3 null:
```

The sizing is deliberate. `g.pam` is an 8 × 8 grayscale frame, so the destination tween resized from it is 1 channel per pixel and its pixel cache is 8 × 8 × 1 channel × 4 bytes, which is exactly 256 bytes. `b.pam` is an 8 × 8 TrueColorAlpha frame, so the source carries red/green/blue at offsets 0/1/2 (plus alpha). As noted above, the grayscale tween still defines R/G/B traits, so the gate passes and the source's offset-1 and offset-2 writes are issued against a stride-1 buffer. `-morph 3` asks for three tween frames between the pair, and `null:` discards output so nothing but the crash is observed. Here is the verbatim sanitizer output, identical across all three runs:

```
==4127566==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x512000001040 at pc 0x55cb39fad224 ...
WRITE of size 4 at 0x512000001040 thread T0
    #0 0x55cb39fad223 in SetPixelChannel /root/im-hunt/ImageMagick/./MagickCore/pixel-accessor.h:775:46
    #1 0x55cb39fb3385 in MorphImages /root/im-hunt/ImageMagick/MagickCore/visual-effects.c
    #2 0x55cb3a1c8ea7 in CLIListOperatorImages /root/im-hunt/ImageMagick/MagickWand/operation.c:4559:23
    #3 ... CLIOption ... ProcessCommandOptions ... MagickImageCommand ... main        (plain `-morph` CLI)
0x512000001040 is located 0 bytes after 256-byte region [0x512000000f40,0x512000001040)
allocated by thread T0 here:
    #3 ... OpenPixelCache /root/im-hunt/ImageMagick/MagickCore/cache.c:3897:46
SUMMARY: AddressSanitizer: heap-buffer-overflow ... in SetPixelChannel
```

How to read it. The header line says the fault is a `heap-buffer-overflow` and the next line says it is a `WRITE of size 4`, four bytes being one quantum on this build: this is an out-of-bounds write, not a read. Frame `#0` is `SetPixelChannel` at `pixel-accessor.h:775`, the accessor that applied the source's offset; frame `#1` is `MorphImages`, the loop above; frame `#2` is `CLIListOperatorImages`, the command-line list operator reached straight from `-morph`. The line `0 bytes after 256-byte region [0x..f40,0x..1040)` is the crucial one: the write address is exactly at the end of a 256-byte allocation, so the first overflowing byte is the first byte past the destination tween's pixel cache (the 8 × 8 × 1 channel × 4 byte block). The allocation's own stack, truncated here, points at `OpenPixelCache` in `cache.c`, confirming the overflowed object is the pixel cache rather than anything incidental. The first out-of-bounds write is the source's green channel (map offset 1) applied to the last pixel of the last row, exactly the stride mismatch from the previous section; the blue channel (offset 2) would be the eighth byte past.

I did not stop at one trace. The same crash reproduced identically from twelve independent fuzzer corpus units, all with the same `SetPixelChannel ← MorphImages` stack, and a brute matrix of 27 differing frame-type combinations (grayscale + RGBA, palette + RGB, RGB + CMYK-alpha, and others) all crashed from the plain CLI. Same-type pairs (gray + gray, TrueColorAlpha + TrueColorAlpha) ran clean as negative controls. Separately, on a non-sanitized build with an `LD_PRELOAD` malloc tracer, I confirmed the OOB target is deterministic: for every tween the pixel-cache block is immediately followed in the glibc heap by ImageMagick's roughly 13 KB `CacheInfo` descriptor, observed at `victim_end + 0x110` in every instance, and the eight overflow bytes land on that adjacent chunk's glibc header (bytes 0 to 3 on `prev_size`, bytes 4 to 7 on the chunk `size` field, which takes the attacker-influenced value).

## Impact, honestly stated

I want to be precise about what this is and is not. It is a deterministic heap out-of-bounds *write* with attacker-influenced values, past an attacker-sized allocation, plus guaranteed in-bounds image corruption on every row. That is a real memory-corruption primitive, and the fact that the overflow reliably lands on an adjacent chunk's size field makes it a textbook setup for free-time integrity aborts (a deterministic denial of service in the one-shot CLI case) or, in a long-lived service doing repeated morphs with heap grooming, for the overlapping-chunk games that are the classical route toward a worse write.

It is *not* a demonstrated remote code execution. The overwrite is small (up to two quanta, so 8 bytes on a Q16-HDRI build and 4 on Q16 non-HDRI), and turning it into control of execution depends on the deployment's heap layout and a grooming scenario I did not build end to end. So I graded confidentiality impact as `C:N` in my own assessment and did not claim more.

ImageMagick's published advisory rates this **Low, CVSS 3.1 base 3.3** (`CVSS:3.1/AV:L/AC:L/PR:N/UI:R/S:U/C:N/I:N/A:L`), reflecting a local attacker and an availability-only outcome for the one-shot case. My own write-up argued the risk is higher in a hosted, no-interaction pipeline that morphs user images on demand: network attack vector, no user interaction (`UI:N`), and a memory corruption rather than a mere crash, which I scored as High 7.1 and up to about 8.1 for the service scenario. I present that only as my researcher assessment. The authoritative severity is the Low 3.3 that ImageMagick assigned, and I will not inflate it. Either way the fix is worth taking, because the in-bounds corruption alone silently damages output on every affected morph.

## Methodology: how I targeted the surface and reduced the crash

This finding came out of a decision about *what* to fuzz, not a better mutator. The public ImageMagick fuzzing harnesses, including the OSS-Fuzz ones, overwhelmingly drive a single image through the decode-and-transform path. They are very good at that, and that surface is correspondingly well scrubbed. What almost nobody fuzzes is the *multi-image list operation* surface: the operators that take two or more images and combine them, where a second image's metadata (here, its channel map) drives how the first image's buffer is written. So I built a harness aimed specifically at two-image and list operations, with `-morph` among the targets, feeding pairs of frames of deliberately mismatched types.

That harness produced twelve corpus units that crashed at the same stack. From there the work was reduction and root-causing, not more fuzzing. I minimized the crashing inputs down to the smallest frames that still triggered it, which gave the 8 × 8 grayscale and TrueColorAlpha pair, then rewrote the reproduction as the four-line CLI PoC so it needs no harness at all. I built the 27-combination frame-type matrix to show the crash is a property of mismatched channel maps in general and not an artefact of one format pairing, and ran same-type pairs as negative controls to show it is specifically the mismatch that does it. To prove root cause rather than a coincidental symptom, I wrote the fix, built a second sanitizer binary with it applied, and confirmed the identical PoC is clean on the patched build while still crashing on the stock one. I also checked novelty directly: `git log -S` on the loop's distinctive accessor returns only the 2020 commit that introduced the pattern, the four existing GHSAs in `visual-effects.c` are all in `WaveletDenoiseImage` and `ColorizeImage` and touch `MorphImages` nowhere, and the bug was still live on `origin/main` when I reported it.

## Remediation: the fix and upgrade guidance

The structural fix is to make the write domain the *destination*. The loop should iterate and resolve channels through the destination's map (`morph_images`) and read the corresponding source value, so that every write offset is consistent with the destination's stride by construction. Reading from the source is fine; indexing the write by the source's layout was the bug. Concretely, the change I proposed and verified swaps the four source references in the loop for destination ones:

```
-  for (i=0; i < (ssize_t) GetPixelChannels(morph_image); i++)
+  for (i=0; i < (ssize_t) GetPixelChannels(morph_images); i++)
   {
-    PixelChannel channel = GetPixelChannelChannel(morph_image,i);
+    PixelChannel channel = GetPixelChannelChannel(morph_images,i);
     PixelTrait traits = GetPixelChannelTraits(morph_image,channel);
     PixelTrait morph_traits=GetPixelChannelTraits(morph_images,channel);
     ...
-    SetPixelChannel(morph_image,channel,p[i],q);
+    SetPixelChannel(morph_images,channel,p[i],q);
     ...
-    SetPixelChannel(morph_image,channel,ClampToQuantum(alpha*(double)
+    SetPixelChannel(morph_images,channel,ClampToQuantum(alpha*(double)
       GetPixelChannel(morph_images,channel,q)+beta*(double) p[i]),q);
   }
```

After this change the loop count, the channel offsets, and the buffer stride all come from the destination, so an offset can never exceed the destination's own stride and the trait gate's presence check is sufficient. On the patched sanitizer build the PoC is clean across all three runs while the stock build still crashes, which is what establishes that this mismatch, and nothing else, is the cause.

For operators, the remediation is to upgrade to **ImageMagick 7.1.2-34** or later; everything earlier in the 7 line is affected. ImageMagick 6 is not affected, since its morph uses fixed `PixelPacket` field accessors with no channel-map indirection and the class cannot occur there, but it is end-of-life and 7 remains the right target. If you cannot upgrade immediately and you run a pipeline that morphs user-supplied sequences, the interim mitigation is to avoid feeding adjacent frames of differing channel type into `-morph`: normalize every frame to a common type (for instance force all frames to RGBA) before the morph, which removes the map mismatch that drives the write.

## Honest caveats and disclosure timeline

- **RCE not demonstrated.** The overwrite is small, up to 8 bytes on Q16-HDRI and 4 on Q16, and its exploitability depends on heap context I did not weaponize end to end. I graded it `C:N` accordingly, and the published severity is Low.
- **A list operation is required.** Single-image pipelines are unaffected. You need at least two frames, and the adjacent frames have to have different channel maps.
- **The trait gate caps, but does not fix.** Channels absent from the destination are trait-gated and skipped, which bounds the overflow extent. The gate's flaw is that it reconciles channel *presence* and never channel *offsets*, which is precisely the bug.
- **Severity framing.** The Low 3.3 is ImageMagick's and is the number to cite. The High 7.1 / 8.1 figures are my own researcher assessment of a hosted no-interaction scenario and are labelled as such throughout.

Timeline:

- **2026-10-05 to 2026-10-11:** found by the two-image / list-operation fuzzer (twelve corpus units, identical stacks); minimized to the four-line CLI PoC; captured the 27-combination frame-type matrix; root-caused to the destination/source channel-map mismatch.
- **2026-10-12:** patched-build differential control verified 3/3 clean; novelty verified against all `visual-effects.c` advisories and against `origin/main`; advisory finalized and reported.
- **Subsequently:** published as GHSA-x48p-6655-8qgv, rated Low (CVSS 3.3), credited to me, and fixed in 7.1.2-34. No CVE was assigned.

The one line to remember
When two objects each carry their own layout, a loop must index the write by the layout of the thing it is writing into. Morph read the source's channel map and wrote into the destination's buffer, and the gap between the two walked off the heap.

### Your image pipeline runs more C than you think.

Imperonlabs fuzzes the operations nobody else targets (multi-image list operations, exotic option combinations, format converters) and reduces every crash to a minimal, root-caused, fixable finding. If users can upload images, this is your attack surface.

[Request an assessment](https://imperonlabs.com/#contact)
