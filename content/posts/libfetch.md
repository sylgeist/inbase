+++
title = "The HTTP Client pkg Uses Is in Base, and You Can Call It in Three Lines (libfetch)"
date = 2026-06-12
draft = false
summary = "Fetching a URL from C usually means taking a dependency on libcurl. FreeBSD ships libfetch in base: HTTP, HTTPS, FTP, and file URLs behind one small, regular API that returns a plain FILE *. It's what pkg itself downloads with."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

You want to download a URL from a C program. The reflex, correctly, is "link libcurl." curl is superb and everywhere, and on most systems it's the only sane answer. But on FreeBSD there's a second answer that needs no dependency at all, because the HTTP client is already in the base system, and it's not a toy: it's the same library `pkg` uses to download every package you install.

Here's the whole thing:

```c
#include <fetch.h>
#include <stdio.h>

int
main(void)
{
	FILE *f = fetchGetURL("https://www.freebsd.org/", "");
	if (f == NULL)
		errx(1, "fetch failed: %s", fetchLastErrString);

	char buf[4096];
	size_t n;
	while ((n = fread(buf, 1, sizeof(buf), f)) > 0)
		fwrite(buf, 1, n, stdout);

	fclose(f);
	return (0);
}
```

```sh
cc -o get get.c -lfetch
./get | head
```

`fetchGetURL()` parses the URL, opens the connection, does the TLS handshake for `https`, follows the protocol, and hands you back a `FILE *`. Not a custom handle, not a callback soup: a normal stdio stream you `fread`, `fgets`, or `fclose` like any other. That return type is the first half of why this library is pleasant, and the API's shape is the second half. Let me take them in turn, because the design is the actual story here.

## A `FILE *` is the whole integration

Most HTTP libraries hand you a bespoke object with its own read function, its own lifecycle, its own idea of buffering, and a callback interface you register into. libfetch hands you a `FILE *`. Everything in the C standard library that already consumes a stream just works on it:

```c
FILE *f = fetchGetURL("https://example.com/data.txt", "");

char line[256];
while (fgets(line, sizeof(line), f) != NULL)   /* line-oriented reads */
	process(line);
```

The remote resource is just a stream, indistinguishable at the call site from a local file you `fopen`ed. That means a function you already wrote to parse a local file parses a URL with no changes; you hand it the `FILE *` and it neither knows nor cares that the bytes are arriving over TLS from another continent. There's no impedance layer to write, because the impedance layer is the one every C programmer already knows.

## The API is a grid, and that regularity is the point

Here's the part that rewards a second look. libfetch's function set isn't a pile of special cases; it's a small, completely regular matrix. There are a handful of *operations*:

- **`Get`** open a resource for reading
- **`Put`** open it for writing (upload)
- **`Stat`** get size and modification time without downloading
- **`List`** enumerate a directory or index
- **`XGet`** a `Get` that also fills in a stat struct in one round trip

And there are the *schemes* those operations apply to: `File`, `HTTP`, `FTP`. Every operation exists for every scheme, named by concatenation, and then there's a scheme-independent top layer that dispatches on the URL string itself:

```c
/* scheme-specific, if you already parsed the URL into a struct url: */
fetchGetHTTP(u, "");     fetchStatHTTP(u, &st, "");     fetchListHTTP(u, "");
fetchGetFTP(u, "");      fetchStatFTP(u, &st, "");      fetchListFTP(u, "");
fetchGetFile(u, "");     fetchStatFile(u, &st, "");     fetchListFile(u, "");

/* scheme-independent, dispatches on the URL's scheme for you: */
fetchGetURL("https://host/doc", "");
fetchGetURL("ftp://host/doc", "");
fetchGetURL("file:///etc/motd", "");
```

Look at what that buys you. `fetchGetURL()` picks the right protocol implementation from the URL, so the *same call* fetches an HTTP resource, an FTP resource, or a local file, and in every case you get a `FILE *` back. Switching a program from reading a local config file to reading it from an internal HTTP server is a URL-string change, not a rewrite. The manual describes each scheme's handler as working "in a manner consistent with the rest of the" library, and that consistency is deliberate: the operation you want (`Get`, `Stat`, `List`) means the same thing and returns the same type regardless of whether the bytes come from `http`, `ftp`, or the local disk.

This is what a *standardized interface* actually earns you, and it's worth being concrete about, because "standardized" gets said a lot and cashed out rarely:

**One mental model covers every source.** You learn `Get`/`Put`/`Stat`/`List` once. You do not learn an HTTP API and then a separate FTP API and then remember that local files are yet another thing. The verbs are the same; only the URL changes. A `Stat`-then-decide-then-`Get` pattern you write for HTTP is line-for-line identical for FTP.

**Uniform error handling.** Every call reports failure the same way: `NULL` (or a nonzero return for `Stat`), with a code in `fetchLastErrCode` and a human string in `fetchLastErrString`. You write one error path, not one per protocol.

**Uniform configuration.** Proxies come from the environment (`HTTP_PROXY`, `FTP_PROXY`, and their lowercase forms) for every scheme at once. Timeouts are one global, `fetchTimeout`. You configure the library, not each protocol.

**Substitutable sources.** Because `file://`, `http://`, and `ftp://` are behind one interface returning one type, the *source becomes configuration*. A tool can accept "a URL" for its input and let the operator decide at deploy time whether that's a local path, an internal mirror, or a public endpoint, with zero code branches. That substitutability is a direct consequence of the interface being standardized across schemes, and it's the kind of flexibility you normally have to build a small abstraction layer to get. Here the library *is* the abstraction layer.

The `Stat` operation is a good example of the grid paying off. Say you want to skip a download if you already have the current version:

```c
struct url_stat st;
if (fetchStatURL("https://mirror/file.iso", &st, "") == 0) {
	/* st.size and st.mtime, without downloading the body */
	if (st.mtime <= local_mtime)
		return;   /* already current */
}
FILE *f = fetchXGetURL("https://mirror/file.iso", &st, "");  /* get + stat */
```

`fetchStatURL` does a `HEAD`-equivalent for HTTP, a `SIZE`/`MDTM` for FTP, and a `stat(2)` for a local file, and you get the same `struct url_stat` (`size`, `atime`, `mtime`) out of all three. The conditional-download logic you write is protocol-blind.

## It's real infrastructure, not a demo

The reason to trust libfetch with actual work is that FreeBSD already does. The `pkg` bootstrap in base downloads the package manager over libfetch. The `fetch(1)` command-line tool is a thin wrapper over it. `bsdinstall`'s distribution-fetching stage uses it. This is load-bearing code on the path that installs and updates the system, which is about as battle-tested as a download library gets.

What it supports, from that base install with no packages: HTTP and HTTPS with TLS, FTP and FTPS, local `file://` URLs, HTTP redirects, transfer resumption via the offset field in `struct url` (which is how package downloads resume after an interruption), proxies from the environment, authentication (including `.netrc`), and If-Modified-Since via the stat fields. `fetchParseURL()`/`fetchMakeURL()`/`fetchFreeURL()` give you URL parsing and construction as a bonus, which is itself a thing people pull in a library for.

## The honest contrast with libcurl

This is not a "libfetch beats curl" post, because it doesn't. curl is the right tool when you need HTTP/2 or HTTP/3, fine-grained control over headers and methods, cookie jars, a huge protocol range (SMTP, IMAP, SCP, and on and on), or portability across every OS on earth. libfetch is smaller in every dimension: fewer protocols, no HTTP/2, a modest knob set, FreeBSD-oriented. If you're writing portable software that must build everywhere, you link curl and move on.

What libfetch offers is a different trade: for the extremely common case of "GET this URL" or "download this file, resumable, maybe check the size first," it does the job with *zero dependencies* on FreeBSD, through an interface small enough to hold in your head, returning a type you already use. When your program is FreeBSD-targeted anyway (a port's helper, a jail-management tool, a system daemon), reaching past base to vendor curl for a single `GET` is weight you don't need to carry. The standardized `Get`/`Put`/`Stat`/`List`-over-any-scheme interface means you also get the local-file and FTP cases for free, which a curl integration would handle differently or not at all.

## Why "in base" matters here

A URL-fetching interface in the base system, at a known version, documented in `fetch(3)`, changes the default calculus of a whole category of program. "This tool needs to download something" stops implying "add a dependency." Because the same library ships on every FreeBSD install and the base system itself relies on it, you can write a system tool that fetches over HTTPS and know it will build and run on a stock box with nothing added. And because the interface is standardized across schemes, that tool can treat "where the data comes from" as a URL the operator supplies rather than a protocol the programmer hardcoded. The uniformity is only possible because one library, in base, owns all the schemes behind one set of verbs, instead of each protocol arriving as a separate dependency with its own API. That is the practical dividend of a standardized interface that ships with the system: not just that it's present, but that it makes every source look the same to the code that consumes it.

---

*Man pages: `fetch(3)` for the full API (the `Get`/`Put`/`Stat`/`List` matrix across `File`/`HTTP`/`FTP` schemes, the `struct url` and `struct url_stat` layouts, the `fetchLastErrCode`/`fetchLastErrString` error model, and the proxy/timeout globals), and `fetch(1)` for the command-line tool built on it. Link with `-lfetch`.*

*A note on the code: signatures here are from `fetch.h` and `fetch(3)` in FreeBSD 15.1. The examples elide `#include <err.h>` for `errx`; a real build wants it. TLS certificate verification behavior is controlled through the library's documented environment and flags, so check `fetch(3)` for the current verification defaults before shipping something security-sensitive.*
