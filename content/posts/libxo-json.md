+++
title = "Every FreeBSD Base Tool Already Speaks JSON (libxo)"
date = 2026-06-26
draft = false
summary = "Pipe ls straight into jq on a stock FreeBSD box, with no install. Around a hundred base tools emit JSON, XML, or HTML from the same code path that prints their text. Structured output as a property of the system, not a per-tool afterthought."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Try this on a stock FreeBSD box. Nothing installed, no packages, just base:

```sh
ls --libxo json | jq
```

You get your directory listing as structured JSON, ready to pipe into `jq`. Now do it to things you'd never expect to parse without regex:

```sh
netstat --libxo json
vmstat --libxo json
df --libxo json
```

These aren't bolted-on `--json` flags that some maintainer added to one tool. They're a single facility, **libxo**, wired through roughly a hundred base utilities, so the same program that prints human-readable text can emit JSON, XML, or HTML from the *exact same* internal calls. Structured output isn't a feature of `netstat`; it's a property of the base system that `netstat` participates in.

If you've ever written `awk '{print $4}'` against the output of a system tool and watched it break when the format shifted by a column, this is the post for you.

## The problem libxo kills

The Unix tradition is text streams, which is glorious for interactive use and miserable for programmatic use. Every script that scrapes `df` or `netstat` or `ifconfig` is parsing output that was designed for a human's eyes: column positions, whitespace alignment, headers that may or may not be there, fields that change shape when a value gets long. The parse works until the day the format drifts by a space, and then it fails silently, usually in production, usually at 3am.

The usual responses are all unsatisfying. You pin tool versions and pray. You write increasingly baroque `awk`. On Linux you maybe reach for `jc`, a tool that *parses the text back* into JSON, which works but is a second guess at structure the original tool already knew and threw away. Every one of these is reconstructing, downstream, information that existed and was discarded at the source.

libxo's bet is the opposite: don't serialize structured data to text and re-parse it; keep it structured at the source and render text as just *one* of the output formats.

## How it works

A tool that uses libxo doesn't `printf`. It calls libxo's emit API, `xo_emit()` and friends, with a format string that describes the *data*, not just its visual layout. From that single set of calls, libxo can render:

- **text**: the normal human output, indistinguishable from the old `printf` version
- **JSON**: for `jq` and scripts
- **XML**: for tooling that wants it
- **HTML**: folding-and-styling-ready markup, with classes

One code path, four renderings, chosen at runtime by a flag. The tool author writes the emit calls once; the formats come for free. That's why it scales to a hundred tools: each tool isn't implementing JSON output, it's just *using libxo*, and JSON falls out.

The flag conventions you'll actually type:

```sh
# the long form, works broadly
ls --libxo json

# many tools also take a short style
ls --libxo=json,pretty

# some tools wired early have their own shorthand too, e.g.
top -j
```

The `pretty` modifier indents; `--libxo xml` or `html` swap the format. There's a warning modifier and others documented in `xo(1)`. The flag is a small language, not a boolean.

## Discovering the roster

The fun question: *which* tools speak it? The answer on a 15.1 system is "a lot of the base ones": `ls`, `netstat`, `df`, `vmstat`, `iostat`, `ps`, `top`, `wc`, `ifconfig` (via its own path), `systat`, and many more. Rather than memorize a list, the honest move is to probe: tools that link `libxo` accept the `--libxo` option, and you can grep the base for them or just try the flag. (`xo(1)` and the libxo documentation enumerate the wired-up utilities; the set grows release over release.)

The payoff of the roster being large is composition. Once *structured* is the default contract rather than a special case, your monitoring scripts stop being fragile text-scrapers and start being data pipelines:

```sh
# free memory, as a number, no column-counting
vmstat --libxo json | jq '.["vmstat"]...'

# established TCP connections, structured
netstat --libxo json | jq '...'
```

(The exact `jq` paths depend on each tool's schema, which is itself documented, because the structure is deliberate rather than incidental.)

## Use it in your own programs

libxo isn't just for base tools; it's a library in base (`-lxo`), so your own C can emit human text and JSON from one set of calls, same as `netstat` does. The shape:

```c
#include <libxo/xo.h>

int
main(int argc, char **argv)
{
    /* parse --libxo and friends out of argv */
    argc = xo_parse_args(argc, argv);
    if (argc < 0)
        return (1);

    xo_open_container("flock");
    xo_emit("The {:count/%d} {:animal} flew over {:place}\n",
        12, "geese", "the barn");
    xo_close_container("flock");

    xo_finish();
    return (0);
}
```

Build it:

```sh
cc -o flock flock.c -lxo
./flock                      # The 12 geese flew over the barn
./flock --libxo json         # {"flock": {"count": 12, "animal": "geese", ...}}
./flock --libxo xml          # <flock><count>12</count>...</flock>
```

The format string is the interesting part. Those `{:name}` fields aren't `printf` positions, they're *named, typed* data fields. libxo knows "count" is a number and "animal" is a string, so it can render them as text for humans or as typed JSON/XML for machines, from the one description. You write the data's shape once; the renderings are mechanical.

## The other-Unix contrast

Linux's structured-output story is a patchwork. Some tools grew their own `--json` (`ip -j`, `ss -j`, a few systemd utilities), inconsistently, with different flag spellings, and most base tools never got it at all. Where the tool offers nothing, you reach for `jc`, which wraps a command and parses its *text* into JSON after the fact, clever and genuinely useful but fundamentally a downstream reconstruction: it re-derives structure the tool computed and then flattened to text. libxo never flattens-then-reparses. The structure is retained at the source and text is just one rendering of it. It's the difference between "this tool happens to support JSON" and "this system treats structured output as a first-class rendering of every tool's data."

The contrast isn't that Linux can't produce JSON; it's that on Linux it's a per-tool accident, and on FreeBSD it's a base-system property with one library, one flag convention, and one place (`xo(1)`) to read about it.

## Why "in base" matters here

The reason `ls --libxo json | jq` works on *every* FreeBSD box, with no install step, is that libxo ships in base and the base tools are compiled against it. That's the whole pitch in one command: a capability you can rely on being present, at a known version, the same on every install, because it's part of the audited system rather than a thing each tool or distro bolted on independently. Structured output stops being something you hope a tool supports and becomes something you assume, because it's a property of the base, not a feature of the tool.

---

*Man pages: `xo(1)` for the command-line flag language, `libxo(3)` for the library overview, `xo_emit(3)` for the emit API and format-string syntax, `xo_parse_args(3)` for wiring it into your own programs.*

*A note on the code and commands: the `--libxo` flag spelling and the `xo_emit` format-string syntax follow the documented libxo interface, but tool-by-tool the exact shorthand varies (some take `--libxo=json`, a few have their own `-j`), so confirm a given tool's invocation against its own man page and `xo(1)` on your 15.1 box. The `flock` demo compiles against `-lxo` in base.*
