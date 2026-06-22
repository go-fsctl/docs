# `loop`

`github.com/go-fsctl/loop`

Pure-Go Linux **loop-device** control: attach, configure, inspect, and
detach loop devices directly through `/dev/loop-control` and the `LOOP_*`
ioctls — the same kernel interface util-linux's `losetup` uses — with no
cgo and no shelling out.

Validated against a live Linux 6.12 kernel (arm64): the attachment is
cross-checked with `losetup -a` / `losetup -j`, `Status()` matches, a
write/read round-trip succeeds on `/dev/loopN`, and `Detach()` removes it.
The ABI structs and `LOOP_*` numbers are derived from the kernel uapi
header `linux/loop.h`.

## Install

```sh
go get github.com/go-fsctl/loop
```

## Attach and detach

```go
import "github.com/go-fsctl/loop"

// Attach a backing file to the first free loop device
// (LOOP_CTL_GET_FREE + LOOP_CONFIGURE, falling back to
// LOOP_SET_FD + LOOP_SET_STATUS64):
dev, err := loop.Attach("/path/to/disk.img", loop.Options{
    Offset:    1 << 20, // start 1 MiB into the file
    SizeLimit: 0,        // 0 = to end of file
    ReadOnly:  false,
    Autoclear: false,    // LO_FLAGS_AUTOCLEAR
    PartScan:  false,    // LO_FLAGS_PARTSCAN
})
// dev == "/dev/loop3"

// Detach when done (LOOP_CLR_FD):
err = loop.Detach(dev)
```

`Options` carries the configurable attach parameters:

```go
type Options struct {
    Offset    uint64 // start offset into the backing file, in bytes
    SizeLimit uint64 // exposed size in bytes; 0 = rest of the file
    ReadOnly  bool   // attach read-only
    Autoclear bool   // LO_FLAGS_AUTOCLEAR (auto-detach on last close)
    PartScan  bool   // LO_FLAGS_PARTSCAN (scan for partitions)
}
```

## Inspect

```go
// LOOP_GET_STATUS64:
info, err := loop.Status(dev)
// info.Number, info.Offset, info.SizeLimit, info.Flags, info.BackingFile
// flag accessors:
info.ReadOnly()
info.Autoclear()
info.PartScan()
```

`Info` is the decoded device status; the three boolean methods test the
`LO_FLAGS_*` bits:

```go
type Info struct {
    Number      int
    Offset      uint64
    SizeLimit   uint64
    Flags       uint32
    BackingFile string
}

func (i Info) ReadOnly() bool  // LO_FLAGS_READ_ONLY
func (i Info) Autoclear() bool // LO_FLAGS_AUTOCLEAR
func (i Info) PartScan() bool  // LO_FLAGS_PARTSCAN
```

## Find devices by backing file

```go
// Scans /sys/block/loop*/loop/backing_file (no ioctl):
devs, err := loop.FindByBacking("/path/to/disk.img")
// devs == ["/dev/loop3", ...]
```

## Refresh capacity after growing the backing file

```go
// LOOP_SET_CAPACITY — re-read the backing file's size:
err := loop.SetCapacity(dev)
```

## Manage device nodes via /dev/loop-control

```go
n, err := loop.CtlAdd(8)    // LOOP_CTL_ADD    -> creates /dev/loop8
err = loop.CtlRemove(8)     // LOOP_CTL_REMOVE -> removes /dev/loop8
```

## Helpers

```go
ok := loop.Available()                   // /dev/loop-control exists (no privilege needed)
n, err := loop.DeviceNumber("/dev/loop3") // parse the trailing number -> 3
```

## API reference

| Function | ioctl | Purpose |
| --- | --- | --- |
| `Available() bool` | — | `/dev/loop-control` exists |
| `Attach(imagePath string, opt Options) (devPath string, err error)` | `LOOP_CTL_GET_FREE` + `LOOP_CONFIGURE` (fallback `LOOP_SET_FD` + `LOOP_SET_STATUS64`) | attach a file to the first free loop device |
| `Detach(devPath string) error` | `LOOP_CLR_FD` | detach |
| `Status(devPath string) (Info, error)` | `LOOP_GET_STATUS64` | read device status |
| `SetCapacity(devPath string) error` | `LOOP_SET_CAPACITY` | re-read backing-file size |
| `FindByBacking(path string) ([]string, error)` | — (sysfs) | devices backed by a file |
| `CtlAdd(n int) (int, error)` | `LOOP_CTL_ADD` | create `/dev/loopN` |
| `CtlRemove(n int) error` | `LOOP_CTL_REMOVE` | remove `/dev/loopN` |
| `DeviceNumber(devPath string) (int, error)` | — | parse `N` from `/dev/loopN` |

All mutating operations require `CAP_SYS_ADMIN` (in practice, root). On
non-Linux platforms every function returns `loop.ErrUnsupported`.

## Testing

```sh
# Host-runnable unit tests (ioctl numbers, struct sizes/offsets, flags):
GOWORK=off go test ./...

# Integration tests are gated on /dev/loop-control + root:
sudo -E go test ./...
```

There is also a live demo binary, `cmd/loopprobe`, that attaches a temp
image, prints its kernel status, does a round-trip, and detaches it.
