# `zfs`

`github.com/go-fsctl/zfs`

Pure-Go `libzfs_core`: drive ZFS kernel operations via `/dev/zfs` ioctls —
no cgo, no `libzfs`, no shelling out to `zpool` / `zfs` / `zdb`.

This library talks to the live OpenZFS kernel module the same way OpenZFS's
own `libzfs_core` does: by opening `/dev/zfs` and issuing `ZFS_IOC_*`
ioctls whose payloads are `nvlist`s packed in the kernel's native,
host-endian (`NV_ENCODE_NATIVE`) wire format. That native encoding is
distinct from the XDR encoding used in on-disk vdev labels — this package
implements the native codec from scratch.

Targets **OpenZFS 2.2.x on Linux** (validated against 2.2.2, aarch64). The
`zfs_cmd_t` ABI and `ZFS_IOC_*` numbers are pinned to that release.

## Install

```sh
go get github.com/go-fsctl/zfs
```

## Open the handle

```go
import "github.com/go-fsctl/zfs"

h, err := zfs.Open()   // open("/dev/zfs")
defer h.Close()
```

`Open` returns a `*Handle`; every operation below is a method on it.

## Pool lifecycle

Pure-Go pool creation — no libzfs, no `zpool(8)`:

```go
root := zfs.Vdev{Type: zfs.VDEV_TYPE_ROOT, Children: []zfs.Vdev{
    {Type: zfs.VDEV_TYPE_FILE, Path: "/var/tmp/disk0.img"},
}}
err = h.PoolCreate("tank", root, nil)             // ZFS_IOC_POOL_CREATE
cfgs, err := h.PoolConfigs()                      // ZFS_IOC_POOL_CONFIGS -> map[name]Nvlist
names, err := h.PoolNames()                       // imported pool names
pp, err := h.PoolGetProps("tank")                 // ZFS_IOC_POOL_GET_PROPS -> map[string]Value
err = h.PoolExport("tank", false, false)          // ZFS_IOC_POOL_EXPORT (force, hardforce)
_, err = h.PoolImport("tank", cfgs["tank"])       // ZFS_IOC_POOL_IMPORT
tc, err := h.PoolTryImport(cfgs["tank"])          // ZFS_IOC_POOL_TRYIMPORT (probe a config)
err = h.PoolDestroy("tank")                       // ZFS_IOC_POOL_DESTROY
```

## Dataset lifecycle

```go
err = h.CreateFilesystem("tank/ds1")              // ZFS_IOC_CREATE
err = h.SetProp("tank/ds1", zfs.Nvlist{"quota": uint64(64 << 20)}) // ZFS_IOC_SET_PROP
props, err := h.GetProps("tank/ds1")              // ZFS_IOC_OBJSET_STATS (flattened) -> map[string]Value
nv, err := h.ObjsetStats("tank/ds1")              // the raw Nvlist
err = h.Rename("tank/ds1", "tank/ds2", false)     // ZFS_IOC_RENAME
err = h.Snapshot("tank", []string{"tank/ds2@s1"}) // ZFS_IOC_SNAPSHOT
err = h.Destroy("tank/ds2@s1", false)             // ZFS_IOC_DESTROY (defer)
```

`SetProp` takes the kernel's native value type per property: a `uint64`
enum index for `INDEX` properties (e.g. `compression`, `atime`) and
`NUMBER` properties (e.g. `quota`), or a `string` for `STRING`-typed
properties — the same conversion the `zpool`/`zfs` CLI performs before the
ioctl. Enabling a feature-gated value (e.g. `compression=lz4`) requires
that feature to be enabled on the pool at creation time.

## Clone / rollback / hold / bookmark

```go
err = h.Clone("tank/ds2@s1", "tank/clone", nil)   // ZFS_IOC_CLONE
target, err := h.Rollback("tank/ds2")             // ZFS_IOC_ROLLBACK (-> latest snapshot)
target, err = h.RollbackTo("tank/ds2", "tank/ds2@s1")
err = h.Hold("tank/ds2@s1", "keep", false)        // ZFS_IOC_HOLD (blocks destroy with EBUSY)
holds, err := h.Holds("tank/ds2@s1")              // ZFS_IOC_GET_HOLDS (tag -> timestamp)
err = h.Release("tank/ds2@s1", "keep")            // ZFS_IOC_RELEASE
err = h.Bookmark("tank/ds2@s1", "tank/ds2#bm1")   // ZFS_IOC_BOOKMARK
bms, err := h.GetBookmarks("tank/ds2")            // ZFS_IOC_GET_BOOKMARKS
err = h.DestroyBookmarks("tank/ds2#bm1")          // ZFS_IOC_DESTROY_BOOKMARKS
```

## Promote / inherit

```go
err = h.Promote("tank/clone")                     // ZFS_IOC_PROMOTE (clone becomes origin)
err = h.Inherit("tank/ds2", "compression", false) // ZFS_IOC_INHERIT_PROP (clear local prop)
```

## Encryption

The wrapping key travels in the hidden-args channel as a
`DATA_TYPE_UINT8_ARRAY`; the `encryption` / `keyformat` props are NUMERIC
enums (the kernel reads them as `uint64`), `keylocation` is a string.

```go
key := make([]byte, zfs.WRAPPING_KEY_LEN)         // 32 raw bytes for keyformat=raw
err = h.CreateEncrypted("tank/enc", key, zfs.Nvlist{ // ZFS_IOC_CREATE + wkeydata
    "encryption":  uint64(zfs.ZIO_CRYPT_AES_256_GCM),
    "keyformat":   uint64(zfs.ZFS_KEYFORMAT_RAW),
    "keylocation": "prompt",
})
err = h.UnloadKey("tank/enc")                     // ZFS_IOC_UNLOAD_KEY (keystatus -> unavailable)
err = h.LoadKey("tank/enc", key, false)           // ZFS_IOC_LOAD_KEY  (keystatus -> available)
newKey := make([]byte, zfs.WRAPPING_KEY_LEN)
err = h.ChangeKey("tank/enc", newKey, zfs.Nvlist{ // ZFS_IOC_CHANGE_KEY
    "keyformat":   uint64(zfs.ZFS_KEYFORMAT_RAW),
    "keylocation": "prompt",
})
```

## Send / receive (replication)

The kernel writes/reads the DMU replay stream to/from the file descriptor;
the library only drives the ioctl + nvlist.

```go
import "os"

out, _ := os.Create("snap.stream")
err = h.Send("tank/ds2@s1", out, zfs.SendOptions{})  // ZFS_IOC_SEND_NEW
// incremental: zfs.SendOptions{FromSnap: "s0", LargeBlocks: true, Compress: true}

in, _ := os.Open("snap.stream")
br, err := h.Receive("tank/restored@s1", in, zfs.RecvOptions{}) // ZFS_IOC_RECV_NEW
// br.ToName / br.ToGuid / br.Type come from the stream's DRR_BEGIN record.
```

## The native nvlist codec

Exported and usable on any platform (no kernel calls):

```go
b, err := zfs.EncodeNative(zfs.Nvlist{"name": "tank", "version": uint64(5000)})
nv, err := zfs.DecodeNative(b)
```

`Nvlist` is `map[string]Value` where `Value` is `any`; the codec supports
the kernel's typed values including `Boolean`, `Byte`, `Uint8Array`, and
nested `Nvlist`s.

## API reference

| Operation | ioctl | Method |
| --- | --- | --- |
| Open / close `/dev/zfs` | — | `Open() (*Handle, error)`, `(*Handle) Close() error` |
| List imported pools | `ZFS_IOC_POOL_CONFIGS` | `PoolConfigs`, `PoolNames` |
| Pool create / destroy | `ZFS_IOC_POOL_CREATE` / `DESTROY` | `PoolCreate`, `PoolDestroy` |
| Pool import / export / try | `ZFS_IOC_POOL_IMPORT` / `EXPORT` / `TRYIMPORT` | `PoolImport`, `PoolExport`, `PoolTryImport` |
| Pool / dataset props | `ZFS_IOC_POOL_GET_PROPS` / `OBJSET_STATS` / `SET_PROP` | `PoolGetProps`, `GetProps`, `ObjsetStats`, `SetProp` |
| Create filesystem / snapshot | `ZFS_IOC_CREATE` / `SNAPSHOT` | `CreateFilesystem`, `CreateEncrypted`, `Snapshot` |
| Rename / destroy | `ZFS_IOC_RENAME` / `DESTROY` | `Rename`, `Destroy` |
| Clone / rollback | `ZFS_IOC_CLONE` / `ROLLBACK` | `Clone`, `Rollback`, `RollbackTo` |
| Hold / release | `ZFS_IOC_HOLD` / `GET_HOLDS` / `RELEASE` | `Hold`, `Holds`, `Release` |
| Bookmark | `ZFS_IOC_BOOKMARK` / `GET_BOOKMARKS` / `DESTROY_BOOKMARKS` | `Bookmark`, `GetBookmarks`, `DestroyBookmarks` |
| Promote / inherit | `ZFS_IOC_PROMOTE` / `INHERIT_PROP` | `Promote`, `Inherit` |
| Encryption keys | `ZFS_IOC_LOAD_KEY` / `UNLOAD_KEY` / `CHANGE_KEY` | `LoadKey`, `UnloadKey`, `ChangeKey` |
| Send / receive | `ZFS_IOC_SEND_NEW` / `RECV_NEW` | `Send`, `Receive` |
| Native nvlist codec | — | `EncodeNative`, `DecodeNative` |

`Available()` reports whether `/dev/zfs` is present. On non-Linux
platforms every kernel operation returns `ErrUnsupported`, while the
native nvlist codec stays available.

## Testing

```sh
GOWORK=off go test ./...   # host-runnable unit tests incl. the native nvlist codec
sudo -E go test ./...      # integration: gated on /dev/zfs (loaded OpenZFS module) + root
```
