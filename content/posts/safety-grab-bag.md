+++
title = "Four Footguns the Base System Already Defused"
date = 2026-06-04
draft = false
summary = "A temp file without the race, a string copy without the overflow, a password read without the echo leak, and a way to ask 'am I running with privileges someone might be exploiting?' Each replaces a standard-library call that has a subtle security bug, and each is in base."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

There's a category of C standard-library function that works fine in the demo and carries a security bug into production. Not a bug in your use of it, a bug in the function's design, the kind that passes code review because the code looks correct and does the right thing on the happy path. The base system ships safer replacements for a handful of these, and using them is usually a smaller diff than the comment explaining why the unsafe version was okay. Here are four, each paired with the footgun it defuses.

## 1. Temp files without the race: `mkstemp`, not `mktemp` or `tmpnam`

The classic temp-file bug is a TOCTOU (time-of-check-to-time-of-use) race. `mktemp(3)` (the function, not the command) and `tmpnam(3)` *generate a unique filename* and hand it back, and then you open it. In the window between "generate the name" and "open the file," an attacker who can guess or observe the name creates it first, perhaps as a symlink to something sensitive, and your subsequent open follows it. In a privileged program that's a file-clobbering or privilege-escalation bug.

The fix is to never separate naming from creation. `mkstemp(3)` generates the name *and* atomically creates and opens the file in one operation, returning a descriptor:

```c
#include <stdlib.h>
#include <unistd.h>

char tmpl[] = "/tmp/myapp.XXXXXX";   /* the X's get replaced */
int fd = mkstemp(tmpl);              /* creates + opens atomically, O_EXCL */
if (fd == -1)
    err(1, "mkstemp");
/* tmpl now holds the actual filename; fd is open and yours alone. */
```

There's no window: the file is created with `O_EXCL` semantics as part of getting the name, so it cannot pre-exist or be substituted. The family extends to `mkostemp` (pass extra `open` flags like `O_CLOEXEC`), `mkdtemp` (a directory instead of a file), and `mkostemps`/`mkostempsat` (suffixes, and relative-to-a-descriptor for Capsicum-friendly code).

The base system will even warn you off the unsafe one. libc marks `mktemp` with a linker note, so using it produces, at link time:

```
warning: mktemp() possibly used unsafely; consider using mkstemp()
```

The system is nagging you toward the safe call from inside the toolchain. Listen to it.

## 2. String copies without the overflow: `strlcpy` / `strlcat`

`strcpy` overflows if the source is longer than the destination; everyone knows that. The trap is the "safe" fix, `strncpy`, which has two of its own problems. It does *not* NUL-terminate if the source is as long as or longer than the buffer, so you can hand the rest of your program a string with no terminator, a bug that surfaces as a later overread. And it pads the entire remaining buffer with NUL bytes, which is just wasted work on a large buffer. `strncpy` was designed for an old fixed-width-field format, not for safe string copying, and pressing it into that role produces subtle bugs.

`strlcpy` and `strlcat` are the bounded copies designed for the job:

```c
#include <string.h>

char buf[64];
if (strlcpy(buf, src, sizeof(buf)) >= sizeof(buf)) {
    /* truncation happened; handle it instead of silently continuing */
}

strlcat(buf, more, sizeof(buf));   /* append, always NUL-terminated */
```

Two design choices make them safe. Both take the **total size of the destination buffer** (so `sizeof(buf)`, not "size minus one" arithmetic you can botch), and both **always NUL-terminate** as long as the buffer isn't zero-length. You always get a valid C string out. And the **return value is the length the function tried to produce**, so `>= dstsize` is a clean truncation test, exactly the check `strncpy` makes awkward. Truncation becomes something you detect and handle, not something that silently corrupts.

The FreeBSD element here is worth stating precisely, because it's a nice one: these came out of OpenBSD (Todd Miller, 1998) and the whole BSD family has shipped them in base ever since, declared right in `string.h`. glibc pointedly *refused* to add them for twenty-five years, on the argument that they encourage silent truncation, so on Linux you needed libbsd until glibc finally relented in 2023. FreeBSD not only ships them, it has largely *converted its own base* to them: there are more uses of `strlcpy` in the base source than of `strcpy` and `strncpy`. The safe version isn't just available, it's the house style.

## 3. Reading a password without leaking it: `readpassphrase`

Reading a passphrase looks trivial and is full of small hazards. If you read from stdin, a pipe can feed it non-interactively in ways you didn't intend, and you can't control terminal echo. If you disable echo with `termios` by hand and the user hits Ctrl-C mid-entry, your signal handling had better restore the terminal, or you leave their shell with echo off and no visible typing, the hallmark of a program that rolled its own password prompt and got the cleanup wrong.

`readpassphrase(3)` does the whole thing correctly:

```c
#include <readpassphrase.h>

char pw[256];
if (readpassphrase("Password: ", pw, sizeof(pw), RPP_ECHO_OFF) == NULL)
    err(1, "readpassphrase");
/* ... use pw ... */
explicit_bzero(pw, sizeof(pw));   /* wipe it when done */
```

By default it reads directly from `/dev/tty`, not stdin, so it talks to the actual terminal regardless of redirection. It turns off echo (`RPP_ECHO_OFF`), and, the part hand-rolled versions get wrong, if an error or signal interrupts entry, it *restores the terminal state* before returning. The flags cover the real variations: `RPP_REQUIRE_TTY` to fail rather than proceed without a terminal, `RPP_STDIN` to deliberately read from stdin when you do want that, `RPP_ECHO_ON` for confirmation prompts. (Pair it with `explicit_bzero` to wipe the buffer after, a companion base function that, unlike `memset`, the compiler is not permitted to optimize away.)

Like the string functions, this is an OpenBSD-origin idiom that's been first-class in FreeBSD base for years. It's what you want any time a program asks for a secret at the terminal.

## 4. "Am I running with borrowed privileges?": `issetugid`

This one guards against a bug you might not know exists. A setuid or setgid program inherits its *environment* from whoever launched it, and that person may be an attacker. So any setuid program that trusts an environment variable, to find a config file (`$HOME`), a library path (`$LD_LIBRARY_PATH`), a resolver option (`$RES_OPTIONS`), is potentially handing an unprivileged attacker a lever on its elevated privileges. The classic attack is setting a hostile library path and exec'ing a setuid binary.

`issetugid(2)` is the check that closes this off:

```c
#include <unistd.h>

if (issetugid()) {
    /* We got extra privileges from the exec, or changed uid/gid.
       Do not trust attacker-controllable inputs like the environment;
       use safe built-in defaults instead. */
} else {
    /* Normal invocation; honoring $CONFIG_PATH etc. is fine. */
}
```

It returns 1 if the process is "tainted", created by exec'ing a setuid/setgid binary that actually gained privileges, or having changed any real/effective/saved uid or gid, and 0 otherwise. It cannot fail. The idiom is to consult it before trusting any input an attacker could have set, and fall back to safe defaults when tainted.

The reason to trust this is that libc itself relies on it. The functions that read security-relevant environment variables, `getenv` for the sensitive ones, the resolver's `res_init`, the name-service dispatcher, call `issetugid` to decide whether to honor the environment, which is how FreeBSD keeps a hostile `$LD_*` or `$RES_OPTIONS` from subverting every setuid program on the system. If you write anything that might be installed setuid, or a library that might be linked into one, `issetugid` is how you make the same decision correctly. (Linux's nearest equivalent is checking `getauxval(AT_SECURE)`; the BSD call is a cleaner, purpose-built interface, and it tracks address-space taint too, not just the secure-exec flag.)

## The pattern

None of these four is exotic. Each replaces a function you already know with one that removes a specific, well-understood security bug: the temp-file race, the unterminated or overflowing copy, the echo-leak-and-terminal-corruption, the trusted-environment escalation. The base system shipped the safe version, in some cases decades ago, and in the case of `mktemp` it will literally warn you at link time to switch. The reason these are worth a post is that the unsafe versions *look* fine, pass review, and work in testing, which is exactly the profile of a bug that ships. Reaching for the base system's safer primitive is the cheap habit that keeps that class of bug out of your program in the first place.

---

*Man pages: `mkstemp(3)` (and `mkostemp`/`mkdtemp`/`mkostemps` siblings), `strlcpy(3)` and `strlcat(3)`, `readpassphrase(3)`, `issetugid(2)`, plus `explicit_bzero(3)` for wiping secrets the compiler can't optimize away. `mktemp(3)` and `tmpnam(3)` document their own hazards in their BUGS/SECURITY sections, worth reading once so you recognize them in code.*

*A note on provenance and portability: `strlcpy`/`strlcat` and `readpassphrase` originated in OpenBSD and are shipped across the BSDs in base; on Linux they may require libbsd (though glibc gained the string functions in 2.38). `issetugid` and the `mkstemp` family are widely available but with per-platform quirks worth checking if you're writing portable code. Signatures and the `mktemp` link-warning text here are from FreeBSD 15.1 headers and libc sources.*
