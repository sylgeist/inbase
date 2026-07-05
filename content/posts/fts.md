+++
title = "You Keep Writing readdir Loops. The Tools You Use Don't (fts)"
date = 2026-06-10
draft = false
summary = "Walking a directory tree correctly means handling symlink loops, unreadable subdirs, deep nesting, and pre- vs post-order traversal. rm, cp, chmod, ls, and du don't hand-roll that with readdir. They use fts, which is in base and handles the hard parts for you."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Every C programmer, at some point, writes the directory walk. `opendir`, loop on `readdir`, `stat` each entry, recurse into subdirectories, and you have a tree traversal. It works on your home directory. Then it meets a symlink that points at an ancestor and loops forever, or a directory you can't read, or a tree deep enough to exhaust your file descriptors because you kept them open across the recursion, or you realize you need to process a directory *after* its contents (to delete it) and your preorder walk can't. Each of these is a patch, and the patched-up result is a worse version of something already in the base system.

The tell is which programs *don't* hand-roll it. Look at what `rm`, `cp`, `chmod`, `ls`, `du`, `chflags`, `pax`, and `setfacl` have in common: they all recurse through directories, and not one of them uses a raw `readdir` loop to do it. They use `fts`, the file-tree-traversal library in libc. If the tools whose entire job is walking trees correctly reach for `fts`, the `readdir` loop in your program is probably reinventing it with more bugs.

## What the walk looks like with fts

Here's a complete tree walk that prints every file with its size and depth. No recursion in your code, no `opendir`, no manual stack:

```c
#include <fts.h>
#include <stdio.h>
#include <string.h>

int
main(int argc, char **argv)
{
	/* fts_open takes a NULL-terminated array of paths. */
	char *paths[] = { argv[1], NULL };

	/* FTS_PHYSICAL: don't follow symlinks (see their targets as links).
	   fts stats each entry for you unless you ask it not to. */
	FTS *ftsp = fts_open(paths, FTS_PHYSICAL, NULL);
	if (ftsp == NULL) {
		perror("fts_open");
		return (1);
	}

	FTSENT *ent;
	while ((ent = fts_read(ftsp)) != NULL) {
		switch (ent->fts_info) {
		case FTS_F:            /* a regular file */
			printf("%*s%s (%lld bytes)\n",
			    (int)ent->fts_level * 2, "",
			    ent->fts_name, (long long)ent->fts_statp->st_size);
			break;
		case FTS_D:            /* a directory, preorder (on the way in) */
			printf("%*s%s/\n",
			    (int)ent->fts_level * 2, "", ent->fts_name);
			break;
		case FTS_DNR:          /* directory we can't read */
			fprintf(stderr, "%s: unreadable (%s)\n",
			    ent->fts_path, strerror(ent->fts_errno));
			break;
		case FTS_ERR:
		case FTS_NS:           /* stat failed */
			fprintf(stderr, "%s: %s\n",
			    ent->fts_path, strerror(ent->fts_errno));
			break;
		}
	}

	fts_close(ftsp);
	return (0);
}
```

```sh
cc -o walk walk.c
./walk /etc
```

`fts_open` sets up the traversal over one or more root paths; `fts_read` returns one `FTSENT` per visit; `fts_close` tears it down. Your loop is flat. The recursion, the descent, the stat calls, and the bookkeeping happen inside the library. What you write is a `switch` on what kind of thing you're looking at.

## The `fts_info` field is the whole point

That `switch` is where `fts` earns its place, because `ent->fts_info` tells you *what situation you're in*, and the set of situations is exactly the list of things your hand-rolled walk gets wrong. Straight from `fts.h`, these are the cases the library distinguishes for you:

- **`FTS_D`** a directory, visited in **preorder** (on the way down)
- **`FTS_DP`** the same directory in **postorder** (on the way back up, after its contents). This is the one a naive walk can't do without extra work, and it's exactly what `rm -rf` needs: you can't remove a directory until you've removed what's inside it, so you act on `FTS_DP`.
- **`FTS_DC`** a directory that **causes a cycle**. `fts` detects that this directory would send you into a loop (a symlink back up the tree, say) and *tells* you instead of spinning forever. The infinite-loop bug in the hand-rolled version simply cannot happen; the library hands you `FTS_DC` and moves on.
- **`FTS_DNR`** a directory you **can't read**. Not a crash, not a silent skip: a labeled event with `fts_errno` set, so you decide what to do.
- **`FTS_SL`** / **`FTS_SLNONE`** a symlink, and specifically a symlink whose **target doesn't exist**. The dangling-symlink case you'd otherwise discover by accident.
- **`FTS_NS`** a file `fts` **couldn't stat**, distinguished from one it could, again with `fts_errno`.
- **`FTS_F`**, **`FTS_DOT`**, **`FTS_DEFAULT`** regular files, the `.`/`..` entries (which it can filter or surface), and everything else.

Every one of those is a bug waiting in a `readdir` loop. The cycle detection alone (`FTS_DC`) is the kind of thing people ship without and then discover in production when a backup job or a `chmod -R` wanders into a symlink loop and never returns. `fts` does it as a matter of course.

And each `FTSENT` hands you what you'd have gathered by hand anyway: `fts_path` (full path for this visit), `fts_name` (just the filename), `fts_level` (depth, so you don't track it yourself), `fts_statp` (a `struct stat *`, already populated, so no separate `stat` call), and `fts_errno` (per-entry error). The `st_size` in the demo above came free; `fts` already did the `stat`.

## The controls you'd otherwise build

`fts_open`'s options cover the traversal decisions you'd otherwise hand-code:

- **`FTS_PHYSICAL`** vs **`FTS_LOGICAL`** don't-follow vs do-follow symlinks, the single most common source of "wait, why did it recurse into that." One flag.
- **`FTS_NOSTAT`** skip the per-entry `stat` when you only need names (faster; you get `FTS_NSOK` entries).
- **`FTS_XDEV`** don't cross filesystem (mount-point) boundaries, the "stay on this device" behavior that `du -x` and friends expose.
- **`FTS_SEEDOT`** include `.` and `..` if you actually want them.

There's also `fts_set(ftsp, ent, FTS_SKIP)` to tell the walk "don't descend into this one after all," decided mid-traversal, and `fts_children()` to peek at a directory's contents as a linked list without iterating. You get a custom sort order by handing `fts_open` a comparison function, which is how `ls` produces sorted recursive output without sorting anything itself.

The point isn't any single flag. It's that the decisions a directory walk actually involves (follow symlinks or not, stat or not, cross devices or not, pre- or post-order, skip-this-subtree, sorted or not) are *parameters* here, not code you write and debug. You're configuring a correct traversal instead of authoring a buggy one.

## The other-Unix contrast

The portable POSIX answer is `ftw(3)` and its better sibling `nftw(3)`, "file tree walk," which take a callback you invoke per entry. They exist on Linux and are fine, but the callback model is more awkward than `fts`'s pull-style loop: passing state into an `nftw` callback means globals or thread-local storage, whereas `fts_read` just returns the next entry to your own loop where your state already is. `fts` originated in BSD and is available on Linux via glibc too, but it's a native, first-class part of the FreeBSD base, and the code you'll read in `rm`, `cp`, and `ls` uses it, so it's the idiom on this system. Compared to raw `readdir`, both `fts` and `nftw` are the same category of improvement: stop hand-rolling the recursion and the edge cases, use the library that already handles them.

## Why "in base" matters here

`fts` being in libc means the correct tree walk is always present, at a known version, and it's the *same* traversal the base tools are built on. That last part matters more than it sounds: when your `chmod -R`-like tool uses `fts` with `FTS_PHYSICAL`, it handles symlinks the way the real `chmod -R` does, because it's the same library making the decision, not your approximation of it. Consistency with the base tools' behavior comes free, because you're using the base tools' machinery. A directory walk is one of those things that looks trivial, invites a hand-rolled version, and hides a half-dozen edge cases that only surface on someone else's filesystem. The base system solved it once, correctly, and every recursive tool you rely on is proof it works. Reaching for `readdir` to walk a tree is, on FreeBSD, reaching past the solution the whole system already trusts.

---

*Man pages: `fts(3)` for the full API (`fts_open`/`fts_read`/`fts_children`/`fts_set`/`fts_close`, every `FTS_*` option and `fts_info` value, and the `FTSENT` layout). Compare `ftw(3)`/`nftw(3)` for the POSIX callback-style alternative. To see it in real use, the base sources for `bin/rm`, `bin/cp`, `bin/chmod`, and `bin/ls` are all short and all built on `fts`.*

*A note on the code: the API and `FTS_*` values here are from `fts.h` in FreeBSD 15.1. The demo omits `<sys/stat.h>` and error-path polish for brevity; a real build wants the former. `fts` changes directory during traversal by default (that's how it stays efficient on deep trees); if that matters to your program, `FTS_NOCHDIR` disables it, at a performance cost the man page describes.*
