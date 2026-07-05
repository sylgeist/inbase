+++
title = "Capsicum: Take Away a Process's Ambient Authority, Then Hand Back Exactly What It Needs"
date = 2026-06-08
draft = false
summary = "Most sandboxes filter syscalls: a blocklist you have to get exhaustively right. Capsicum inverts it. cap_enter() drops a process into a world with no ambient authority at all, where the only things it can touch are the descriptors it already holds. This is the capability model, in base, and it's the design the libnv, process-descriptor, and kqueue pieces have all been pointing at."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

A normal Unix process carries an enormous amount of authority it never uses. A program that just compresses the one file you named on the command line can, at any moment, open any other file its user can read, connect to the network, spawn processes, read `/etc/passwd`, and rummage through `/proc`-equivalents. It won't, if it's well behaved. But if an attacker finds a bug in its input parser, all of that authority is right there to be misused. This is *ambient authority*: power that comes from who you are (your uid, your view of the global filesystem namespace) rather than from what you were specifically handed.

Capsicum's idea is to take the ambient authority away entirely, and then give back only the specific things the program actually needs, as capabilities. It's in the FreeBSD base system, it's the security model several earlier pieces in this series have been circling, and once it clicks it changes how you think about what a program is *allowed* to be.

## Two mechanisms: capability mode, and capability rights

Capsicum is two things that work together. Keep them separate in your head.

**Capability mode** is the big switch. One call:

```c
#include <sys/capsicum.h>

cap_enter();
```

After `cap_enter()`, the process is in a mode of execution where, in the words of `capsicum(4)`, access to system calls requiring global namespace access is restricted: the calls that reach into a global namespace are gone. No `open()` by pathname, because pathnames are a global namespace. No `socket()` to a new connection. No `PID`-based signaling into the global process table. The man page is precise about the shape of it: some system calls requiring global namespace access are unavailable, while others are permitted, and everything reachable is reachable only through descriptors the process *already holds*. There is no way back out; a process in capability mode stays there (and its children start there).

A syscall that's forbidden in capability mode fails with a specific errno, `ECAPMODE`, distinct from ordinary permission errors, so you can tell "the sandbox refused this" apart from "the file wasn't there."

**Capability rights** are the fine-grained half. Capability mode takes away the ambient namespace, but you still hold open file descriptors, and by default a descriptor carries all its normal powers. Capsicum lets you *narrow* what a specific descriptor can do:

```c
cap_rights_t rights;

/* This fd may be read. That is all. Not written, not seeked,
   not fchmod'd, not anything else. */
cap_rights_init(&rights, CAP_READ);
cap_rights_limit(fd, &rights);
```

`cap_rights_limit()` restricts an individual descriptor to an explicit set of rights: `CAP_READ`, `CAP_WRITE`, `CAP_SEEK`, `CAP_FSTAT`, `CAP_MMAP`, `CAP_LOOKUP`, and dozens more, composable, from `capsicum(4)`. Try to do something to the descriptor beyond its granted rights and the call fails with `ENOTCAPABLE`, again a distinct errno, so "this fd isn't allowed to do that" is legible in your error handling.

Put together: `cap_enter()` removes the ability to name *new* resources, and `cap_rights_limit()` bounds what you can do with the resources you *already named*. Authority stops being ambient and becomes a property of the specific descriptors you're holding.

## The mental shift: openat, not open

The one practical consequence that reorganizes your code is that pathname lookups have to become *relative to a descriptor you hold*. In capability mode you can't `open("/etc/foo")`, but you can `openat(dirfd, "foo", ...)` where `dirfd` is a directory descriptor you opened *before* entering capability mode and limited to lookups. The pattern for a real program is therefore:

1. At startup, while you still have ambient authority, open the descriptors you'll need: your input file, an output directory, a logging socket.
2. Narrow each with `cap_rights_limit()` to exactly the operations you'll perform on it.
3. Call `cap_enter()`.
4. Do all the actual work through those pre-opened, rights-limited descriptors, using `openat()` relative to directory descriptors when you need to reach further.

Everything before step 3 runs with full authority and should do as little as possible; everything after runs in the sandbox. If the parser you run in step 4 has a memory-corruption bug, the attacker who triggers it inherits a process that cannot open new files, cannot reach the network, cannot signal other processes, and holds a handful of descriptors each limited to a couple of operations. The blast radius is the capabilities you explicitly granted, and nothing else.

## In practice: the caph_ helpers

The raw calls are the model; real programs use the convenience layer in `libcasper`'s `capsicum_helpers.h`, because a few things (like keeping `stdio` working, or `localtime()` needing timezone data it can no longer open) have standard solutions worth not re-deriving:

```c
#include <capsicum_helpers.h>

/* Limit stdin to read, stdout/stderr to write, in one call. */
caph_limit_stdio();

/* Cache timezone data now, so localtime() works after cap_enter. */
caph_cache_tzdata();

/* Enter capability mode (wrapper over cap_enter). */
caph_enter();
```

`caph_limit_stdio()` applies the obvious rights to the three standard streams so your `printf`/`fgets` keep working in the sandbox. `caph_cache_tzdata()` pre-loads timezone data (a classic gotcha: `localtime()` wants to open a file that capability mode forbids, so you warm it first). `caph_rights_limit()` is a thin wrapper over `cap_rights_limit`. These are what you'll actually call; the base `cat`, `kdump`, `dhclient`, and a growing set of tools use exactly this pattern.

## Casper: handing specific authority back

Capability mode is deliberately brutal: no DNS, because resolving a name needs to open `/etc/resolv.conf` and a socket, both forbidden. So how does a sandboxed network program resolve a hostname? This is where the design completes itself, and it's the subject of the next post in this series, but here's the shape.

Before entering capability mode, the process starts **Casper**, a service that stays *outside* the sandbox and performs specific, well-defined dangerous operations on the sandboxed process's behalf:

```c
#include <libcasper.h>
#include <casper/cap_dns.h>

cap_channel_t *casper = cap_init();               /* connect to Casper */
cap_channel_t *dns = cap_service_open(casper, "system.dns");
/* ... now enter capability mode ... */
cap_enter();
/* Inside the sandbox, DNS lookups go through the channel: */
struct addrinfo *res;
cap_getaddrinfo(dns, host, port, &hints, &res);   /* Casper does it for you */
```

The sandboxed process can't resolve names itself, but it holds a *channel* to a Casper service that can, and that channel only does DNS, nothing else. Casper offers a set of these narrow services: `system.dns` (name resolution), `system.net` (a restricted sockets API), `system.sysctl` (specific sysctl reads), `system.grp`/`system.pwd` (group and user lookups). Each hands back one specific slice of the authority `cap_enter()` took away. The communication runs over `libnv` (the typed, descriptor-passing serialization from the earlier post in this series, this is the wire format under Casper), which is why that library needed to pass file descriptors: Casper hands a resolved socket *back into* the sandbox as a descriptor.

So the full arc is: `cap_enter()` removes ambient authority, `cap_rights_limit()` bounds what your held descriptors can do, and Casper hands back exactly the dangerous operations you genuinely need, each as a narrow channel. Take everything, then return the minimum.

## The other-Unix contrast: capabilities vs syscall filtering

Linux's closest equivalent is seccomp, specifically seccomp-bpf, and the comparison is the most instructive one in this whole series, because the two are solving the same problem from opposite ends.

**seccomp filters syscalls.** You write a BPF program that inspects each syscall (number and argument register values) and decides allow or deny. It's a policy over the *syscall interface*: "this process may call `read` and `write` but not `open` or `socket`." Powerful, and the foundation of container sandboxing, but it has two structural awkwardnesses. First, it's a blocklist/allowlist you must get *exhaustively* right, over a syscall surface of 300-plus calls with subtle interactions; miss one and you've left a hole. Second, it filters on syscall *arguments as register values*, which means it can't easily reason about what a *file descriptor* actually refers to; it sees an int, not "a descriptor limited to reading one file."

**Capsicum limits capabilities.** Instead of policing every syscall, it removes the ambient authority that makes most syscalls dangerous, and attaches rights to the descriptors themselves. You don't enumerate forbidden syscalls; you enter a mode where the dangerous *category* (naming new global resources) is gone wholesale, and you carry explicit, per-descriptor authority for what remains. The security property is positive ("this process can do exactly these things") rather than negative ("this process can't do these particular things I remembered to block"). A bug can't grant authority that was never delegated, because there's no ambient authority sitting around to escalate into.

Neither is strictly better at everything (seccomp's syscall-level policy can express things Capsicum doesn't, and containers lean on it heavily), but the capability-oriented model is easier to reason about *soundly*: you specify what's allowed and the default is nothing, rather than specifying what's denied and hoping the default-allow didn't leave a gap. Capsicum shipped this model in FreeBSD 9.0 (2010), out of the TrustedBSD project, years before the surrounding ecosystem converged on capability-style thinking.

## Why "in base" matters here

Capsicum only works because it's in the base system, all the way down. The capability mode is a kernel-enforced property of a process, `cap_rights_limit` is enforced in the kernel's descriptor layer, the `caph_` helpers are in base, Casper and its services are in base, and the `libnv` transport underneath Casper is in base. You cannot bolt this on from a package, because it's not a library that intercepts things; it's the kernel changing what a process is permitted to do at the syscall boundary, with a userland toolkit built to match. Every piece has to agree, at the same version, and the way you get every piece to agree is to ship them together as one audited system. That is also why the earlier posts kept pointing here: the process descriptors (`pdfork`, so you can manage children without the global PID namespace capability mode forbids), the `EVFILT_PROCDESC` kqueue filter (so a sandboxed process can wait on those children through the one event interface it's still allowed), and `libnv`'s descriptor passing (so Casper can hand authority back across the boundary) are not separate clever features. They are the pieces of one capability-oriented design, and Capsicum is the center of it. The next post follows Casper in depth, the service that makes the sandbox survivable by handing back exactly what was taken.

---

*Man pages: `capsicum(4)` for the model overview, `cap_enter(2)` and `cap_getmode(2)` for capability mode, `cap_rights_limit(2)` and `rights(4)` for per-descriptor rights and the full `CAP_*` list, `capsicum_helpers(3)` for the `caph_*` convenience layer, and `libcasper(3)` with `cap_dns(3)`/`cap_net(3)`/`cap_sysctl(3)` for the Casper services. Capsicum has been in FreeBSD base since 9.0; it came out of the TrustedBSD project (Robert Watson, Jonathan Anderson, and others).*

*A note on the code: signatures and errno names (`ECAPMODE`, `ENOTCAPABLE`) here are from `sys/capsicum.h`, `cap_enter(2)`, and `cap_rights_limit(2)` in FreeBSD 15.1. The examples are skeletal; a real sandbox opens and rights-limits every descriptor it needs before `cap_enter()`, and the exact `CAP_*` rights for a given operation are enumerated in `rights(4)`, worth reading before you rely on a particular one.*
