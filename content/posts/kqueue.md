+++
title = "You've Heard of kqueue. Here's Why It's Actually Different (One Interface for Everything)"
date = 2026-06-28
draft = false
summary = "Sockets, timers, signals, file changes, process exit: on Linux those are five separate mechanisms. On FreeBSD they're one interface. kqueue isn't just an epoll competitor, it's a unification, and that's the part people miss."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

If you've spent any time around FreeBSD, you've heard kqueue praised. It comes up whenever someone explains why the network stack scales, or why a given daemon is fast. You've probably filed it as "FreeBSD's version of epoll" and moved on. That filing is the thing this post wants to fix, because kqueue is not epoll-with-a-different-name. The difference is the whole point, and once you see it you can't unsee it.

Here's the difference in one sentence: **epoll watches file descriptors; kqueue watches *events*, and an event can be almost anything the kernel knows about.** A socket becoming readable, yes, but also a timer firing, a signal arriving, a file being modified, a child process exiting, a jail being created. On Linux each of those is a separate subsystem with its own API. On FreeBSD they are one interface, one struct, one wait loop.

Let me show you before I explain.

## One loop, three completely different event sources

This program waits on three things at once: input on stdin, a timer that fires every two seconds, and the SIGINT signal (Ctrl-C). Three things that, on Linux, you'd reach for `epoll`, `timerfd`, and `signalfd` to handle. Here they go into one kqueue and come back out of one `kevent()` call.

```c
#include <sys/event.h>
#include <err.h>
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

int
main(void)
{
	int kq = kqueue();
	if (kq == -1)
		err(1, "kqueue");

	/* We want to *receive* SIGINT as a kevent, not have it kill us.
	   EVFILT_SIGNAL fires in addition to normal signal handling, so
	   we ignore the default action first; the filter still records it. */
	signal(SIGINT, SIG_IGN);

	struct kevent changes[3];
	/* 1: stdin is readable. ident is the fd. */
	EV_SET(&changes[0], STDIN_FILENO, EVFILT_READ, EV_ADD, 0, 0, NULL);
	/* 2: a 2-second repeating timer. ident is any id we choose. */
	EV_SET(&changes[1], 1, EVFILT_TIMER, EV_ADD, NOTE_SECONDS, 2, NULL);
	/* 3: SIGINT. ident is the signal number. */
	EV_SET(&changes[2], SIGINT, EVFILT_SIGNAL, EV_ADD, 0, 0, NULL);

	/* Register all three in one call. */
	if (kevent(kq, changes, 3, NULL, 0, NULL) == -1)
		err(1, "kevent register");

	printf("Type something, wait for the timer, or hit Ctrl-C.\n");

	for (;;) {
		struct kevent ev;
		/* Block until *any* of the three fires. One call. */
		int n = kevent(kq, NULL, 0, &ev, 1, NULL);
		if (n == -1)
			err(1, "kevent wait");
		if (n == 0)
			continue;

		/* Dispatch on which filter fired. */
		switch (ev.filter) {
		case EVFILT_READ: {
			char buf[256];
			ssize_t r = read(ev.ident, buf, sizeof(buf) - 1);
			if (r <= 0) {
				printf("stdin closed; exiting.\n");
				return (0);
			}
			buf[r] = '\0';
			printf("read %zd bytes from stdin\n", r);
			break;
		}
		case EVFILT_TIMER:
			/* data = how many times it fired since last wait. */
			printf("timer fired (x%lld)\n", (long long)ev.data);
			break;
		case EVFILT_SIGNAL:
			printf("caught SIGINT (delivered %lld times); bye.\n",
			    (long long)ev.data);
			return (0);
		}
	}
}
```

Build and run:

```sh
cc -o demo demo.c
./demo
```

Type a line: it reports the read. Wait: the timer ticks every two seconds. Hit Ctrl-C: it reports the signal and exits cleanly. Three event sources of completely different *kinds*, handled in one loop with no special-casing, no helper file descriptors, no separate APIs. That uniformity is what people are sensing when they say kqueue is elegant.

## The vocabulary is tiny

The reason it composes is that kqueue reduces every kind of event to one small struct. From `kqueue(2)`:

```c
struct kevent {
	uintptr_t ident;   /* identifier for this event */
	short     filter;  /* filter for event */
	u_short   flags;   /* action flags for kqueue */
	u_int     fflags;  /* filter-specific flags */
	int64_t   data;    /* filter-specific data value */
	void     *udata;   /* opaque user data, passed through unchanged */
};
```

An event is identified by the `(ident, filter)` pair. The `filter` says what *kind* of thing you're watching, and the `ident` means whatever that filter says it means: for `EVFILT_READ` it's a file descriptor, for `EVFILT_TIMER` it's an arbitrary id you pick, for `EVFILT_SIGNAL` it's a signal number, for `EVFILT_PROC` it's a PID. The `flags` are the action (add it, delete it, make it one-shot). The `fflags` and `data` carry filter-specific parameters in and results out. The `udata` is yours, passed through untouched, which in real programs is where you stash a pointer to the connection or state object the event belongs to.

`EV_SET()` populates one of these, you hand an array of them to `kevent()` as the *changelist*, and the same call hands you back an array of fired events as the *eventlist*. Register and wait are the same syscall. That's the entire model.

## The filter list is the actual feature

Here's where "it's just epoll" falls apart. The power isn't the struct, it's the roster of filters, because each one is a category of thing that is a *separate mechanism* on Linux. Walking the list from `kqueue(2)`, with the Linux counterpart for each:

**`EVFILT_READ` / `EVFILT_WRITE`** are the familiar ones: a descriptor is readable or writable. This is the part that overlaps epoll. The `data` field even hands you useful detail, like the size of the listen backlog on a listening socket, or the bytes available to read. *(Linux: epoll.)*

**`EVFILT_TIMER`** establishes a timer with no file descriptor involved at all. You pick an id, set `NOTE_SECONDS`/`NOTE_MSECONDS`/etc. in `fflags`, and it fires periodically (or once, with `EV_ONESHOT`). On return, `data` tells you how many times it expired since you last looked. *(Linux: timerfd, a separate fd-creating syscall.)*

**`EVFILT_SIGNAL`** delivers signals as events. One subtlety the man page is explicit about: the filter records the signal even if it's marked `SIG_IGN`, and it fires *after* normal signal processing, so the idiom is to ignore the default action first (as the demo does) and then receive the signal through the queue. `data` counts how many arrived. *(Linux: signalfd, again a separate mechanism.)*

**`EVFILT_VNODE`** watches a file or directory for changes via a rich set of `fflags`: `NOTE_WRITE`, `NOTE_DELETE`, `NOTE_RENAME`, `NOTE_ATTRIB`, `NOTE_EXTEND`, `NOTE_LINK`, and more. A file-change watcher in a handful of lines. *(Linux: inotify, an entirely separate API with its own fd type and read-the-events protocol.)*

**`EVFILT_PROC`** watches a process by PID for `NOTE_EXIT`, `NOTE_FORK`, `NOTE_EXEC`, even `NOTE_TRACK` to follow it across forks. On exit, `data` carries the wait status. *(Linux: historically waitpid plumbing; more recently pidfd.)*

**`EVFILT_PROCDESC`** watches a *process descriptor* from `pdfork(2)` for its exit. If you read the process-descriptor post in this series, this is the other half of that story: the descriptor drops straight into your kqueue loop, so a child process's exit is just another event next to your sockets and timers. *(Linux: pidfd in epoll, the same idea arrived at later.)*

And it keeps going: **`EVFILT_USER`** for events your own code triggers (a clean way to wake the loop from another thread), **`EVFILT_AIO`** for async I/O completion, **`EVFILT_EMPTY`** for "the write buffer drained," and even **`EVFILT_JAIL`** / **`EVFILT_JAILDESC`** for watching jail lifecycle events. Every one of these is a thing you'd otherwise wait on through a different door.

The list is the argument. Five or six unrelated Linux subsystems, each with its own API and fd semantics, are here as entries in one filter enum, reached through one struct and one syscall.

## Why this stays coherent: filters are an abstraction

There's a design reason kqueue could absorb all of this without becoming a mess, and it's worth seeing because it's the deeper "in base" point.

The man page describes a filter as a small piece of kernel code: kqueue notifies you "based on the results of small pieces of kernel code termed filters." A filter is an internal abstraction with a defined contract (evaluate a condition, optionally aggregate, report). Adding a new kind of watchable event means writing a new filter, not adding a new syscall or a new user-facing fd type. That's why `EVFILT_PROCDESC` could appear to support process descriptors, and `EVFILT_JAILDESC` for jail descriptors, slotting into the same `kevent()` interface every existing program already speaks.

Contrast the Linux trajectory. epoll, timerfd, signalfd, inotify, and pidfd were each added separately, at different times, each its own syscall surface, because there was no single extensible event abstraction to hang them on. They arrive at roughly the same destination (you can funnel several of them into one epoll), but as five mechanisms federated together rather than one mechanism extended. kqueue made the opposite bet in 2000: define the event abstraction first, then every new event source is just another filter. Same problem, opposite architecture, and the architecture is why kqueue reads as one coherent thing instead of a federation.

## The other-Unix contrast, stated plainly

This isn't FreeBSD being earlier and Linux catching up, exactly. epoll is excellent at what it does and scales superbly. The honest framing is that the two systems made different structural choices. Linux grew specialized, highly-optimized mechanisms per event type and gave you epoll to combine the fd-shaped ones. FreeBSD defined one event-notification abstraction up front and expressed every event type as a filter within it. If you write portable network code you'll meet both, and the concepts map; what doesn't map is the *shape*. On FreeBSD, "wait for any of these unrelated things" is the native, obvious operation. On Linux it's an assembly job.

There's real pedigree here too: kqueue shipped in FreeBSD 4.1 in 2000, designed by Jonathan Lemon, with a USENIX paper laying out the generic-and-scalable-notification argument. It's not a recent flourish; it's a 25-year-old design decision that the rest of the base system has been hooking into ever since.

## Why "in base" matters here

The unification is a *consequence* of kqueue being in base. Because the event abstraction ships and versions with the kernel, every other base subsystem can express its events as kqueue filters rather than inventing a private notification path. Process descriptors hook in. Jails hook in. AIO hooks in. The vnode layer hooks in. That shared primitive is why "watch a socket, a timer, a signal, and a child process in one loop" is a normal thing to write on FreeBSD instead of a five-API integration exercise. A unified event interface only stays unified if everything agrees to use it, and the way you get everything to agree is to ship it in the base system, at a known version, that every other part of the system is built against.

So: yes, you've heard of kqueue. The thing worth knowing is that it isn't an epoll alternative you reach for in the same slot. It's a different and broader idea, watch *events*, not just descriptors, and that idea is the reason FreeBSD people keep bringing it up.

---

*Man pages: `kqueue(2)` for the full filter roster, the `kevent` struct, and every `EVFILT_`/`EV_`/`NOTE_` flag referenced here; `EV_SET(3)` for the struct-initializer macro; `sigaction(2)` and `signal(3)` for how `EVFILT_SIGNAL` coexists with normal signal handling. The original design is in Jonathan Lemon's 2001 USENIX paper, "Kqueue: A Generic and Scalable Event Notification Facility." kqueue has been in FreeBSD base since 4.1 (2000); `kqueuex()` and `kqueue1()` were added in 14.0.*
