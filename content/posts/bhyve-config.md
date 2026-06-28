+++
title = "Your bhyve Command Line Is a Config File in Disguise (bhyve -k)"
date = 2026-06-18
draft = false
summary = "bhyve takes a config file (a hierarchical tree of dotted keys with variable expansion) instead of a wall of -s slot flags. And it'll translate your existing command line into that config for you. All in base, no vm-bhyve required."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

If you've run a bhyve VM by hand, you've written something like this, a single line that grows a new `-s` every time you add a device, until it's an unreadable wall you copy-paste between shell scripts and never quite trust:

```sh
bhyve -c 4 -m 8G -H -w \
  -s 0,hostbridge \
  -s 3,virtio-blk,/dev/zvol/tank/vm/web1 \
  -s 4,virtio-net,tap0 \
  -s 31,lpc \
  -l com1,stdio \
  -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd \
  web1
```

Most people reach for `vm-bhyve` or `cbsd` to escape this, and those are good tools. But they're *ports*. What far fewer people know is that bhyve itself, in base, already takes a **config file**: a clean hierarchical key/value description of the VM. You don't need a management framework to stop writing slot-flag walls. You need `-k`.

And the best part, the part that makes this a five-minute adoption instead of a rewrite, is that bhyve will hand you the config file for your existing command line. It can translate itself.

## The config tree

bhyve's configuration is, in its own words, a hierarchical tree of variables: bhyve uses a hierarchical tree of configuration variables to describe global and per-device settings. Internal nodes in this tree do not have a value, only leaf nodes have values. You write it as dotted keys. The global knobs are flat:

```ini
name=web1
cpus=4
memory.size=8G
```

Those map exactly to flags you already know: `memory.size` is `-m`, `cpus` is `-c`. Memory takes the same human sizes you're used to, parsed per `expand_number(3)`, so `8G` is `8G`.

Devices live under a node named for their bus address. A PCI device is `pci.<bus>.<slot>.<function>`, and every PCI device node must declare what it is:

```ini
pci.0.0.0.device=hostbridge

pci.0.3.0.device=virtio-blk
pci.0.3.0.path=/dev/zvol/tank/vm/web1

pci.0.4.0.device=virtio-net
pci.0.4.0.backend=tap0

pci.0.31.0.device=lpc
```

Read that against the command line above and the translation is one-to-one: `-s 3,virtio-blk,/dev/zvol/...` becomes a `pci.0.3.0` node with `device` and `path` leaves. The flat positional CSV of the `-s` flag, where you had to *remember* that the third comma-field of a virtio-blk is its backing path, becomes named keys. `path` is `path`. You're not counting commas anymore.

The serial console and boot ROM, which were the awkward `-l` flags, get their own nodes. The LPC bridge keeps its serial-port config under a top-level `lpc` node (not under the PCI device), which the man page calls out explicitly:

```ini
lpc.com1.path=stdio
lpc.bootrom=/usr/local/share/uefi-firmware/BHYVE_UEFI.fd
```

Run it with:

```sh
bhyve -k web1.conf
```

That's the whole VM, in a file you can read, diff, and version-control.

## The feature that sells it: `config.dump`

Here's the trick that makes adopting this painless. There's a global boolean, `config.dump`, and the man page describes it precisely: if this value is set to true after bhyve has finished parsing command line options, then bhyve will write all of its configuration variables to stdout and exit. No VM will be started.

bhyve builds the *same* internal config tree whether you fed it `-s` flags or a `-k` file; the flags are just a front-end that sets tree variables. So you can take your existing, battle-tested command line, append one setting, and bhyve prints the file you would have written by hand:

```sh
bhyve -c 4 -m 8G -H -w \
  -s 0,hostbridge \
  -s 3,virtio-blk,/dev/zvol/tank/vm/web1 \
  -s 4,virtio-net,tap0 \
  -s 31,lpc \
  -l com1,stdio \
  -o config.dump=true \
  web1
```

(`-o key=value` sets a single config variable inline, here `config.dump`.) Instead of booting, bhyve dumps its fully-resolved config tree to stdout. Redirect it to a file and you have a working `-k` config that exactly reproduces the command line you trust, with no manual translation and no guessing which comma meant what:

```sh
bhyve ...your exact flags... -o config.dump=true web1 > web1.conf
# review it, then from now on:
bhyve -k web1.conf
```

This is the migration path. You don't rewrite anything; you ask bhyve to rewrite it for you, then read the result to learn the key names. The config file *is* your command line, printed in the format bhyve was using internally all along.

## Variable expansion: one config, many VMs

The config format has a small feature that turns a single file into a template. Values can reference other values: instances of the pattern %(var) are replaced by the value of the configuration variable var. The man page's own example is exactly the homelab pattern you'd want, a disk path derived from the VM name:

```ini
name=web1
memory.size=8G
cpus=4

pci.0.3.0.device=virtio-blk
pci.0.3.0.path=/dev/zvol/tank/vm/%(name)
```

With `path` set to `/dev/zvol/tank/vm/%(name)`, the backing store resolves to the zvol matching the VM's name. Change `name=web2` (or override it with `-o name=web2`) and the same file points at `web2`'s disk. One template, parameterized by the one value that differs. A literal `%` is escaped by doubling it, so the expansion is unambiguous.

This is the seam where the config file stops being "my command line, but tidier" and starts being a small abstraction, without dragging in a whole management framework to get it.

## What's actually in the tree

Skimming the config namespace is its own "didn't know that" moment, because it exposes knobs the flag interface buries. A few from the `bhyve_config(5)` tables worth knowing exist:

- **`destroy_on_poweroff`**: destroy the VM on guest-initiated power-off, so a `poweroff` inside the guest cleans up the host-side VM instead of leaving it lingering.
- **`rtc.use_localtime`**: RTC tracks host local time (default true); set false for UTC. The kind of thing you'd otherwise hunt for.
- **`gdb.port` / `gdb.wait`**: bhyve has a built-in GDB stub; set a port and it listens, optionally pausing the guest until a debugger attaches. A kernel debugger for your VM, configured by two keys.
- A whole **SMBIOS surface** (`system.manufacturer`, `board.serial_number`, `chassis.asset_tag`, and friends) so the guest's reported hardware identity is fully configurable, not stuck at bhyve's defaults.
- Per-device serial numbers, sector sizes (`sectorsize=logical/physical`), `nocache` (open the backing store `O_DIRECT`), `ro` for read-only disks: block-device settings that were painful or impossible to express in the CSV flag.

The point isn't any one knob; it's that a *named tree* makes the full surface discoverable. `config.dump` plus the man page's tables means you can see every variable the VM resolved to, instead of reverse-engineering behavior from flag order.

## The other-Unix contrast

On the Linux side, the closest equivalents are libvirt's domain XML or a QEMU command line, and the comparison is instructive in both directions. QEMU's raw command line has the same wall-of-flags problem bhyve's `-s` interface does, arguably worse. libvirt solves it with XML, but libvirt is a large external daemon-plus-toolkit you install and run; the XML is its format, not the hypervisor's. bhyve's config file is notable for being the *hypervisor's own* native format, in base, with no daemon, and for the `config.dump` self-translation, which neither QEMU nor libvirt offers as cleanly (you can't trivially ask `qemu` to print its args as a libvirt domain). The FreeBSD version is smaller in scope but tighter: the thing that runs the VM is the thing that reads the config and the thing that can emit the config.

To be clear about base-versus-ports: `vm-bhyve` and `cbsd` are ports, and they're genuinely useful: they manage networking, datasets, startup ordering, a whole lifecycle the bare config file doesn't. This post isn't "don't use them." It's that the *config file itself*, the thing those tools ultimately drive bhyve with, is a base feature you can use directly, understand, and fall back to when you want to know what the framework is actually doing.

## Why "in base" matters here

The reason this matters beyond tidiness: when the config format ships with the hypervisor, in base, documented in `bhyve_config(5)`, it's the ground truth that every higher-level tool sits on top of. `vm-bhyve` generates this; `cbsd` generates this; your hand-written file is this. Understanding the base config tree means you can read what any of those tools produce, debug a VM that won't boot by dumping its resolved config, and run a VM with nothing installed but the base system. The management frameworks are conveniences over a documented base format, not a substitute for one, the way an opaque tool-specific config would be. The hypervisor, its config language, and the docs for that language all shipped together, at 15.1, as one thing.

---

*Man pages: `bhyve_config(5)` for the full variable tree (every key referenced here lives in its tables), `bhyve(8)` for the runtime flags including `-k`, `-o`, and `-C`. See also `expand_number(3)` for the size syntax and `tap(4)`/`netgraph(4)`/`netmap(4)` for the network backends.*

*A note before you rely on this: the config keys here are taken directly from `bhyve_config(5)` as shipped in 15.1, but bhyve's UEFI boot ROM (`BHYVE_UEFI.fd`) comes from the `sysutils/bhyve-firmware` port, not base, so a UEFI guest needs that one package even though the config mechanism itself is base. Run `bhyve ... -o config.dump=true` against your own working invocation to get the authoritative key names for your exact device set before committing a hand-written file.*
