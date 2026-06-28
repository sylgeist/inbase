+++
title = "There's a Typed Serialization Library in the Base System, and It Passes File Descriptors (libnv)"
date = 2026-06-24
draft = false
summary = "nvlist is a typed, nested, dependency-free serialization primitive in the FreeBSD base system (libnv) that can move open file descriptors between processes. That last property makes it quietly perfect for handing secrets around."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Here's the problem that sent me looking. I wanted one process to hand a secret (an API token, a private key, a database credential) to another process, without the secret touching disk in a spot the receiver could read on its own, without serializing it into a string that lands in a log or a core dump, and ideally without the receiver having any filesystem access to go fetch it itself. On most Unixes you start reaching for a library, or a local socket protocol you design by hand, or you shrug and write the token to a file with tight permissions and hope.

On FreeBSD the building block already ships in base, in a small library called `libnv` (link it with `-lnv`): the `nvlist`. It's a typed name/value container that serializes cleanly, nests arbitrarily, and, the part that matters here, can **send an open file descriptor across a socket to another process**. That last property turns it from "a nice serialization format" into something genuinely useful for moving secrets around safely. This post is about why.

## What an nvlist is

An `nvlist` (name/value list, from `libnv`, declared in `<sys/nv.h>`) is a container mapping string keys to typed values. Unlike a blob of JSON or a hand-packed struct, every value carries its type, and the types include things text formats can't represent:

- `bool`, `number` (uint64), `string`
- `nvlist`: a nested nvlist, so you get arbitrarily structured messages
- `binary`: arbitrary byte buffers, length-tagged
- `descriptor`: an **open file descriptor**
- array variants of the above

You build one, add typed fields, and either query it in place or serialize it. The API is regular to the point of being boring, which is the right kind of boring:

```c
#include <sys/nv.h>

nvlist_t *nvl = nvlist_create(0);

nvlist_add_string(nvl, "role", "garage-s3");
nvlist_add_number(nvl, "ttl_seconds", 3600);
nvlist_add_bool(nvl, "renewable", true);

/* read back, type-checked */
const char *role = nvlist_get_string(nvl, "role");
uint64_t ttl    = nvlist_get_number(nvl, "ttl_seconds");

nvlist_destroy(nvl);
```

There's a parallel `nvlist_exists_<type>(nvl, key)` family for presence/type checks, and an error model that's worth knowing about up front because it shapes how you write the code.

## The error model is "deferred," and that's a feature

`nvlist` doesn't make you check a return value after every `add`. Instead the nvlist carries an internal error state: once any operation fails (out of memory, a type misuse), the nvlist latches into an error and subsequent operations become no-ops. You build up the whole structure, then check once:

```c
nvlist_t *nvl = nvlist_create(0);
nvlist_add_string(nvl, "role", role);
nvlist_add_number(nvl, "ttl_seconds", ttl);
/* ... more adds ... */

int err = nvlist_error(nvl);
if (err != 0) {
    /* the whole thing is poisoned; bail */
    nvlist_destroy(nvl);
    errno = err;
    return (-1);
}
```

This reads much more cleanly than the error-after-every-call ritual, and it's safe because a poisoned nvlist won't silently half-serialize; the error rides along until you look. The one gotcha: don't read values out of an nvlist you haven't error-checked, because on a poisoned list the `get` accessors will abort on a missing key.

## Ownership: `add` copies, `move` transfers

The lifetime rule trips people up once and then never again. The `nvlist_add_*` functions **copy** their argument into the nvlist: after `nvlist_add_string`, you still own your string and must free it if it was heap-allocated; the nvlist holds its own copy. The `nvlist_move_*` family instead **transfers ownership** into the nvlist: after `nvlist_move_string`, the nvlist will free it, and you must not. `move` matters for two things: large buffers you don't want to copy, and descriptors.

Descriptors are the interesting case. `nvlist_add_descriptor` does work, but it `dup(2)`s your fd and keeps the copy, so you're left holding the original to close yourself. `nvlist_move_descriptor` instead consumes the fd you hand it: the nvlist owns it now, and on the receiving side `nvlist_take_descriptor` hands ownership back to you (you `close` it when done). For passing a secret as an open fd, `move` is usually what you want, so there's one owner of the descriptor at each step rather than a dup you have to remember to clean up.

For secrets specifically, `move` has a bonus: it shortens the window in which the sensitive bytes exist in two places. More on that below.

## The part that matters: descriptors cross the boundary

Here's where nvlist stops being "msgpack that ships with the OS" and becomes something you can't easily replicate with a serialization library. You can add an open file descriptor to an nvlist, serialize the nvlist, send it over a Unix-domain socket, and the receiving process gets a **working descriptor**: the kernel duplicates the open file into the receiver, the same machinery that backs `SCM_RIGHTS` fd-passing, but wrapped in a typed message you don't have to hand-assemble.

```c
/* sender side */
nvlist_t *nvl = nvlist_create(0);
nvlist_add_string(nvl, "kind", "x509-key");
nvlist_move_descriptor(nvl, "fd", key_fd);   /* transfers the fd in */

/* nvlist_send packs and sends over a connected socket,
   carrying the descriptor with it */
if (nvlist_send(sock, nvl) < 0)
    err(1, "nvlist_send");
nvlist_destroy(nvl);
```

```c
/* receiver side */
nvlist_t *nvl = nvlist_recv(sock, 0);
if (nvl == NULL)
    err(1, "nvlist_recv");

int key_fd = nvlist_take_descriptor(nvl, "fd");  /* now owned by us */
const char *kind = nvlist_get_string(nvl, "kind");
/* read the key material straight off key_fd; we never had a path to it */
nvlist_destroy(nvl);
```

(There's also `nvlist_pack`/`nvlist_unpack` if you want the serialized bytes in hand rather than sending them directly, but note that **packed bytes can't carry descriptors**, only the `send`/`recv` path can, because a raw byte buffer has nowhere to put a kernel reference. That asymmetry is the whole point: the fd travels through the socket's ancillary-data channel, not through the serialized payload.)

## Why this is good for secrets

Lay the descriptor-passing property against the threat model for handing a secret between processes and several nice things fall out.

**The secret can travel as a descriptor, not as bytes in a parsable message.** If the receiver needs to *use* a private key but never needs to see its raw bytes at the application layer, you can hand it an open fd to the key and let it read what it needs, rather than serializing the key into the message where it could be logged, copied, or end up in a core dump of the IPC layer. The sensitive material rides in the kernel's fd table, not in your wire format.

**The receiver doesn't need filesystem access to the secret.** This is the strong version, and it's where nvlist meets Capsicum (a future post, but the connection matters here). A process in capability mode can't `open()` a path; it has only the descriptors it was handed. A secrets-broker process that *does* have filesystem access can open the key file, then hand the open descriptor to a sandboxed worker over an nvlist. The worker reads the key through the fd it was given and has no way to reach any *other* secret, because it has no ambient filesystem authority at all. The nvlist is the courier; Capsicum is the reason the courier model is enforceable rather than merely polite.

**Typed fields mean no stringly-typed parsing of metadata.** The credential's role, TTL, renewability, and bucket scoping ride alongside as typed `number`/`bool`/`string` fields. You're not parsing a flat string and hoping the format holds; a missing or wrong-typed field is a checkable condition, not a substring bug.

**`move` semantics shrink the exposure window.** Using `nvlist_move_*` for the sensitive buffer (or descriptor) means the bytes aren't duplicated into the nvlist and then left lying in your original buffer; ownership transfers, and you zero-and-free fewer copies. It's not magic; you still have to be disciplined about wiping secret buffers, but fewer copies is fewer places to wipe.

**Zero dependencies, present everywhere, versioned with the OS.** This is the series refrain, but it earns its keep for security code specifically: a secrets-passing path you build on `nvlist` has no third-party library in its trust base. The serialization, the fd-passing, and the type system are all part of the same audited base system that ships, at the version the OS shipped. No supply-chain surface from a serialization dependency, because there's no dependency.

A sketch of the broker shape, to make it concrete:

```
[ broker process ]                         [ sandboxed worker ]
  has filesystem access                      cap_enter()'d, no fs access
  opens /secrets/garage.key  ──┐
  builds nvlist:               │
    role   = "garage-s3"       │  nvlist_send over
    ttl    = 3600              ├──  socketpair ──▶   nvlist_recv
    fd     = <open key fd>  ───┘                     take_descriptor("fd")
                                                     read key bytes via fd
                                                     use key; can reach
                                                     nothing else
```

The worker spawned for a given job receives exactly the one credential it needs, as a descriptor, inside a typed envelope, and, being in capability mode, physically cannot open any other secret. The blast radius of a compromised worker is one already-issued credential, not the whole secret store.

## The other-Unix contrast

The honest comparison: the *serialization* half of nvlist has plenty of equivalents: protobuf, msgpack, Cap'n Proto, flatbuffers, even JSON for the non-binary cases. What none of those do is **move a file descriptor**, because a descriptor is a kernel object, not a value, and a portable serialization format has nowhere to put it. To pass an fd on Linux you drop to raw `sendmsg` with `SCM_RIGHTS` and `cmsg` ancillary data, which works and is the same underlying mechanism, but you're hand-assembling control messages, and your *typed metadata* still needs a separate serialization layer bolted alongside. nvlist unifies the two: the typed message and the descriptor travel together through one API, and both halves ship in base. Linux has the pieces; FreeBSD has them assembled and documented as one thing.

The closest spiritual cousin is actually how nvlist is used inside base: it's the wire format under **Casper** (`libcasper`), the service that hands specific capabilities back to a sandboxed process. That's not a coincidence: a system built around taking away ambient authority needs a clean way to pass *specific* authority (including descriptors) back, and nvlist is that way. Which is the strongest possible endorsement of using it for your own secret-passing: the base system already trusts it for exactly this shape of problem.

## Why "in base" matters here

A secrets-passing path is security-critical code, and the appeal of building it on `nvlist` is that the entire trust base is the OS. The typed serialization, the fd-passing over sockets, the ownership model: all in base, all documented in `nv(9)`, all at the version FreeBSD 15.1 shipped. There's no `cargo add`, no pinning a serialization library's CVE feed, no "which version of the IPC dependency is in this container." When the courier for your credentials is part of the audited base system rather than a dependency you pulled in, the security review surface shrinks to code you were already trusting to boot the machine.

---

*Man pages: `nv(9)` for the full API and type system (it documents `nvlist_create`, the typed `add`/`move`/`get`/`take` families, and `nvlist_send`/`recv`), `cnv(9)` and `dnv(9)` for the cookie-based and default-value extensions, `cap_enter(2)` and `capsicum(4)` for the sandbox half of the secrets story, and `unix(4)` for the socket channel the descriptors travel through. The nvlist API is also the wire format under Casper (`libcasper`).*

*Provenance: `libnv` has been in base since FreeBSD 11.0, implemented by Pawel Jakub Dawidek under FreeBSD Foundation sponsorship. One caveat worth knowing if you wander into kernel code: the `nv` API exists in-kernel too, but descriptor values and the `nvlist_send`/`recv`/`xfer` calls are userland-only, since fd-passing has no meaning across the kernel boundary the same way. Everything in this post is userland. All signatures here are from `nv(9)` as shipped in 15.1.*
