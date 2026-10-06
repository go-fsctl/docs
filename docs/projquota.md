# `projquota`

`github.com/go-fsctl/projquota`

Pure-Go Linux **project quotas** for XFS and ext4: give a directory tree a
project id, limit its space and inodes, and read its usage, straight through
the kernel — `FS_IOC_FSGETXATTR` / `FS_IOC_FSSETXATTR` and `quotactl_fd(2)` —
with no cgo and without shelling out to `xfs_quota`, `setquota` or `chattr`.

It is the project-quota member of the family; [`btrfs/`](btrfs.md)'s qgroups
are btrfs's own answer to the same question, and [`zfs/`](zfs.md)'s
`refquota` is ZFS's. `/etc/projects` and `/etc/projid` are **not** needed:
they are `xfs_quota`'s name tables, and the kernel never reads them.

!!! danger "On ext4, root is not held to the limit"
    The generic quota code ext4 uses lets any writer with `CAP_SYS_RESOURCE`
    past the hard limits (`fs/quota/dquot.c`, `ignore_hardlimit`). XFS has no
    such exemption. A file server that writes into an ext4 project directory
    as root — or with `CAP_SYS_RESOURCE` — is **not limited at all**. The
    first CI run found this by writing 16 MiB as root under an 8 MiB ext4
    limit; the integration tests now write as `nobody` and assert both
    behaviours.

## Requirements

- **Linux ≥ 5.14**, for `quotactl_fd(2)`. The older `quotactl(2)` needs the
  block device behind the mount; this package does not go looking for it.
- **Project quotas on the mount.** XFS: `mount -o prjquota`. ext4: a
  filesystem made with `mkfs.ext4 -O quota,project -I 256` (the project
  feature needs inodes larger than 128 bytes) and mounted with `-o prjquota`;
  the kernel needs the `quota_v2` format module (`CONFIG_QFMT_V2`), or the
  mount fails with `ESRCH` (on Ubuntu it ships in `linux-modules-extra`).
- **`CAP_SYS_ADMIN` for `SetLimits` and for `Usage`.** The kernel lets an
  unprivileged caller read only its own user and group quotas, never a
  project's (`fs/quota/quota.c`, `check_quotactl_permission`). Reading or
  setting a project id needs no privilege — see below.
- XFS project ids above 65535 need the `projid32bit` feature, the `mkfs.xfs`
  default for years.

Off Linux every function returns `ErrUnsupported`. There is no
`Available()`: `Detect` says whether a path is on XFS or ext4, and a
`QuotaError`'s hint says why a mount refused (below).

## Install

```sh
go get github.com/go-fsctl/projquota
```

## Which filesystem

```go
fs, err := projquota.Detect("/srv/shares")  // fstatfs f_type
// projquota.XFS or projquota.Ext4 ("xfs", "ext4"),
// else an error wrapping ErrUnsupportedFilesystem, with the magic
```

The ext4 magic is shared by ext2 and ext3 mounts, which the ext4 driver also
serves; only a filesystem made with the `project` feature accepts project ids.

## Project ids

```go
// Tag a fresh directory: project 4242, and FS_XFLAG_PROJINHERIT so that
// everything created inside it joins the project.      FS_IOC_FSSETXATTR
err = projquota.SetProject("/srv/shares/alice", 4242, true)

id, inherit, err := projquota.GetProject("/srv/shares/alice")  // no privilege

// Tag a directory that already has content.
err = projquota.SetProjectTree("/srv/shares/bob", 4243)
```

`SetProject` reads, modifies and writes the attributes, keeping every other
flag. Only the directory itself changes: files already inside keep their
project. `inherit` only makes sense on a directory; ext4 refuses it on a
regular file (`EOPNOTSUPP`).

`SetProjectTree` gives the root and everything below it on the same
filesystem the id — directories get `FS_XFLAG_PROJINHERIT` too, regular files
the id only. Symbolic links, devices, FIFOs and sockets are left alone, as
are other filesystems mounted inside. The walk opens every entry relative to
its parent's fd with `O_NOFOLLOW`, so a name swapped for a link during the
walk is skipped rather than followed out of the tree. A regular file with
other hard links outside the tree is charged to the project all the same: the
project id belongs to the inode, not the name.

Project id 4294967295 is the kernel's `INVALID_PROJID` and is refused with
`ErrInvalidProject`.

!!! danger "Do not give the tenant ownership of its directory"
    Changing a project id does **not** need privilege: `vfs_fileattr_set`
    (`fs/file_attr.c`) asks only that the caller own the file
    (`inode_owner_or_capable`), and refuses a project change only to callers
    *outside* the initial user namespace. So the owner of a project
    directory, as an ordinary user on the host, can move it — or any file
    they own inside it — to project 0, out of the limit. Keep the directory
    and its files owned by someone other than the tenant: serve it through a
    file server that writes on the tenant's behalf under its own uid, or
    confine the tenant to a user namespace, where the kernel answers
    `EINVAL`. The integration tests prove this on real XFS and ext4: an
    unprivileged owner clears its project id.

## Limits and usage

```go
// Bytes; 0 = no limit.        quotactl_fd Q_XSETQLIM (XFS) / Q_SETQUOTA (ext4)
err = projquota.SetLimits("/srv/shares", 4242, projquota.Limits{
    BlockSoft: 9 << 30,
    BlockHard: 10 << 30,
    InodeHard: 1_000_000,
})

// Usage and limits.           quotactl_fd Q_XGETQUOTA / Q_GETQUOTA
q, err := projquota.Usage("/srv/shares", 4242)
// q.Bytes, q.Inodes, q.BlockSoft, q.BlockHard, q.InodeSoft, q.InodeHard,
// q.BlockGrace, q.InodeGrace (zero when not over the soft limit)
```

The limits functions take any file or directory on the filesystem, usually
the mount point. Each path function has a `…File(*os.File, …)` twin:
`DetectFile`, `GetProjectFile`, `SetProjectFile`, `SetLimitsFile`,
`UsageFile` (not on an `O_PATH` file for the project-id calls).

| Operation | Kernel interface |
|---|---|
| `GetProject`, `SetProject`, `SetProjectTree` | `FS_IOC_FSGETXATTR`, `FS_IOC_FSSETXATTR` |
| `SetLimits` on XFS | `quotactl_fd(QCMD(Q_XSETQLIM, PRJQUOTA))`, `struct fs_disk_quota`, 512-byte basic blocks |
| `SetLimits` on ext4 | `quotactl_fd(QCMD(Q_SETQUOTA, PRJQUOTA))`, `struct if_dqblk`, 1 KiB block limits |
| `Usage` on XFS / ext4 | `Q_XGETQUOTA` / `Q_GETQUOTA` |
| `Detect` | `fstatfs` `f_type`: `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC` |

On ext4, `Q_GETQUOTA` needs write access to the mount and fails with `EROFS`
on a read-only one.

### Three things the kernel does that this package makes explicit

- **XFS silently ignores a hard limit below its soft limit**:
  `xfs_setqlim_limits` logs `hard < soft`, keeps the old limits, and the call
  returns success. `SetLimits` refuses it with `ErrSoftAboveHard`.
- **Project 0 on XFS is the default for every project**
  (`xfs_qm_scall_setqlim`). `SetLimits` refuses id 0 with `ErrInvalidProject`.
- **XFS has no record of a project that never used anything** and answers
  `ENOENT`; `Usage` reports that as a zero `Quota`.

### Units and rounding

`Limits` and `Quota` are in bytes. The kernel ABIs are not: XFS speaks
512-byte basic blocks and the generic interface 1 KiB blocks for limits, so
limits are rounded **up** to those units, as the kernel's own conversions
(`quota_btobb`, `stoqb`) do — and XFS rounds up again to its filesystem
block. `Usage` reads back what was stored. On ext4 `Quota.Bytes` is exact; on
XFS it is a multiple of 512.

`statfs(2)` — and so `df` — of a directory carrying `FS_XFLAG_PROJINHERIT`
reports the project's limit as the size of the filesystem: the **soft** limit
when one is set, else the hard limit.

## A full project

**XFS says `ENOSPC`, ext4 says `EDQUOT`.** XFS reports an exhausted project
quota as a full filesystem (`fs/xfs/xfs_trans_dquot.c`); a file server should
treat both as "share full".

## Errors

| Error | When |
|---|---|
| `ErrUnsupported` | every kernel operation off Linux |
| `ErrUnsupportedFilesystem` | neither XFS nor ext4 (wrapped, with the statfs magic) |
| `ErrInvalidProject` | project id 4294967295 anywhere; project 0 for `SetLimits` |
| `ErrSoftAboveHard` | a non-zero hard limit below its soft limit |
| `*QuotaError` | a failed `quotactl_fd(2)`: `Op`, `Path`, `ID`, the errno (`errors.Is(err, unix.EPERM)` works) and a `Hint` |

The hints name the likely cause: `EPERM` — `CAP_SYS_ADMIN` is needed;
`ENOSYS` — no `quotactl_fd` before Linux 5.14, a kernel without
`CONFIG_QUOTA`, or quotas not enabled on the mount; `ESRCH` — project quotas
not enabled (mount with `-o prjquota`); `EROFS` — a read-only mount;
`ERANGE` — a limit larger than the quota format can store.

## Testing

```sh
# Unprivileged: ABI layouts and ioctl numbers, unit conversions, and every
# kernel-call branch through fault-injecting seams. 100% coverage on Linux.
GOWORK=off go test ./...

# Root: real XFS and ext4 in loop-mounted images (needs mkfs.xfs, mkfs.ext4).
GOWORK=off go test -c -o projquota.test . && sudo ./projquota.test -test.v -test.run '^TestIntegration'
```

CI runs the unit tests natively on amd64 and arm64 with a 100.0% coverage
floor, under QEMU on riscv64, loong64, ppc64le and s390x, and the root
integration tests on a hosted runner's own kernel.

## Used by

[fileshare](https://go-fileshare.github.io/docs/latest/administration/volumes/)'s
volumes (v0.21.0): its provisioner creates XFS and ext4 volumes as project
directories with this package, and the server refuses to serve them while it
runs as root or holds `CAP_SYS_RESOURCE`.

## Licence

BSD-3-Clause.
