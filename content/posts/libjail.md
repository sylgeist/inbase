+++
title = "Jails Are a Programmable Primitive, Not Just a Sysadmin Tool (libjail, and Lua in Base)"
date = 2026-06-06
draft = false
summary = "Everyone knows jail(8) the command. Fewer know jails are a kernel primitive with a C library your application can drive directly, and that FreeBSD's base Lua interpreter ships jail bindings. You can create and manage isolated environments from inside your program, no shelling out."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Jails get introduced to newcomers as "FreeBSD's containers," which is true enough to be useful and wrong enough to hide the interesting part. The framing makes you picture a sysadmin tool: you write a config, you run `jail -c`, you get an isolated environment, same as `docker run`. That's a real way to use jails, but it buries what they actually are underneath. A jail is a *kernel primitive*, a partitioning of the system that the kernel enforces, and it's exposed as a programmable interface. Your application can create, configure, enter, and tear down jails directly, in a few library calls, without ever shelling out to the `jail` command. And in a detail almost nobody knows, FreeBSD's base system ships Lua bindings for it, so you can do all of that from a script with no packages installed.

This post is about jails as something you *program against*, not something you configure.

## The C library, not the command

The `jail(8)` command is a thin userland program over a library, `libjail`, and a pair of system calls. The library is in base, and for the common cases its interface is almost aggressively simple. Creating a jail is one variadic call taking name/value pairs:

```c
#include <sys/param.h>
#include <sys/jail.h>
#include <jail.h>

/* Create a jail: a name, a root directory, an IP, and make it persist. */
int jid = jail_setv(JAIL_CREATE | JAIL_ATTACH,
    "name", "sandbox",
    "path", "/var/sandbox",
    "host.hostname", "sandbox.local",
    "ip4.addr", "10.0.0.99",
    "persist", NULL,
    NULL);
if (jid < 0)
    errx(1, "jail_setv: %s", jail_errmsg);
```

`jail_setv()` takes a null-terminated list of name and value strings, which is exactly the same set of parameters you'd put in a `jail.conf` stanza, passed as arguments instead. `JAIL_CREATE` makes a new jail; `JAIL_ATTACH` drops the calling process into it in the same step. After this call, this process (and its children) are inside the jail: a restricted view of the filesystem rooted at the path, its own hostname, its own network address, no visibility into other jails or the host's process table. The kernel enforces all of it.

The reverse call reads parameters back:

```c
char hostname[MAXHOSTNAMELEN];
jail_getv(0, "name", "sandbox", "host.hostname", hostname, NULL);
```

And there are lookups for the common identity mapping:

```c
int jid = jail_getid("sandbox");      /* name -> jid */
char *name = jail_getname(jid);       /* jid -> name */
```

For structured work, the `jailparam` family (`jailparam_init`, `jailparam_import`, `jailparam_set`, `jailparam_get`, `jailparam_all`) gives you typed access to the full parameter set and the ability to enumerate every jail and every parameter programmatically, which is how a monitoring or orchestration tool would introspect the system. But the point stands at the simple layer: **creating an isolated environment is a function call**, not a subprocess and a config file.

## Why an application would want this

The shell-out approach (fork, exec `jail -c`, parse its output, handle its exit code) works, but it's the same kind of compromise as shelling out to `curl` instead of using a library: you're driving a human-oriented tool through its text interface when a real API exists. Driving `libjail` directly means no argument-quoting hazards, no output parsing, structured error reporting through `jail_errmsg` instead of scraped stderr, and the ability to create a jail and *become* it in one atomic step rather than coordinating across a process boundary.

Concretely, this is what lets an application isolate *itself* or its workers. A server that processes untrusted input can, at startup, create a jail with a minimal filesystem view and attach a worker process to it, so a compromised worker sees almost nothing of the host. This composes with the other isolation primitives in this series: a worker can enter a jail *and* drop into Capsicum capability mode, layering a restricted namespace (the jail) under removed ambient authority (Capsicum). The jail bounds *what exists* from the process's point of view; Capsicum bounds *what it can do* with what exists. Building that from inside your program, through `libjail` and `cap_enter()`, is a very different and much tighter thing than launching a container runtime.

Hierarchical jails (jails inside jails) make this even more of a building block: a program can subdivide its own jail further for sub-workers, creating a tree of progressively-narrower environments, all through the same library calls.

## The part nobody knows: it's in base Lua

FreeBSD ships a Lua interpreter in its base system, `flua` (at `/usr/libexec/flua`), used internally for parts of the boot and install tooling. What almost nobody realizes is that `flua` comes with a `jail` module, also in base, so the entire programmable-jail interface is available from a scripting language with nothing installed:

```lua
#!/usr/libexec/flua

local jail = require("jail")

-- Enumerate every jail on the system and its parameters.
for params in jail.list() do
    print(params["jid"], params["name"])
end

-- Look up a jail by name.
local jid = jail.getid("sandbox")

-- Read parameters for a jail.
local params = jail.getparams("sandbox", { "host.hostname", "path" })
print(params["host.hostname"], params["path"])

-- Create or modify parameters.
jail.setparams("sandbox", { ["host.hostname"] = "renamed.local" },
    jail.CREATE)

-- Attach into a jail, or remove it.
jail.attach("sandbox")
jail.remove("sandbox")
```

The module surface is small and direct: `jail.getid` and `jail.getname` for identity, `jail.list` to iterate all jails, `jail.getparams`/`jail.setparams`/`jail.allparams` for reading and writing parameters, and `jail.attach`/`jail.remove` for lifecycle. It's a faithful binding of the C library, and it means a jail-management script is a base-system artifact: no `pkg install`, no language runtime to provision, no bindings to build. The interpreter, the bindings, and the jail primitive all ship together, at the same version, documented (the binding has its own `jail(3lua)` man page, and there's a worked example at `/usr/share/examples/flua/libjail.lua`).

That's a genuinely unusual capability to find sitting in a base install: a systems primitive exposed through a scripting language that's *also* in base, so a small orchestration script has zero external dependencies.

## The other-Unix contrast

On Linux, the equivalent primitives (namespaces and cgroups) are also programmable, through `clone(2)`/`unshare(2)` and the cgroup filesystem, and they're genuinely powerful. But there are two differences worth naming. First, Linux containers are an *assembly* of separate primitives (mount, PID, network, user namespaces, plus cgroups for resources), which you compose yourself or lean on a runtime (runc, systemd-nspawn) to compose for you; a jail is a single primitive that bundles the partitioning into one kernel object with one creation call. Second, the "drive it from a base scripting language with no dependencies" property doesn't have a clean Linux analogue: you'd reach for a language runtime you installed and a bindings library from a package. The FreeBSD version's distinctive trait isn't that it's more capable (it isn't, for every use case), it's that the whole stack from kernel primitive to scripting binding is one integrated, in-base thing.

## Why "in base" matters here

The reason you can create a jail from C in one call, or from Lua in a base-shipped interpreter, is that the primitive and its interfaces are all part of the base system. The `jail(2)` system call, `libjail`, the `flua` interpreter, and the `jail` Lua module version and ship together, so "isolate this workload" doesn't imply "install and configure a container platform." For an application that wants to sandbox its own components, that lowers the cost of isolation from an infrastructure decision to a library call, which is exactly the kind of thing that changes whether isolation gets used at all. A primitive is only a building block if it's cheap and reliable to reach for; being in base, callable from C and from a base scripting language, with the kernel doing the enforcement, is what makes jails a primitive you build *with* rather than a platform you deploy *onto*.

---

*Man pages: `jail(3)` for the `libjail` C API (`jail_setv`/`jail_getv`, the `jailparam_*` family, `jail_getid`/`jail_getname`, and `jail_errmsg`), `jail(2)` and `jail_set(2)`/`jail_get(2)` for the underlying system calls, `jail(8)` for the command and the full parameter list, and `jail(3lua)` for the base Lua bindings. The worked Lua example ships at `/usr/share/examples/flua/libjail.lua`.*

*A note on the code: the C signatures and Lua module surface here are from `jail.h`, `jail(3)`, and `libexec/flua/libjail` in FreeBSD 15.1. Creating a jail requires privilege, and a usable jail needs a populated root filesystem at its `path` (the examples assume one exists); see `jail(8)` for the parameter set and the filesystem setup a real jail wants. The full jail parameter namespace is large, so consult `jail(8)` for what each `name`/`value` pair does before relying on it.*
