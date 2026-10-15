# kfs

KFS is a journaled filesystem written in [sysl](https://github.com/sysl-lang/sysl): the filesystem
[keel](https://github.com/sysl-lang/keel) mounts. It gives programs the POSIX file model — hard and
symbolic links, device nodes, FIFOs, extended attributes, nanosecond times, unlinking a file that is
still open, an atomic `rename` (with `RENAME_NOREPLACE` and `RENAME_EXCHANGE`) — and keeps that model
intact across a power cut at any instant, through a block journal with checksummed commits.

It is meant to be read as well as run. **The design is [`SPEC.md`](SPEC.md)**: the on-disk format
(superblock, groups, bitmaps, inodes, extent trees, hash-indexed directories), the journal and
recovery, the orphan list, the write ordering of every operation with its crash analysis, the locks of
a multi-core kernel (§19), and fsck (§23). Where the specification and the package disagree, the
package is wrong.

## Using it

```hocon
dependencies {
  kfs { git = "github.com/sysl-lang/kfs", version = "0.1.0" }
}
```

```sysl
import sh.sysl.kfs.{default_options, format, mount, ram_device}
```

The package is written against a small **platform** its user supplies: a block device (`read`,
`write`, `flush`, `blocks`), a clock, the named locks of §19, and a hook that runs the commit thread.
`host_platform()` is the host's (trivial locks checked for §19's order, the committer run inline);
keel's kernel supplies its own over keel's threads and virtio-blk. The same source builds for the host
and for `aarch64-freestanding`: nothing in it needs an operating system, a float, or an allocator
beyond `Buf`.

## What is here

| path | what |
|---|---|
| `sh/sysl/kfs/` | the package `kfs`, module `sh.sysl.kfs`: format, mount-time checks, the buffer layer, allocators, inodes, extents, directories, the journal, recovery, every namespace and file operation, the checker and fsck's repairs, and the test devices (`RamDevice`, `CrashDevice`) |
| `host/` | the workspace member `kfshost`, module `sh.sysl.kfshost`: a block device over an image file, the host's entropy and clock, and each tool's engine as a function of its arguments |
| `tools/mkfs-kfs` | `mkfs.kfs <image> <blocks> [-i bytes-per-inode] [-m reserved-percent] [-d dir [-o owners]]` — format an image, optionally filled from a host directory |
| `tools/fsck-kfs` | `fsck.kfs [-n \| -y [--rebuild-corrupt]] <image>` — check, or repair; e2fsck's exit codes |
| `tools/kfsdump` | `kfsdump <image> [super \| groups \| inode <n> \| get <path> <host file>]` — print what an image holds, or copy a file out |

## What it is not

Not a general Linux filesystem driver, and not compatible with ext4 although it borrows ext4's
shapes. Deliberately absent (§1 and §24 give the reasons): copy-on-write, snapshots, compression,
encryption, deduplication; checksums of file data (metadata only); full data journalling (metadata is
journalled, data is ordered before the commit that makes it reachable) and delayed allocation; online
resize, quotas, interpreted POSIX ACLs, block sizes other than 4 KiB, inline data for small files.

## How it is tested

```
sysl test .
sysl build-lib . --target aarch64-freestanding
```

`sysl test .` runs the whole workspace on the host: **607 tests in `kfs`** (about six and a half
minutes) and **72 in `kfshost`**, on sysl 0.1.0-alpha.12.

- **Every-write crash loops** (§27): for each operation — appends and writes across extent boundaries,
  directory inserts that split leaves and removals that shrink them, every namespace operation, and
  fsck's own repairs — the test runs it once to count its writes W, then for every W cuts the power
  there through `CrashDevice` (dropping every later write, tearing the Wth to a subset of its sectors,
  or losing a seeded subset of the writes since the last flush), remounts, and requires that recovery
  succeeds, the checker reports the image clean, and the operation is either wholly there or wholly
  absent.
- **fsck**: every diagnostic the checker can emit is produced by breaking one structure alone on an
  otherwise clean image, then checked for the exact diagnostic, the repair, and a clean second pass;
  recovery is run against logs written by the journal and then damaged on the device; a crash at any
  write of a repair leaves an image the next run finishes.
- **The format**: every structure's size and field offsets, every checksum against the bytes the spec
  gives, geometry at its edges, the directory hash, and thousands of directory inserts and removals
  against a model.
- **The host tools**: the image-file device, populating an image from a directory with owners, and
  each tool's exit codes as the command line runs them.
- **The freestanding build** is the check that the package carries nothing an OS would have to answer.

keel's boot test (`scripts/boot-test.sh` there) mounts KFS in the kernel on four cores and checks it
end to end: an image built by `mkfs.kfs` mounted beside keel's old filesystem, four threads creating,
writing and removing files at once under §19's locks, a power cut after three commits and before their
checkpoint, the replay of exactly those three on the next boot, a clean unmount, a disk without a
flush mounted read-only, and `fsck.kfs -n` clean and `kfsdump get` byte-identical after each boot.
Its controls — sending no flush, writing a pinned buffer home early, freeing inodes still open, never
running the committer — each turn that test red.

## License

ISC — see [LICENSE](LICENSE).
