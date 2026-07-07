# `cowclone`

`github.com/go-fsctl/cowclone`

Pure-Go **copy-on-write file cloning**: share a file's blocks with a new
copy in O(metadata), falling back to a full byte copy when the host
filesystem can't — with no cgo and no shelling out to `cp --reflink` or
`clonefile`.

Unlike the rest of the family, which each drive one Linux subsystem's
ioctls, `cowclone` is cross-platform: it speaks the two portable reflink
primitives directly and degrades gracefully everywhere else.

- **darwin** — APFS `clonefile(2)` (`golang.org/x/sys/unix.Clonefile`).
- **linux** — the `FICLONE` ioctl (`unix.IoctlFileClone`): reflink on
  btrfs, XFS (`reflink=1`), and OpenZFS ≥ 2.2 block cloning.
- **everywhere else**, across filesystems, or on a non-CoW filesystem — a
  streaming byte copy with identical observable semantics.

On a copy-on-write filesystem the clone is near-instant and shares
storage with the source until one side is written; the fallback produces
the same result — a full, independent `dst` — at the cost of copying
every byte. Either way the caller gets an independent `dst`.

## Install

```sh
go get github.com/go-fsctl/cowclone
```

## Clone a file

```go
import "github.com/go-fsctl/cowclone"

// Make dst a copy-on-write clone of src (replacing dst if it exists).
// Tries a real reflink first; transparently falls back to a byte copy when
// the filesystem can't share blocks or src/dst are on different filesystems.
err := cowclone.Clone("base.img", "instance.img")
```

`Clone(src, dst)` is the whole surface:

1. It removes any pre-existing `dst` (a CoW clone fails if the target
   already exists), erroring only if that removal genuinely fails.
2. It attempts the platform reflink primitive. On success `dst` is a
   block-sharing clone and `Clone` returns `nil`.
3. If the primitive reports the filesystem can't share blocks
   (`ENOTSUP`/`EOPNOTSUPP`), or that `src` and `dst` live on different
   filesystems (`EXDEV`), or that the ioctl is unimplemented (`ENOSYS`),
   it falls back to a streaming byte copy.
4. Any other error (`ENOENT`, `EACCES`, `ENOSPC`, …) is surfaced,
   wrapped, and *not* silently retried as a copy.

The result is observable-equivalent across all paths: after `Clone`,
writing to one file never affects the other.

## Build tags

| File | Builds on | Clone primitive |
| --- | --- | --- |
| `clone_darwin.go` | `darwin` | APFS `clonefile(2)` |
| `clone_linux.go` | `linux` | `FICLONE` ioctl (reflink) |
| `clone_other.go` | everything else | always byte-copy fallback |

## Testing without a reflink-capable mount

Following the `go-fsctl` house style, the OS primitives are reached
through indirection **seams** (`seams_linux.go`, `seams_darwin.go`) — a
`var` over `unix.Clonefile` / `unix.IoctlFileClone` and the file opens —
so tests fault-inject each success and errno branch deterministically.
On a plain ext4 or tmpfs CI filesystem `FICLONE` only ever returns
`EOPNOTSUPP`, so neither the reflink success path nor the other errnos
would otherwise be reachable without root or a dedicated btrfs/xfs mount.
Every branch is covered on every platform, and the suite needs neither
root nor a special filesystem — so it runs identically on the native
(linux + macOS) and QEMU-emulated CI lanes.

## Platforms

Built and tested on the six 64-bit Go architectures — `amd64`, `arm64`,
`riscv64`, `loong64`, `ppc64le`, `s390x` (big-endian) — plus native
macOS (`darwin/arm64`) for the real APFS `clonefile` path, and a
cross-build check on `darwin/amd64`, `windows` and `freebsd` (the
fallback stub). 100% statement coverage throughout.
