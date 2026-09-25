# `blk`

`github.com/go-fsctl/blk`

Pure-Go **generic Linux block-device ioctls**: `BLK*` from
`linux/fs.h`, `BLKPG` from `linux/blkpg.h`, and the zoned queries from
`linux/blkzoned.h`. Size and geometry, discard and zero-out, the
read-only flag, and partition-table maintenance — with no cgo and
without shelling out to `blockdev`, `hdparm` or `partx`.

!!! danger "These operate on real devices"
    `Discard`, `SecureDiscard` and `ZeroOut` destroy data, and the
    partition calls edit a live kernel partition table. Point them at a
    loop device (see [`loop/`](loop.md)) or a scratch disk — never at
    the machine you are working on.

## Platforms

The ioctls exist only on Linux. Everywhere else **every operation
returns `ErrUnsupported`** rather than failing in some device-specific
way:

```go
_, err := blk.GetSize64(fd)
// darwin: blk: block-device ioctls are only supported on Linux
```

The ABI definitions and the ioctl-number derivation stay available on
every platform, so tooling and tests that only need the numbers build
anywhere:

```go
blk.BLKGETSIZE64  // 0x80081272
blk.BLKDISCARD    // 0x1277
```

## Install

```sh
go get github.com/go-fsctl/blk
```

Every call takes an already-open file descriptor, so opening the device
— and deciding whether to open it `O_RDONLY` — stays with the caller.

```go
f, err := os.OpenFile("/dev/loop0", os.O_RDWR, 0)
if err != nil { return err }
defer f.Close()
fd := int(f.Fd())
```

## Size and geometry

```go
bytes, err := blk.GetSize64(fd)          // BLKGETSIZE64, the one to use
sectors, err := blk.GetSize(fd)          // BLKGETSIZE, in 512-byte sectors
```

```go
blk.GetBlockSize(fd)        // BLKBSZGET — the soft block size
blk.SetBlockSize(fd, 4096)  // BLKBSZSET
blk.GetSectorSize(fd)       // BLKSSZGET — logical sector size
blk.GetPhysBlockSize(fd)    // BLKPBSZGET — physical, which may be larger
blk.GetIOMin(fd)            // BLKIOMIN — minimum I/O the device prefers
blk.GetIOOpt(fd)            // BLKIOOPT — optimal I/O size
blk.GetAlignmentOffset(fd)  // BLKALIGNOFF
```

The logical and physical sizes differ on 512e drives — a 4096-byte
physical sector presented as 512 — which is what
`GetAlignmentOffset` exists to tell you about.

## Discard, secure discard, zero-out

```go
blk.Discard(fd, start, length)        // BLKDISCARD
blk.SecureDiscard(fd, start, length)  // BLKSECDISCARD
blk.ZeroOut(fd, start, length)        // BLKZEROOUT
blk.GetDiscardZeroes(fd)              // BLKDISCARDZEROES
```

`GetDiscardZeroes` reports whether a discarded region reads back as
zeroes. It is a property of the device, not a promise of the call:
where it is false, a discard frees blocks without defining what a read
returns, and `ZeroOut` is what actually writes zeroes.

## The read-only flag, and flushing

```go
blk.SetReadOnly(fd, true)   // BLKROSET
ro, err := blk.GetReadOnly(fd) // BLKROGET
blk.FlushBuf(fd)            // BLKFLSBUF — drop the buffer cache for this device
```

## Partitions

```go
blk.AddPartition(fd, blk.Partition{Number: 1, Start: 1 << 20, Length: 64 << 20})
blk.ResizePartition(fd, blk.Partition{Number: 1, Start: 1 << 20, Length: 128 << 20})
blk.DelPartition(fd, 1)
blk.RereadPartitionTable(fd)  // BLKRRPART
```

`Partition` is `{Number int; Start, Length int64}`, with `Start` and
`Length` in **bytes**.

The `BLKPG` calls tell the kernel about one partition without
re-reading the whole table, which `RereadPartitionTable` does and which
fails with `EBUSY` when any partition is mounted. That is the reason
`BLKPG` exists: it is the way to add or remove a partition on a disk
that is in use.

## Zoned devices

```go
blk.GetNumZones(fd)  // BLKGETNRZONES
blk.GetZoneSize(fd)  // BLKGETZONESZ, in 512-byte sectors
```

## Licence

BSD-3-Clause.
