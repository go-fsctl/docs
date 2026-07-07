# go-fsctl

**Pure-Go (CGO=0) Linux kernel-ioctl control libraries — no cgo, no
shelling out to `losetup` / `dmsetup` / `btrfs` / `zpool`.**

The `go-fsctl` org publishes small, focused libraries that drive Linux
storage subsystems the way the C user-space tools do: by opening the
subsystem's control device and issuing the native `ioctl`s directly. No
kernel modules to build, no cgo, no external binaries to fork — just the
Go standard library and `golang.org/x/sys/unix`.

## The family

| Module | Kernel surface | What it drives |
| --- | --- | --- |
| [`loop`](loop.md) | `/dev/loop-control` + `LOOP_*` | Attach / detach / inspect loop devices (what `losetup` does). |
| [`dm`](dm.md) | `/dev/mapper/control` + `DM_*` | Device-mapper: linear, striped, snapshot, crypt, thin, verity (what `dmsetup` does). |
| [`btrfs`](btrfs.md) | `BTRFS_IOC_*` on directory fds | Subvolumes, snapshots, scrub, balance, qgroups, defrag, send (what `btrfs(8)` does). |
| [`zfs`](zfs.md) | `/dev/zfs` + `ZFS_IOC_*` | OpenZFS pools, datasets, encryption, send/recv, clone/rollback/hold/bookmark/promote (what `libzfs_core` does). |
| [`cowclone`](cowclone.md) | APFS `clonefile(2)` + Linux `FICLONE` | Copy-on-write file cloning (reflink) on btrfs/XFS/OpenZFS and APFS, with a byte-copy fallback everywhere else (what `cp --reflink=auto` does). Cross-platform. |

## Why pure-Go

1. **Static linking, no cgo.** Drop the binary in a minimal initramfs,
   a `scratch`-style container, or a single-binary CLI. No `libzfs`, no
   `libdevmapper`, no `util-linux` runtime dependency.
2. **No external tools at runtime.** The libraries speak the kernel's
   `ioctl` / `nvlist` / `dm_ioctl` wire protocols themselves; they never
   fork `losetup`, `dmsetup`, `btrfs`, or `zpool`.
3. **No kernel-module fight.** OpenZFS normally needs the DKMS module
   rebuilt per kernel; `go-fsctl/zfs` talks to the already-loaded module
   through `/dev/zfs` without `libzfs`.
4. **ABI pinned and tested.** Every `ioctl` number and struct layout is
   derived from the kernel uapi headers and pinned by host-runnable unit
   tests (sizes, offsets, ioctl encodings), so the wire format is checked
   on every CI run.

## Install

```sh
go get github.com/go-fsctl/loop
go get github.com/go-fsctl/dm
go get github.com/go-fsctl/btrfs
go get github.com/go-fsctl/zfs
go get github.com/go-fsctl/cowclone
```

```go
import (
    "github.com/go-fsctl/loop"
    "github.com/go-fsctl/dm"
    "github.com/go-fsctl/btrfs"
    "github.com/go-fsctl/zfs"
    "github.com/go-fsctl/cowclone"
)
```

## Conventions across the family

- **Linux-only at runtime, buildable everywhere.** On non-Linux
  platforms every kernel operation returns `ErrUnsupported`, while the
  ABI definitions and the (de)serialization codecs stay available for
  tooling and tests. This is what lets the libraries build and unit-test
  on all six 64-bit architectures.
- **`Available()` probes.** Each package exposes a privilege-free
  `Available()` (and `btrfs.Available(path)`) that reports whether the
  control device / mount is present, so callers and integration tests can
  self-skip.
- **Privilege.** Mutating operations require `CAP_SYS_ADMIN` (in
  practice, root). Probing and decoding do not.
- **BSD-3-Clause, 100% test coverage, green CI on all six 64-bit Go
  targets** — `amd64`, `arm64`, `riscv64`, `loong64`, `ppc64le`, and
  `s390x`. Unit tests run on any host (`GOWORK=off go test ./...`);
  integration tests are gated on the real control device plus root and
  self-skip otherwise.

## Quick taste

```go
// Loop: attach a backing image to the first free /dev/loopN.
dev, err := loop.Attach("/var/tmp/disk.img", loop.Options{})

// Device-mapper: a one-shot linear mapping over that loop device.
err = dm.CreateWithTable("myvol", []dm.Target{
    dm.Linear(0, 131072, dev, 0),
})

// btrfs: snapshot a subvolume on a mounted btrfs filesystem.
err = btrfs.SnapshotCreate("/mnt/bt/sub1", "/mnt/bt", "snap1", true)

// ZFS: create a pool and a dataset with no libzfs and no zpool(8).
h, err := zfs.Open()
defer h.Close()
err = h.PoolCreate("tank", zfs.Vdev{Type: zfs.VDEV_TYPE_ROOT, Children: []zfs.Vdev{
    {Type: zfs.VDEV_TYPE_FILE, Path: "/var/tmp/disk0.img"},
}}, nil)
err = h.CreateFilesystem("tank/ds1")
```
