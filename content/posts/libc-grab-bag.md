+++
title = "Five Things You're Reimplementing That libc Already Ships"
date = 2026-06-14
draft = false
summary = "A bias-free random number, a bitset, a one-call file hash, the program's own name, and human-readable byte sizes. Each is a small thing C programmers rewrite constantly, and each is already in the FreeBSD base system."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

The big base-system features get the attention: ZFS, DTrace, jails, Capsicum. This post is about the opposite end of the scale. Each of these is a small, unglamorous thing you have almost certainly hand-written, copy-pasted, or pulled a dependency in for, and each one has been sitting in the base system the whole time, documented, at a known version. None is big enough for its own post. Together they make a point: a surprising amount of the "I'll just write a quick helper" reflex is unnecessary on FreeBSD, because the quick helper already shipped.

Five of them.

## 1. A random number without modulo bias: `arc4random_uniform`

You need a random number in `[0, n)`. The reflex is `rand() % n` or `arc4random() % n`. Both are biased: unless `n` divides the generator's range evenly, the low values come up slightly more often, because the leftover values at the top of the range wrap around to them. For a dice roll nobody cares; for anything security-adjacent (picking a random token, shuffling, sampling) it's a real flaw.

The base system has the correct version:

```c
#include <stdlib.h>

uint32_t n = arc4random_uniform(6);   /* 0..5, uniform, no bias */
```

`arc4random_uniform(upper_bound)` returns a value uniformly distributed and strictly less than `upper_bound`, with the modulo bias handled for you. It's part of the `arc4random` family in libc (`arc4random()` for a full 32-bit value, `arc4random_buf(buf, len)` to fill a buffer), all seeded from the kernel's CSPRNG, so you also don't manage seeding. No `srand()`, no `/dev/urandom` plumbing, no bias.

The contrast: on Linux you reach for `arc4random` from libbsd (a dependency) or roll the rejection-sampling loop yourself. Here the right primitive is the default one, in libc.

## 2. A bitset you don't have to hand-roll: `bitstring(3)`

You need a few thousand flags. The reflex is an array of `uint64_t` and some `>> & |` bit-twiddling you'll get subtly wrong at least once. The base system ships a bitset as a set of macros:

```c
#include <sys/bitstring.h>

bitstr_t *bits = bit_alloc(1000);   /* 1000 bits, cleared */

bit_set(bits, 42);                  /* set bit 42 */
bit_clear(bits, 42);                /* clear it */
if (bit_test(bits, 42)) { /* ... */ }

int first_set;
bit_ffs(bits, 1000, &first_set);    /* find first set bit -> index or -1 */
int first_clear;
bit_ffc(bits, 1000, &first_clear);  /* find first clear bit */

int count;
bit_count(bits, 0, 1000, &count);   /* population count over a range */
free(bits);
```

The win isn't `bit_set`, which you could write. It's `bit_ffs` ("find first set") and `bit_ffc` ("find first clear") and `bit_count`, the operations that are annoying and bug-prone to write by hand against a packed array, handed to you correct. It's macro-only, in a base header, so there's nothing to link. The same `sys/bitstring.h` is used inside the kernel, so it's not a toy.

The contrast: this is roughly what C++ gives you as `std::bitset`, except it's available to plain C, in base, with the find-first and counting operations built in.

## 3. Hash a file in one call: the digest `File`/`Fd` helpers

You want the SHA-256 of a file. The reflex is the four-step dance: `SHA256_Init`, then a `read()` loop feeding `SHA256_Update`, then `SHA256_Final`, plus the buffer management and error handling around the loop. The base system's digest libraries (`libmd`) ship convenience variants that do the whole thing:

```c
#include <sha256.h>
/* link with -lmd */

char out[SHA256_DIGEST_STRING_LENGTH];

/* By path: */
SHA256_File("/etc/rc.conf", out);          /* out now holds the hex digest */

/* By open descriptor: */
SHA256_Fd(fd, out);

/* From an in-memory buffer: */
SHA256_Data(data, len, out);
```

`SHA256_File(path, buf)` opens, reads, hashes, and formats the result as a hex string, in one call. There are `Fd` variants for an already-open descriptor (which compose nicely with sandboxed code that was handed a descriptor rather than a path), `Data` for a memory buffer, and `FileChunk`/`FdChunk` to hash a byte range. Every digest in base has the same family: `MD5File`, `SHA1_File`, `SHA512_File`, `RIPEMD160_File`, `SKEIN256_File`, and so on, with matching `Fd`/`Data` siblings.

The contrast: with OpenSSL you write the init/update/final loop yourself, or wrap it. The base digests give you the loop pre-written for the overwhelmingly common "just hash this file" case.

## 4. The program's own name: `getprogname`

You want your error messages to start with the program's name, the Unix convention (`myprog: cannot open file`). The reflex is to stash `argv[0]` in a global and `basename()` it. The base system already tracked it for you:

```c
#include <stdlib.h>

fprintf(stderr, "%s: something went wrong\n", getprogname());
```

`getprogname()` returns the program's name with no leading path, no `argv[0]` bookkeeping, no `basename` call, available from anywhere without passing `argv` around. (There's a `setprogname()` too, but in practice the runtime has already set it for you by the time `main` runs.) This is also exactly what the `err(3)`/`warn(3)` family uses internally to prefix its messages, which is itself another base nicety worth knowing: `err(1, "open %s", path)` prints the program name, your message, and the `strerror(errno)` text, then exits.

The contrast: glibc has the `program_invocation_short_name` global as a non-portable extension; `getprogname`/`setprogname` are the BSD idiom and read more clearly.

## 5. Human-readable sizes, both directions: `humanize_number` / `expand_number`

You want to print `1048576` as `1.0M`, the way `ls -h`, `df -h`, and `du -h` do. The reflex is a little ladder of `if (n > 1024*1024*1024) ...` with a fixed set of suffixes. The base system has the function those tools actually use:

```c
#include <libutil.h>
/* link with -lutil */

char buf[8];
humanize_number(buf, sizeof(buf), 1048576, "B",
    HN_AUTOSCALE, HN_DECIMAL);
/* buf -> "1.0MB" */
```

`humanize_number(buf, len, number, suffix, scale, flags)` formats a number with an automatically chosen scale and a unit suffix. `HN_AUTOSCALE` picks the magnitude, `HN_DECIMAL` keeps one decimal place, `HN_IEC_PREFIXES` switches to Ki/Mi/Gi, `HN_DIVISOR_1000` uses powers of 1000 instead of 1024. It's the same routine the base utilities use, so your output matches theirs.

And it goes the other way, which is the part people miss. `expand_number` parses `"1.0M"` back into an integer:

```c
int64_t n;
expand_number("2G", &n);    /* n -> 2147483648 */
```

`expand_number(buf, &num)` reads a human-written size like `2G` or `512k` and gives you the integer. This is exactly what you want for parsing a size out of a config file or a command-line flag, the reverse of the pretty-printing, and it's the same parser the base tools use to read size arguments. Two functions, both directions, formatting and parsing agreeing with each other by construction.

The contrast: on Linux you hand-roll both halves, and your formatter and parser disagree on an edge case eventually. Here they're a matched pair in `libutil`.

## The pattern

None of these is clever. That's the point. Each is the kind of thing you write in five minutes, get 90% right, and carry a latent bug in (the modulo bias, the off-by-one in the bit loop, the formatter that disagrees with the parser). The base system wrote them once, carefully, and ships them to every install at a known version with a man page. The reflex to "just write a quick helper" is worth pausing on, because on FreeBSD the quick helper is frequently already there, already correct, and already documented.

When you find yourself reaching for a small utility function, it is worth thirty seconds with `apropos` first. The base system is much wider than its headline features, and most of that width is exactly this: small, correct, unglamorous things you'd otherwise rewrite.

---

*Man pages: `arc4random(3)`, `bitstring(3)`, `sha256(3)` (and its `md5(3)`, `sha(3)`, `sha512(3)` siblings, all from `libmd`), `getprogname(3)` and `setprogname(3)`, `humanize_number(3)` and `expand_number(3)` (from `libutil`). Also worth a look while you're in there: `err(3)` and `warn(3)` for the program-name-prefixed error idiom that `getprogname` underpins.*

*A note on linking: `arc4random`, `getprogname`, and friends are in libc (no extra flag); the digest helpers need `-lmd`; `humanize_number`/`expand_number` need `-lutil`; `bitstring` is header-only. Signatures here are from FreeBSD 15.1 headers and man pages; confirm the exact flag spelling for `humanize_number` against `humanize_number(3)` on your box, since the `HN_` flags compose and the page documents the full set.*
