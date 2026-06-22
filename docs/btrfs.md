# `btrfs`

`github.com/go-fsctl/btrfs`

Pure-Go btrfs kernel control: drive btrfs operations directly via
`BTRFS_IOC_*` ioctls on directory file descriptors — no cgo, and no
shelling out to the `btrfs` CLI.

Where [`go-fsctl/zfs`](zfs.md) talks to the OpenZFS kernel module through
`/dev/zfs` and native `nvlist`s, this package talks to the btrfs kernel
module the way `btrfs-progs` does — by opening a directory on a btrfs
mount and issuing struct-based ioctls (magic `'X'` = `0x94`,
`_IOW`/`_IOWR`-encoded). Validated against a live Linux 6.12 kernel
(arm64) on a `mkfs.btrfs` loopback mount. The ABI structs and
`BTRFS_IOC_*` numbers are derived from the kernel uapi header
`linux/btrfs.h`.

## Install

```sh
go get github.com/go-fsctl/btrfs
```

## Subvolumes and snapshots

```go
import "github.com/go-fsctl/btrfs"

const mnt = "/mnt/bt" // a mounted btrfs filesystem

// Create a subvolume (BTRFS_IOC_SUBVOL_CREATE):
err := btrfs.SubvolCreate(mnt, "sub1")

// Read-only snapshot of it (BTRFS_IOC_SNAP_CREATE_V2, flags=BTRFS_SUBVOL_RDONLY):
err = btrfs.SnapshotCreate(mnt+"/sub1", mnt, "snap1", true)

// Toggle the read-only flag (BTRFS_IOC_SUBVOL_GET/SETFLAGS):
ro, err := btrfs.IsReadonly(mnt + "/snap1")
err = btrfs.SetReadonly(mnt+"/sub1", true)
// lower-level flag access:
flags, err := btrfs.SubvolGetFlags(mnt + "/sub1")
err = btrfs.SubvolSetFlags(mnt+"/sub1", flags)

// Inspect (BTRFS_IOC_GET_SUBVOL_INFO / BTRFS_IOC_INO_LOOKUP):
info, err := btrfs.GetSubvolInfo(mnt + "/sub1") // *SubvolInfo
id, err := btrfs.SubvolID(mnt + "/sub1")

// Force a transaction commit (BTRFS_IOC_SYNC):
err = btrfs.Sync(mnt)

// Delete a subvolume/snapshot (BTRFS_IOC_SNAP_DESTROY):
err = btrfs.SubvolDelete(mnt, "sub1")

// List all subvolumes (BTRFS_IOC_TREE_SEARCH_V2 over the root tree):
subs, err := btrfs.ListSubvolumes(mnt) // []Subvolume{ID, ParentID, Name, Path}
```

## Devices

```go
err := btrfs.DeviceAdd(mnt, "/dev/loop1")        // BTRFS_IOC_ADD_DEV
fi, err := btrfs.GetFsInfo(mnt)                   // *FsInfo: NumDevices, MaxID, FSID, ...
dev, err := btrfs.GetDeviceInfo(mnt, 1)           // *DeviceInfo: Devid, Path, TotalBytes, ...
err = btrfs.DeviceRemove(mnt, "/dev/loop1")       // BTRFS_IOC_RM_DEV_V2 (-> RM_DEV fallback)
err = btrfs.DeviceRemoveByID(mnt, 2)
```

## Scrub

```go
sp, err := btrfs.ScrubStart(mnt, 1, btrfs.ScrubOptions{}) // BTRFS_IOC_SCRUB (synchronous)
sp, err = btrfs.ScrubProgressFor(mnt, 1)                  // BTRFS_IOC_SCRUB_PROGRESS
err = btrfs.ScrubCancel(mnt)                              // BTRFS_IOC_SCRUB_CANCEL
```

## Balance

```go
bp, err := btrfs.BalanceStart(mnt, btrfs.BalanceArgs{})   // BTRFS_IOC_BALANCE_V2 (full, synchronous)
bp, err = btrfs.BalanceProgressFor(mnt)                   // BTRFS_IOC_BALANCE_PROGRESS
err = btrfs.BalancePause(mnt)                             // BTRFS_IOC_BALANCE_CTL
err = btrfs.BalanceCancel(mnt)

// The `-dusage=50` equivalent (relocate data chunks below 50% full):
bp, err = btrfs.BalanceStart(mnt, btrfs.BalanceArgs{
    Data: &btrfs.BalanceFilter{Flags: btrfs.BalanceArgsUsage, Usage: 50},
})
```

## Quotas / qgroups

```go
err := btrfs.QuotaEnable(mnt)                       // or QuotaDisable(mnt)
err = btrfs.QgroupCreate(mnt, 1<<48|100)            // higher-level qgroup 1/100
err = btrfs.QgroupDestroy(mnt, 1<<48|100)
err = btrfs.QgroupAssign(mnt, 0<<48|256, 1<<48|100) // QgroupRemove to undo
err = btrfs.QgroupLimit(mnt, 0<<48|256,             // cap referenced bytes (writes past it -> EDQUOT)
    btrfs.QgroupLimits{Flags: btrfs.QgroupLimitMaxRfer, MaxRfer: 16 << 20})
qgs, err := btrfs.ListQgroups(mnt)                  // []Qgroup{ID, Level, SubvolID, Rfer, Excl, ...}
```

## Defragment

```go
err := btrfs.Defrag(mnt + "/file")                              // whole file or a directory's b-tree
err = btrfs.DefragRange(mnt+"/file", btrfs.DefragRangeOptions{}) // zero-value = whole file
```

## Send / receive

```go
import "os"

snap, _ := os.Open(mnt + "/snap_ro")   // a READ-ONLY snapshot (send requires RO)

err := btrfs.Send(int(snap.Fd()), w, btrfs.SendOpts{})                    // BTRFS_IOC_SEND, full -> io.Writer
err = btrfs.Send(int(snap.Fd()), w, btrfs.SendOpts{ParentRoot: parentID}) // incremental delta
err = btrfs.Send(int(snap.Fd()), w, btrfs.SendOpts{NoData: true})         // metadata-only

// Mark a received subvolume (the interop primitive `btrfs receive` uses):
res, err := btrfs.SetReceivedSubvol(fd, uuid, ctransid, btrfs.SetReceivedTimes{})
```

The send-stream parsing primitives work on any platform (no kernel calls):

```go
h, n, err := btrfs.VerifyStream(r)       // Header{Magic, Version} + record count
h, err = btrfs.ParseHeader(r)            // just the stream header
cr := btrfs.NewCommandReader(r).WithData()
for {
    cmd, err := cr.Next()                // decode one Command record
    if cmd.IsEnd() { break }
    // ...
}
```

## API reference

| Operation | ioctl | Function |
| --- | --- | --- |
| Create subvolume | `BTRFS_IOC_SUBVOL_CREATE` | `SubvolCreate(parentDir, name string) error` |
| Create snapshot (RO/RW) | `BTRFS_IOC_SNAP_CREATE_V2` | `SnapshotCreate(srcSubvolPath, destParentDir, name string, readonly bool) error` |
| Delete subvolume/snapshot | `BTRFS_IOC_SNAP_DESTROY` | `SubvolDelete(parentDir, name string) error` |
| Get / set subvolume flags | `BTRFS_IOC_SUBVOL_GET/SETFLAGS` | `IsReadonly`, `SetReadonly`, `SubvolGetFlags`, `SubvolSetFlags` |
| Subvolume id / info | `BTRFS_IOC_INO_LOOKUP` / `GET_SUBVOL_INFO` | `SubvolID`, `GetSubvolInfo` |
| Force commit | `BTRFS_IOC_SYNC` | `Sync(path string) error` |
| List subvolumes | `BTRFS_IOC_TREE_SEARCH_V2` (→ `TREE_SEARCH`) | `ListSubvolumes(path string) ([]Subvolume, error)` |
| Add / remove device | `BTRFS_IOC_ADD_DEV` / `RM_DEV_V2` | `DeviceAdd`, `DeviceRemove`, `DeviceRemoveByID` |
| Filesystem / device info | `BTRFS_IOC_FS_INFO` / `DEV_INFO` | `GetFsInfo`, `GetDeviceInfo` |
| Scrub | `BTRFS_IOC_SCRUB` / `SCRUB_PROGRESS` / `SCRUB_CANCEL` | `ScrubStart`, `ScrubProgressFor`, `ScrubCancel` |
| Balance | `BTRFS_IOC_BALANCE_V2` / `BALANCE_PROGRESS` / `BALANCE_CTL` | `BalanceStart`, `BalanceProgressFor`, `BalancePause`, `BalanceCancel` |
| Quota / qgroup | `BTRFS_IOC_QUOTA_CTL` / `QGROUP_*` | `QuotaEnable/Disable`, `QgroupCreate/Destroy/Assign/Remove/Limit`, `ListQgroups` |
| Defragment | `BTRFS_IOC_DEFRAG` / `DEFRAG_RANGE` | `Defrag`, `DefragRange` |
| Send | `BTRFS_IOC_SEND` | `Send`, `SetReceivedSubvol` |
| Stream parsing | — (no kernel) | `VerifyStream`, `ParseHeader`, `NewCommandReader` |

`Available(path)` reports whether `path` is on a mounted btrfs filesystem
(`statfs` + `BTRFS_SUPER_MAGIC`); integration tests use it to skip
elsewhere. On non-Linux platforms every kernel operation returns
`ErrUnsupported`.

## Testing

```sh
GOWORK=off go test ./...        # host-runnable unit tests (ioctl numbers, struct sizes)
sudo -E go test ./...           # integration: gated on a mounted btrfs filesystem + root
```
