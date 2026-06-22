# `dm`

`github.com/go-fsctl/dm`

Pure-Go control of the Linux **device-mapper** subsystem, driven directly
through the `/dev/mapper/control` character device and the `DM_*` ioctls.

- **No cgo.** Just the standard library and `golang.org/x/sys/unix`.
- **No `dmsetup`.** The library speaks the `struct dm_ioctl` wire protocol
  itself; it never shells out.

The `DM_*` request numbers are *derived* in Go from the
`_IOWR(0xfd, cmd, sizeof(struct dm_ioctl))` encoding rather than
hard-coded, and pinned by unit tests.

## Install

```sh
go get github.com/go-fsctl/dm
```

## Lifecycle: create, load, resume, remove

A device-mapper device is built in three steps that mirror the kernel's
active/inactive table model, then torn down with `Remove`:

```go
import "github.com/go-fsctl/dm"

// 1. Create an empty, suspended device.
err := dm.Create("myvol", "")

// 2. Load a table into the inactive slot. A "linear" target maps a run of
//    sectors onto a backing device at a sector offset.
//    Here: 131072 sectors (64 MiB) of /dev/loop0 starting at offset 0.
tgt := dm.Linear(0, 131072, "/dev/loop0", 0)
err = dm.LoadTable("myvol", []dm.Target{tgt})

// 3. Resume: promote the inactive table to active and enable IO.
err = dm.Resume("myvol")
// /dev/mapper/myvol is now live.

err = dm.Remove("myvol")
```

`CreateWithTable(name, targets)` is a one-shot convenience that does
`Create` + `LoadTable` + `Resume` (and cleans up on failure):

```go
err := dm.CreateWithTable("crypt0", []dm.Target{
    dm.Crypt(0, 131072, "aes-xts-plain64", key, 0, "/dev/loop0", 0, nil),
})
```

## Targets

`Target` is intentionally generic, so any kernel target works by supplying
the right `Type` and `Params`:

```go
type Target struct {
    SectorStart uint64
    Length      uint64 // in sectors
    Type        string // "linear", "striped", "crypt", ...
    Params      string
}

func (t Target) String() string
```

For the common targets there are typed constructors that emit the exact
param string the kernel expects, so you do not have to remember the
grammar:

```go
dm.Linear(0, n, "/dev/loop0", 0)                       // linear:  <dev> <offset>
dm.Striped(0, n, 256, []dm.StripeDev{                  // striped: <num> <chunk> (<dev> <off>)...
    {Dev: "/dev/loop0", Offset: 0},
    {Dev: "/dev/loop1", Offset: 0},
})
dm.Zero(0, n)                                          // zero:  reads zero, writes dropped
dm.Error(0, n)                                         // error: all I/O fails
dm.Snapshot(0, n, "/dev/origin", "/dev/cow", true, 8)  // snapshot: <origin> <cow> <P|N> <chunk>
dm.SnapshotOrigin(0, n, "/dev/origin")                 // snapshot-origin: <origin>
dm.Crypt(0, n, "aes-xts-plain64", key, 0, "/dev/loop0", 0, nil)
    // crypt: <cipher> <hexkey> <iv> <dev> <off> [<#opt> <opt>...]
    // key is raw bytes; Crypt hex-encodes it as the dm-crypt table format requires.
dm.ThinPool(0, n, "/dev/meta", "/dev/data", 128, 1024, nil)
    // thin-pool: <meta> <data> <data_block_sectors> <low_water_blocks> [<#opt> <opt>...]
dm.Thin(0, n, "/dev/mapper/pool", 0, "")
    // thin: <pool> <dev_id> [<external_origin>]
dm.Verity(0, n, dm.VerityParams{Version: 1, DataDev: "/dev/data", HashDev: "/dev/hash",
    DataBlockSize: 4096, HashBlockSize: 4096, NumDataBlocks: n / 8, HashStartBlock: 1,
    Algorithm: "sha256", RootDigest: root, Salt: salt})
```

Each constructor only assembles the table line — the kernel does the
striping, copy-on-write, and crypto.

## Thin provisioning

A thin pool hands out data blocks on demand to thin volumes and snapshots,
all identified by numeric device ids. The pool's metadata device must be
**zeroed** before first use. Thin volumes and snapshots are created
*inside* the pool by sending it target messages (`DM_TARGET_MSG`), not by
reloading its table:

```go
// Pool over a zeroed metadata device and a data device, 64 KiB blocks.
err := dm.CreateWithTable("pool", []dm.Target{
    dm.ThinPool(0, n, "/dev/meta", "/dev/data", 128, 1024, nil),
})

err = dm.ThinPoolCreateThin("pool", 1)       // create thin volume id 1
err = dm.ThinPoolCreateSnap("pool", 2, 1)    // snapshot of id 1 as id 2
err = dm.ThinPoolDeleteThin("pool", 1)       // remove a thin/snap by id
```

`Message(name, sector, msg)` is the general `DM_TARGET_MSG` wrapper (sector
is usually 0 to address a single-row table); the `ThinPool*` helpers are
thin wrappers over it.

## dm-verity

`Verity` builds a read-only integrity-checked mapping over a data device
and a precomputed Merkle hash tree (e.g. from `veritysetup format`). The
device **must be created read-only**, so use `CreateReadOnlyWithTable` (or
`LoadTableReadOnly`):

```go
err := dm.CreateReadOnlyWithTable("verity0", []dm.Target{
    dm.Verity(0, dataSectors, dm.VerityParams{
        Version: 1, DataDev: "/dev/data", HashDev: "/dev/hash",
        DataBlockSize: 4096, HashBlockSize: 4096,
        NumDataBlocks: nBlocks, HashStartBlock: 1,
        Algorithm: "sha256", RootDigest: rootHex, Salt: saltHex,
    }),
})
```

Every block read is verified against the root digest; a mismatch returns
EIO and flips `ParseVerityStatus` from `"V"` to `"C"`. Optional `Opts`
(e.g. `ignore_zero_blocks`) and forward error correction (`VerityFEC`) are
folded into the table's optional-argument list.

## Inspect

```go
info, err := dm.Info("myvol")        // DM_DEV_STATUS: dev_t, open count, flags, target count
info.Suspended()
info.ReadOnly()
info.ActivePresent()
info.InactivePresent()
info.Major(); info.Minor()

tbl, err := dm.TableStatus("myvol")  // DM_TABLE_STATUS: read back the active table
st, err := dm.Status("myvol")        // DM_TABLE_STATUS: per-target runtime status
devs, err := dm.List()               // DM_LIST_DEVICES: enumerate all dm devices
v, err := dm.Version()               // DM_VERSION: negotiate interface version
```

Runtime-status decoders are provided per target type, each with a separate
`Parse*Status` for parsing a status string directly:

```go
snap, err := dm.SnapshotStatus("snap0")  // -> SnapStatus (used/total exception store)
pool, err := dm.ThinPoolStatus("pool")   // -> ThinPoolStat (data + metadata block usage)
thin, err := dm.ThinStatus("thin1")      // -> ThinStat (mapped sectors)
dm.ParseSnapStatus(s); dm.ParseThinPoolStatus(s); dm.ParseThinStatus(s); dm.ParseVerityStatus(s)
```

## API reference

| Function | ioctl | Purpose |
| --- | --- | --- |
| `Available() bool` | — | `/dev/mapper/control` openable |
| `Version() (DMVersion, error)` | `DM_VERSION` | negotiate interface version |
| `Create(name, uuid string) error` | `DM_DEV_CREATE` | create empty suspended device |
| `CreateWithTable(name string, targets []Target) error` | create+load+resume | one-shot bring-up |
| `CreateReadOnlyWithTable(name string, targets []Target) error` | create+load+resume | as above, read-only (dm-verity) |
| `LoadTable(name string, targets []Target) error` | `DM_TABLE_LOAD` | load table into inactive slot |
| `LoadTableReadOnly(name string, targets []Target) error` | `DM_TABLE_LOAD` | load read-only table |
| `Message(name string, sector uint64, msg string) (string, error)` | `DM_TARGET_MSG` | send a target message |
| `Suspend(name string) error` | `DM_DEV_SUSPEND` | suspend IO |
| `Resume(name string) error` | `DM_DEV_SUSPEND` | resume / activate |
| `Info(name string) (DevInfo, error)` | `DM_DEV_STATUS` | dev_t, open count, flags, target count |
| `TableStatus(name string) ([]Target, error)` | `DM_TABLE_STATUS` | read back the active table |
| `Status(name string) ([]Target, error)` | `DM_TABLE_STATUS` | per-target runtime status |
| `List() ([]Device, error)` | `DM_LIST_DEVICES` | enumerate all dm devices |
| `Remove(name string) error` | `DM_DEV_REMOVE` | remove device, destroy tables |

Target constructors: `Linear`, `Striped`, `Zero`, `Error`, `Snapshot`,
`SnapshotOrigin`, `Crypt`, `ThinPool`, `Thin`, `Verity`. Thin lifecycle:
`ThinPoolCreateThin`, `ThinPoolCreateSnap`, `ThinPoolDeleteThin`.

On non-Linux platforms every kernel operation returns `ErrUnsupported`,
while the ABI definitions and the target-spec (de)serialization in
`abi.go` remain available for tooling and tests.

## Testing

```sh
# Unit tests (ioctl numbers, struct sizes/offsets, target-spec serialization):
GOWORK=off go test ./...

# Integration tests are gated on /dev/mapper/control plus root:
sudo -E go test -run Live -v ./...
```
