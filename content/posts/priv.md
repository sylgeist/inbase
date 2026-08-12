+++
title = "Root Isn't a Monolith: How FreeBSD Split Superuser Into ~200 Named Privileges (priv)"
date = 2026-05-29
draft = false
summary = "The reason a jail's root can bind low ports but not reboot the host isn't a special case. FreeBSD long ago broke 'uid 0 can do anything' into a couple hundred specific, named privilege checks. That decomposition is why partial root exists at all, and it's a quietly radical piece of the security architecture."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE. This one is a look under the hood rather than an API you'll call from an app: it's about a design decision, not a function you invoke.*

Here's a thing you've probably observed without asking why. Root inside a jail can do many privileged things, bind to low-numbered ports, change file ownership, load the jail's own state, but it *can't* reboot the host, load kernel modules, or reach outside its filesystem. "Jail root is weaker than real root," everyone says, and moves on. But *how*? If root is defined as "uid 0 can do anything," there's no natural place to stand between "all-powerful" and "powerless." The fact that partial root exists at all means root was never actually a monolith in the FreeBSD kernel. It was taken apart, deliberately, into pieces you can grant and deny individually.

This post is about that decomposition. It's a kernel-internals story rather than a userland API, but it explains behavior you see from userland every day, and it's one of the more quietly consequential design decisions in the system.

## The old way: "are you root?"

For most of Unix history, a privileged operation in the kernel was gated by a single question: is the effective uid zero? The classic check, in old BSD, was `suser()`, "super-user," and it answered exactly one thing: yes or no, are you root. Every privileged operation in the kernel, from mounting a filesystem to changing the system clock to opening a raw socket, called the same `suser()` and got the same binary answer.

That design has a hard consequence: privilege is all-or-nothing. There's no vocabulary to say "this process may change the clock but may not mount filesystems," because the only concept the kernel has is "root" versus "not root." If you want to grant a sliver of root's power, you can't, because root's power isn't divided into slivers. It's one indivisible bit.

## The FreeBSD way: name every privilege

FreeBSD replaced `suser()` wholesale with a different model, the `priv(9)` framework, and the shift is the whole story. Instead of one check answering "are you root," the kernel now asks a *specific* question at each privileged operation:

```c
#include <sys/priv.h>

int priv_check(struct thread *td, int priv);
```

The second argument, `priv`, names *which* privilege is being requested. Not "are you root," but "does this thread hold `PRIV_VFS_MOUNT`?" or "does it hold `PRIV_NETINET_RESERVEDPORT`?" or "does it hold `PRIV_REBOOT`?" There are on the order of two hundred of these named constants in `sys/priv.h`, one for each distinct privileged thing the kernel can do, grouped by subsystem: `PRIV_VFS_*` for filesystem operations, `PRIV_NET_*` and `PRIV_NETINET_*` for networking, `PRIV_VM_*` for virtual memory, `PRIV_CLOCK_SETTIME` for the clock, `PRIV_REBOOT` for rebooting, and so on across every corner of the system.

The man page frames what these represent precisely: a privilege is typically either the right to manage a particular component of the system, or an exemption from a specific policy or access control. Mounting a filesystem is the first kind; overriding file permissions is the second. Every one of them is a named, numbered constant, and the operation deep in the kernel that used to call `suser()` now calls `priv_check(td, PRIV_SOMETHING_SPECIFIC)`.

That sounds like a mechanical refactor, "replace one function with a more specific one," but naming the privileges is what makes everything else possible. Once each privileged operation asks for a *specific* privilege by name, something can sit between the request and the answer and decide *per privilege* whether to grant it. Root stops being one bit and becomes a couple hundred bits, and now you can hold some and not others.

## The policies: who grants which privileges

`priv_check` doesn't just check a uid. It runs the request through the system's privilege *policies*, and this is where partial root comes from. The man page names two base policies:

The **superuser policy** is the familiar one: if your effective uid is 0, it grants the privilege. This is backward-compatible with "root can do anything", a normal root process passes every `priv_check` because the superuser policy says yes to all of them. So on a plain system, nothing looks different; root is still root.

The **jail policy** is where the decomposition pays off. When the process is in a jail, a different function, `prison_priv_check()`, gets consulted, and it holds an explicit list: *which specific privileges is root-in-a-jail allowed to have?* It's not "jail root is weaker" as a vague property; it's a concrete, enumerated set. `PRIV_NETINET_RESERVEDPORT` (bind a low port)? Granted in a jail. `PRIV_REBOOT`? Not granted. `PRIV_VFS_MOUNT`? Not granted (by default). The jail's root passes `priv_check` for the privileges on the list and fails for the ones that aren't, operation by operation.

*That* is the mechanism behind everything you observe about jail root. There's no special "is this a dangerous operation" heuristic; there's a per-privilege allowlist, and the reason a given operation is permitted or denied inside a jail is simply whether its `PRIV_*` constant is in `prison_priv_check()`'s set. Partial root is a list of names.

And the framework is extensible in the same direction: the TrustedBSD **MAC framework** (`mac(9)`) can also influence the outcome of a `priv_check`, so a security module can grant or deny specific privileges on its own policy, layered on top of the uid and jail logic. The named-privilege design is what lets all of these compose: because every privileged operation asks for a specific privilege, any policy, superuser, jail, MAC, can answer per privilege rather than all-or-nothing.

## Why this is the same idea as Capsicum, from the other end

If you read the Capsicum post in this series, this should feel related, and it is, they're the same instinct applied at different layers. Capsicum takes a process's *ambient authority* away and hands back specific capabilities. `priv` takes *root's monolithic power* apart and hands back specific privileges. Both replace an all-or-nothing authority model with a fine-grained, enumerable one. Both are the work of the same lineage (the `priv` framework was built by Robert Watson, who also drove Capsicum and the TrustedBSD project). And both exist because "this process is trusted, so it can do everything" is a security dead end: the moment you want to trust something *partially*, you need the power divided into nameable pieces. `priv` divides root; Capsicum divides ambient authority. The jail that holds a subset of privileges *and* runs its workload under Capsicum is using both decompositions at once.

## The other-Unix contrast: capabilities, differently sliced

Linux also broke up root, into POSIX **capabilities** (`CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`, and about forty others), and the motivation is identical: make root divisible. The comparison is worth drawing honestly because both are real and they made different tradeoffs.

Linux capabilities are a *property of the process and its files*: a process holds a capability set, executables carry file capabilities, and the kernel checks "does this process hold `CAP_X`." FreeBSD's `priv` is a *check performed by the kernel at the operation*, resolved through policies (uid, jail, MAC) rather than carried as a per-process bitmask. One practical difference is granularity and its discontents: Linux's `CAP_SYS_ADMIN` became a notorious catch-all, so many operations were lumped under it that holding it is nearly root anyway, whereas FreeBSD's set is finer in places (though it has its own catch-all, `PRIV_DRIVER`, deliberately, for device drivers rather than minting a privilege per driver). Another is where the policy lives: FreeBSD threads jail-awareness directly into the privilege check, so "what can jail root do" is answered in the same place every privilege is defined, which is why the jail privilege model feels so integrated. Neither is strictly better; they're two designs for the same goal, root as a set of nameable powers instead of one bit.

## Why "in base" matters here

This one isn't a library you link; it's the shape of the kernel's security model, and that it's in base, uniform across the whole system, is exactly the point. Because *every* privileged operation in the base kernel was converted to ask for a specific named privilege through the one `priv_check` interface, a single policy change (adding a privilege to the jail allowlist, writing a MAC module) applies consistently everywhere, rather than each subsystem inventing its own notion of "trusted enough." The privilege constants are part of the kernel module ABI, so they're stable, versioned with the system, and shared by every driver and subsystem. The decomposition only works because it's total and centralized: one framework, in base, that every privileged code path routes through. That's what turns "root can do anything" into "root's power is this specific, enumerable, individually-grantable set", and it's why, when you watch a jail's root bind a low port but fail to reboot the host, you're seeing an architecture decision, not a special case.

---

*Man pages: `priv(9)` for the framework (`priv_check`, `priv_check_cred`, the superuser and jail policies), `jail(2)` for the jail side and how `prison_priv_check` bounds jailed root, and `mac(9)` for the MAC framework's role. The complete privilege list lives in `sys/priv.h`, worth skimming once just to see the full surface of what "root" was decomposed into. The `priv` framework replaced the older `suser(9)` interface and was created by Robert Watson.*

*A note: this is a kernel-internals topic, so unlike most posts in this series there's no userland program to run. If you want to see the privilege set, read `sys/priv.h`; if you want to see the jail policy, read `prison_priv_check()` in the kernel source. The specific `PRIV_*` names used as examples here are from `sys/priv.h` in the FreeBSD source; the exact set granted to jailed root is defined in `prison_priv_check()` and is the authoritative reference for what jail root can and cannot do.*
