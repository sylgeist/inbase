+++
title = "The Process-as-a-File-Descriptor Idea Finally Got Its Missing Piece"
date = 2026-06-22
draft = false
summary = "Process descriptors have been in FreeBSD base since 9.0, but until 15.1 you couldn't reap exit status by descriptor. pdwait closes the loop."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Here's a thing you can do on a stock FreeBSD 15.1 box, no packages, no kernel module:

```c
int pd;
pid_t pid = pdfork(&pd, 0);
if (pid == 0) {
    /* child */
    execlp("sleep", "sleep", "2", NULL);
    _exit(127);
}

/* parent: `pd` is a *file descriptor* that refers to the child.
   No PID handling. Wait on it like anything else. */
int status;
pdwait(pd, &status, 0, NULL, NULL);
printf("child exited: %d\n", WEXITSTATUS(status));
close(pd);
```

That `pdwait` call is the news. The rest of this — a child process you hold as a file descriptor instead of a PID — has been in FreeBSD base since 9.0 in 2012. But until 15.1, shipped this past April, the descriptor couldn't actually *do* the last step you see above. You could hold the process as an fd, signal it as an fd, poll it as an fd — and then, to collect its exit status, you had to fall back to its PID and call `wait6`, walking right back into the exact problem the descriptor was invented to avoid. 15.1 is the release that closes the loop.

If you've reached for Linux's `pidfd` and `waitid()`-on-pidfd, this is the same idea — and FreeBSD just arrived at completeness on its own track. More on that convergence at the end.

## The problem with PIDs

If you've written a process supervisor, a test runner, or anything that spawns and reaps children on a Unix, you've met the PID-reuse race, even if you didn't name it.

A PID is just a small integer the kernel hands back from `fork()`. The trouble is that it's not *yours*. The moment your child dies and you reap it, that integer goes back into the pool, and the kernel is free to hand it to some unrelated process the next time anyone calls `fork()`. PIDs are recycled, and on a busy machine they recycle fast.

So this code has a bug:

```c
pid_t pid = fork();
/* ... time passes, child may have died and been reaped ... */
kill(pid, SIGTERM);   /* are you signalling your child, or a stranger? */
```

Between obtaining `pid` and using it, the process it named can exit, get reaped (by you, or by a `SIGCHLD` handler, or by a library you linked that installed one behind your back), and have its number reissued. Your `kill` now lands on whatever inherited the number. In a supervisor that sends signals based on stored PIDs, this is a genuine "send SIGKILL to the wrong process" footgun, and it's miserable to reproduce because it depends on timing and PID-space pressure.

The usual mitigations are all *discipline*: keep a tight `SIGCHLD` handler, never store a PID you might use after the child could exit, serialize your reaping. None of it is enforced by the kernel. The race lives in the gap between "I have a number" and "the number still means what I think."

## The fix: name the process with a descriptor, not a number

FreeBSD's answer, out of the TrustedBSD/Capsicum work, is `pdfork(2)` — "process-descriptor fork." It forks like `fork()`, but instead of *only* giving you a PID, it writes a file descriptor into an `int` you hand it:

```c
#include <sys/procdesc.h>

int pd;
pid_t pid = pdfork(&pd, 0);
```

In the parent, `pid` is the normal PID *and* `pd` is a file descriptor that refers specifically to that child. The descriptor is the part that matters. Unlike the PID, the descriptor is a reference you own. As the man page puts it, the PID of the referenced process is not reused until the process descriptor is closed — whether or not the zombie has been reaped. So as long as you're holding `pd`, the identity is pinned. There's no window where the number silently re-points at a stranger, because you're not using the number; you're using the fd.

A few consequences fall out of "it's a file descriptor":

**It doesn't raise `SIGCHLD`.** A `pdfork`'d child won't fire the asynchronous `SIGCHLD` machinery on termination. That sounds like a loss until you realize `SIGCHLD` is half the problem — it's the thing that lets some other part of your program reap the child out from under you. With process descriptors, the child's exit is an event *on the descriptor*, delivered to whoever is watching the descriptor, not a global signal anyone can catch.

**It has lifetime semantics.** By default the process is *terminate-on-close*: if you `close(pd)` while the child is alive and that was the last reference, the kernel sends it `SIGKILL`. The child's lifetime is tied to the descriptor's. That's often exactly what you want for a supervised worker — if the supervisor crashes and its fds are reclaimed, the workers don't leak. If you *don't* want that, `pdfork` takes a `PD_DAEMON` flag to let the process outlive the descriptor (killable only explicitly via `kill`). There's also `PD_CLOEXEC` to set close-on-exec on the descriptor itself.

**It signals by fd.** `pdkill(pd, SIGTERM)` is `kill` with a descriptor instead of a PID — same semantics, none of the reuse hazard. You're signalling *this* process, the one the descriptor refers to, full stop.

## Watching for exit, the old way

Here's where, before 15.1, the design had a visible seam.

You can wait for a process descriptor to become "the process died" using the normal readiness machinery, because it's an fd:

- `poll(2)`/`select(2)`: the descriptor raises `POLLHUP` when the process dies. (That's the only event currently defined for it.)
- `kqueue(2)`: register it with the `EVFILT_PROCDESC` filter and you get `NOTE_EXIT` when it exits.

That's great for *integration*: a process descriptor drops into the same `kqueue` loop you're already using for sockets, timers, and signals. One event loop, one readiness model, process exit is just another source. For an experienced Unix hand this is the genuinely lovely part — process lifetime stops being a special-case `SIGCHLD` island and becomes one more thing the event loop knows about.

But notice what `POLLHUP`/`NOTE_EXIT` give you: *notification*. They tell you the process **is dead**. They don't give you the **exit status** — the `WEXITSTATUS`, the signal that killed it, the rusage. To get *that*, pre-15.1, you had to do this:

```c
/* descriptor told us NOTE_EXIT; now actually reap it... */
pid_t pid;
pdgetpid(pd, &pid);          /* recover the PID from the descriptor */
wait6(P_PID, pid, &status, WEXITED, NULL, NULL);   /* ...by PID. ugh. */
```

And there's the whole problem walking back in the door. To collect status, you dropped from the safe fd world back to the PID world — `pdgetpid` then `wait6(P_PID, ...)`. The descriptor's PID-pinning guarantee saves you here *in practice* (the number is still valid because you still hold `pd`), but you're round-tripping through exactly the API the descriptor was supposed to replace, and the ergonomics are backwards: spawn-by-fd, signal-by-fd, poll-by-fd, then... reap-by-PID. The lifecycle handle had a hole in it right at the end.

## What 15.1 added: `pdwait`

`pdwait(2)` is the missing reap-by-descriptor call. Per the man page, it lets the calling thread wait for and retrieve the status information on the process referenced by the descriptor, following `wait6`'s behavior specification. The signature carries the full `wait6` payload:

```c
int pdwait(int fd, int *status, int options,
           struct __wrusage *wrusage, struct __siginfo *info);
```

So you don't just get a status int — you get the optional `__wrusage` (self + children resource usage) and `__siginfo` blocks too, same as `wait6`. The whole point: everything `wait6` could tell you, now keyed on the descriptor instead of a recyclable integer. `pdgetpid` and `wait6(P_PID, ...)` disappear from the reap path.

The lifecycle is finally closed, every step keyed on the fd:

1. **Spawn** — `pdfork(&pd, 0)`
2. **Signal** — `pdkill(pd, SIGTERM)`
3. **Wait for exit** — `kqueue`/`EVFILT_PROCDESC`/`NOTE_EXIT`, or `poll`/`POLLHUP`
4. **Reap status** — `pdwait(pd, &status, ...)`
5. **Release** — `close(pd)`

No PID touches steps 1–5. The number exists (you can still fetch it with `pdgetpid` if you need it for logging), but it's no longer load-bearing anywhere in the spawn/signal/wait/reap cycle. That's what "the descriptor is now a complete lifecycle handle" means concretely.

There's a companion new call too: `pdrfork(2)`, an `rfork`-flavored variant that takes `rfflags` to control resource sharing between parent and child (it requires `RFPROC`+`RFPROCDESC`, or `RFSPAWN`). Same release, same authorship. If `pdfork` is "process-descriptor `fork`," `pdrfork` is "process-descriptor `rfork`" — for when you want descriptor semantics *and* `rfork`'s control over what the child shares.

## A complete, runnable example

This spawns a child as a descriptor, waits for its exit through a `kqueue` loop (the integration payoff), then reaps the real status with `pdwait` (the 15.1 payoff) — no PID anywhere in the lifecycle. Build and run on FreeBSD 15.1.

```c
/* pdwait_demo.c — build: cc -o pdwait_demo pdwait_demo.c */
#include <sys/types.h>
#include <sys/procdesc.h>
#include <sys/event.h>
#include <sys/wait.h>
#include <sys/resource.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int
main(void)
{
	int pd;
	pid_t pid = pdfork(&pd, 0);
	if (pid < 0) {
		perror("pdfork");
		return (1);
	}

	if (pid == 0) {
		/* Child: do a little work, then exit with a distinctive code. */
		sleep(1);
		_exit(42);
	}

	/* Parent. We never touch `pid` again — `pd` is the handle. */
	printf("spawned child as descriptor %d\n", pd);

	/* Put process exit into a kqueue loop, alongside whatever else
	   (sockets, timers, signals) a real program would register here. */
	int kq = kqueue();
	struct kevent ev;
	EV_SET(&ev, pd, EVFILT_PROCDESC, EV_ADD, NOTE_EXIT, 0, NULL);
	if (kevent(kq, &ev, 1, NULL, 0, NULL) < 0) {
		perror("kevent register");
		return (1);
	}

	/* Block until the descriptor reports the process exited. */
	struct kevent out;
	if (kevent(kq, NULL, 0, &out, 1, NULL) < 0) {
		perror("kevent wait");
		return (1);
	}
	printf("kqueue: child reported NOTE_EXIT\n");

	/* 15.1: reap the *status* straight off the descriptor.
	   No pdgetpid + wait6(P_PID, ...) dance. */
	int status;
	struct __wrusage wru;
	if (pdwait(pd, &status, 0, &wru, NULL) < 0) {
		perror("pdwait");
		return (1);
	}

	if (WIFEXITED(status))
		printf("pdwait: child exited with code %d\n",
		    WEXITSTATUS(status));
	else if (WIFSIGNALED(status))
		printf("pdwait: child killed by signal %d\n",
		    WTERMSIG(status));

	printf("child user CPU: %ld.%06ld s\n",
	    (long)wru.wru_children.ru_utime.tv_sec,
	    (long)wru.wru_children.ru_utime.tv_usec);

	close(pd);   /* release the handle */
	close(kq);
	return (0);
}
```

Expected output:

```
spawned child as descriptor 3
kqueue: child reported NOTE_EXIT
pdwait: child exited with code 42
child user CPU: 0.00xxxx s
```

Swap the child's `_exit(42)` for something that takes a signal, and you'll see the `WIFSIGNALED` branch report it — all the `wait6` status decoding works because `pdwait` *is* `wait6` semantics keyed on the descriptor.

## Why this composes with Capsicum

There's a reason process descriptors came out of the Capsicum project specifically, and it's worth seeing because it's where the design pays off most.

Capsicum's capability mode (`cap_enter(2)`) drops a process into a world where it can only act through descriptors it already holds — no global namespaces. No opening paths by name, no looking up arbitrary PIDs. In that world, the *entire* PID-based process API is unavailable to you: you can't `kill(pid, ...)` a number you pulled from somewhere, because reaching a process by global identifier is exactly the ambient authority capability mode takes away.

Process descriptors are how you manage children *at all* inside that sandbox. The descriptor is a capability: holding it is your authorization to signal and wait on that specific process and no other. The rights even subdivide — `pdkill` needs `CAP_PDKILL` on the descriptor, and an under-privileged descriptor returns `ENOTCAPABLE`. A sandboxed supervisor can hold a descriptor it's allowed to *wait* on but not *kill*, and the kernel enforces that split.

But until 15.1, a Capsicumized process that spawned children had that ugly gap: it could `pdfork` and `pdkill` and watch for `NOTE_EXIT`, but to reap status it needed... a PID-based `wait6`, which is precisely the kind of global-namespace call capability mode is supposed to forbid. `pdwait` removes the last reason a sandboxed process would need to reach outside the descriptor model to manage its own children. That's the deeper significance of this particular syscall landing: it's not just ergonomic sugar, it's the piece that lets the capability story be *complete* for process management. (We'll build a fully-sandboxed supervisor in the Capsicum and Casper posts — this is the primitive it stands on.)

## The Linux contrast: convergent evolution

If this all feels familiar from Linux, you're right, and the honest framing is convergence, not catch-up. Linux grew `pidfd_open(2)` / the `CLONE_PIDFD` flag to `clone`, and then `waitid()` learned to take a pidfd via `P_PIDFD`. That's the same destination: a process referenced by a descriptor, signalled with `pidfd_send_signal`, waited on by fd, droppable into `epoll`. Two Unix lineages independently concluded that the PID is the wrong handle and a descriptor is the right one.

The differences are mostly in framing. FreeBSD's version was born inside a capability system — the descriptor *is* a capability, with subdividable rights (`CAP_PDKILL`) and a natural home in `cap_enter` sandboxes — because that's the problem it was built to solve in 2012. Linux's arrived later as a general fix for the reuse race and slotted into its existing `epoll`/`clone` world. Same insight, different center of gravity. Worth knowing both if you write portable supervisors: the concepts map cleanly, the spellings don't.

## Why "in base" matters here

You can write all of the above against a stock FreeBSD 15.1 install — `#include <sys/procdesc.h>`, link nothing special, read `pdfork(2)` for the whole family in one man page. The syscall, the kqueue filter, the capability rights, and the documentation shipped and versioned together, at a known release, the same April. When you're building something that depends on race-free process management, "is this primitive present and complete on the box I'm deploying to?" has a single answer tied to the OS version, not a matrix of kernel configs and library versions. The hole in the lifecycle got filled in 15.1 — and "15.1" is the entire compatibility story you need to track.

---

*Man pages: `pdfork(2)` (covers `pdfork`/`pdrfork`/`pdgetpid`/`pdkill`/`pdwait`), `procdesc(4)`, `capsicum(4)`, `cap_enter(2)`, `kqueue(2)`, `wait6(2)`. The `pdrfork`/`pdwait` additions are credited to Konstantin Belousov with input from Alan Somers; the original family to Robert Watson and Jonathan Anderson out of the TrustedBSD/Capsicum work at Cambridge.*
