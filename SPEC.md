# KFS — keel's filesystem

KFS is the filesystem keel mounts as its root. It gives programs the POSIX file model they expect —
hard and symbolic links, device nodes, nanosecond times, unlinking a file that is still open, an atomic
`rename` — and it keeps that model intact across a crash at any instant. It is meant to be read as well
as run: every structure is small enough to hold in one's head, and every rule that exists to survive a
crash says which crash it survives.

The format is defined once, in the shared sysl package `kfs/` (module `keel.kfs`), which is compiled
into the kernel (aarch64-freestanding) and into the host tools (`mkfs.kfs`, `fsck.kfs`, `kfsdump`).
This document is the specification that package implements; where the two disagree, the package is
wrong.

## Contents

1. Goals and non-goals
2. Conventions: byte order, checksums, the device model
3. Constants and limits
4. Disk layout
5. The superblock
6. Group descriptors
7. Bitmaps
8. Inodes
9. The extent tree
10. Directories
11. Symbolic links
12. Device nodes, FIFOs and sockets
13. Extended attributes
14. The journal
15. Recovery
16. The orphan list
17. Allocation
18. Operations: write ordering and crash analysis
19. Concurrency on a multi-core kernel
20. statfs
21. Feature flags and forward compatibility
22. The integrity model
23. fsck
24. Design tradeoffs
25. Changes from SFS, and why
26. Pedagogical notes
27. Implementation plan

---

## 1. Goals and non-goals

**Goals**

- **POSIX semantics** for regular files, directories, hard links, symbolic links, character and block
  devices, FIFOs and sockets: `rename` that atomically replaces, `unlink` of an open file, `O_APPEND`,
  holes, `ftruncate` both ways, `fsync`/`fdatasync`, `atime`/`mtime`/`ctime` with nanoseconds, and a
  creation time.
- **Crash consistency**: after a crash at any write, mounting the disk replays the journal and
  processes the orphan list, and the result is a filesystem `fsck.kfs` finds clean. Metadata is
  journaled; data is written before the metadata that points to it commits (data=ordered).
- **Scale without redesign**: 64-bit block numbers, extent trees with O(log n) lookup, hash-indexed
  directories with O(log n) lookup, allocation groups for locality and for per-group locking on a
  multi-core kernel.
- **Detection of corruption** in every metadata structure, by CRC32C seeded with the filesystem's
  identity and bound to the structure's owner.
- **One team, in stages**: every part can be built and tested on the host over a RAM block device
  before the kernel uses it.

**Non-goals**, each a deliberate tradeoff (section 24 has the reasons):

- Copy-on-write, snapshots, compression, encryption, deduplication.
- Checksums of file data (metadata only).
- Full data journaling (data=journal) and delayed allocation (the format allows the latter; the first
  implementation allocates at write time).
- Online resize, quotas, POSIX ACLs (ACL xattrs are stored but not interpreted), bad-block lists,
  block sizes other than 4 KiB, inline data for small regular files.

---

## 2. Conventions

### Byte order

**Every multi-byte integer on the disk is little-endian.** Both machines that read KFS — aarch64
keel and the aarch64 or x86-64 host running the tools — are little-endian, so the shared package may
read and write structures in place through a pointer into a block, with every structure's size and
every field's offset pinned by `@assert`s. The package also asserts, at compile time, that the target
is little-endian; a big-endian port would read every field through byte-swapping accessors instead,
and nothing on the disk would change. Signed fields (times) are two's complement.

Byte strings (names, labels, UUIDs, symlink targets, hash seeds) are stored as bytes, in order, with no
terminator unless a field says otherwise.

### Checksums: CRC32C

Every checksum is **CRC32C** (Castagnoli: polynomial 0x1EDC6F41, reflected form 0x82F63B78). aarch64
computes it with the `crc32cb/h/w/x` instructions; the host and the kernel share a table-driven
software version, and the kernel may substitute the instructions once they are proven to give the
same answers.

`C(seed, bytes)` names the raw CRC32C register update: start the register at `seed`, feed the bytes,
**no final inversion**. The standard CRC32C of a message `m` is therefore `~C(0xFFFF_FFFF, m)`, which
gives the test vector `~C(0xFFFFFFFF, "123456789") = 0xE3069283`.

The filesystem's checksum seed is `csum_seed = C(0xFFFF_FFFF, uuid)`, stored in the superblock so it
need not be recomputed. Every checksum except the superblock's own starts from `csum_seed` and feeds
the structure's **identity** (its inode number, group number or owner) before its bytes, so that a
structure written to the wrong place, or left over from another filesystem or another owner, fails its
check. Each structure's section gives the exact bytes. A checksum field is never part of the bytes it
covers.

### The device model

KFS sits on a block device of 512-byte sectors and works in 4 KiB blocks of eight sectors (block `b` is
sectors `8b … 8b + 7`). It assumes exactly this of the device, and the crash-injection harness (section
27) models exactly this:

1. **A sector write is atomic.** A 4 KiB block write may **tear**: after a crash any subset of its
   eight sectors may hold the new bytes and the rest the old.
2. **Writes completed since the last flush may be lost in any subset**, in any order — the device may
   have reordered them in a volatile cache.
3. **`flush` returns only when every write completed before it is durable.**

Every structure written in place outside the journal — the superblock, the journal superblock, and
the commit block — keeps all of its meaningful bytes in its **first sector**, so its update is atomic.
Everything else written in place is either journaled first (so a torn home write is repaired by replay)
or is file data (which KFS does not promise to keep whole across a crash without `fsync`).

On keel the device is virtio-blk, which provides a flush only when `VIRTIO_BLK_F_FLUSH` (feature bit 9)
is negotiated (`VIRTIO_BLK_T_FLUSH`, request type 4). The kernel driver must negotiate it; a device
without it is mounted read-only, because without a flush the commit block may reach the disk before
the blocks it commits.

---

## 3. Constants and limits

| Constant | Value | Notes |
|---|---|---|
| `SECTOR_SIZE` | 512 | the device's unit; the unit of atomic writes |
| `BLOCK_SIZE` | 4096 | fixed |
| `INODE_SIZE` | 256 | 16 per block |
| `NAME_MAX` | 255 | bytes in one name; any byte but `/` and NUL |
| `PATH_MAX` | 4096 | bytes, including the terminating NUL; a symlink target is at most 4095 |
| `SYMLINK_INLINE_MAX` | 128 | targets this long or shorter live in the inode |
| `LINK_MAX` | 2,147,483,647 | `EMLINK` beyond |
| `BLOCKS_PER_GROUP` | 2048 … 32768, a power of two | format-time; one bitmap block's bits bound it |
| `INODES_PER_GROUP` | 16 … 32768, a multiple of 16 | format-time |
| `DEFAULT_BYTES_PER_INODE` | 8192 | one inode per 8 KiB of group (section 17) |
| `FIRST_INO` | 11 | inodes 1–10 are reserved |
| `ROOT_INO` | 2 | |
| `EXTENT_ROOT_SLOTS` | 7 | extent entries in the inode |
| `EXTENT_NODE_SLOTS` | 254 | entries in an extent-tree block |
| `EXTENT_MAX_DEPTH` | 4 | levels of extent blocks below the inode |
| `EXTENT_MAX_LEN` | 32768 | blocks in one extent (never more than a group's data area) |
| `DIR_MAX_LEVELS` | 3 | index levels in a hashed directory, the root included |
| `DIR_INDEX_SLOTS` | 254 | entries in a directory index block |
| `JOURNAL_MIN_BLOCKS` | 1024 | 4 MiB |
| `JOURNAL_MAX_BLOCKS` | 16384 | 64 MiB |
| `MAX_OP_CREDITS` | 192 | the largest operation that is never split (section 14) |
| `COMMIT_INTERVAL` | 5 s | the longest a running transaction stays open |

**Maximum sizes.** Block numbers are 64-bit everywhere, so the volume's limit is the group count
(`u32`) times the largest group: 2^32 × 2^15 blocks = 2^47 blocks = **512 PiB**. A file's logical block
numbers are 32-bit: a file holds at most 2^32 blocks, so its **maximum size is 16 TiB** (2^44 bytes);
`write` or `truncate` past that answers `EFBIG`. A 32-bit logical block keeps an extent entry at 16
bytes and is still four orders of magnitude beyond any file keel will hold; widening it later is an
incompat feature (section 21). Inode numbers are 64-bit on the disk and in `stat`. A hashed directory
of three index levels holds at least 254 × 254 × 254 leaves — far beyond any directory keel will hold.

---

## 4. Disk layout

```
block 0                     the boot block — never read or written by KFS
block 1 ...                 group 0
block 1 + B ...             group 1
...
block 1 + (G-1)·B ...       group G-1 (may be shorter)
```

`B` is `blocks_per_group`, `G` is `group_count`. **Group `g` covers blocks `1 + g·B` to
`min(1 + (g+1)·B, blocks) − 1`.** Block 0 belongs to no group; it is left to a boot loader (keel's
virtio test reads sector 1 and writes sector 2 of it).

### Inside a group

Every group begins with its metadata, then its data area:

| offset in group | blocks | what |
|---|---|---|
| 0 | 1 | **superblock copy** — only in a *copy group* (below) |
| 1 | `gdt_blocks` | **group descriptor table copy** — only in a copy group |
| next | 1 | block bitmap |
| next | 1 | inode bitmap |
| next | `inodes_per_group / 16` | inode table |
| rest | | data area |

**The copy groups are group 0, group 1, and group G − 1.** Group 0's copies are the primary superblock
(block 1) and the primary descriptor table (blocks 2 …); groups 1 and G − 1 hold backups. A filesystem
has at least two groups, so it always has at least one backup.

The location of each group's bitmaps and inode table is recorded in its descriptor (section 6) and is
**also computable** from the superblock alone by the rule above; `fsck.kfs` checks that the two agree.
Recording them explicitly lets a future incompat feature place them elsewhere (as ext4's `flex_bg`
does) without changing the descriptor.

**The last group may be shorter.** It is kept if its data area would hold at least 64 blocks;
otherwise `mkfs.kfs` makes the filesystem end where the previous group ends, and the blocks after it
are unused. Bits in the last group's block bitmap for blocks past the end of the filesystem are set.

**The journal** is one contiguous run of `journal_blocks` blocks inside the data area of group
`G / 2`, recorded in the superblock and marked used in that group's bitmap. `mkfs.kfs` chooses
`blocks_per_group` at least `2 × journal_blocks`, so the run always fits.

### Default geometry

`mkfs.kfs` chooses, unless told otherwise:

- `journal_blocks = clamp(blocks / 64, 1024, 16384)`;
- `blocks_per_group` = the largest power of two in [2048, 32768] that yields at least four groups,
  and at least `2 × journal_blocks`; if none yields four groups, 2048;
- `inodes_per_group = blocks_per_group × 4096 / 8192`, rounded down to a multiple of 16
  (`-i bytes-per-inode` overrides, at least 4096);
- `reserved_blocks` = 2% of the data blocks (`-m percent` overrides; may be 0).

`mkfs.kfs` refuses a filesystem that would have fewer than two groups or whose journal would not fit;
a last group that holds the journal is kept only if 64 data blocks remain besides it. With the
defaults the smallest filesystem is 3,205 blocks (about 12.5 MiB): block 0, a full group 0, and a
group 1 of 68 metadata blocks, the 1024-block journal and 64 data blocks. For example, a 32 MiB
image (8192 blocks): `blocks_per_group` 2048, four groups (the last 2047 blocks), 1024 inodes per
group, journal 1024 blocks at the start of group 2's data area.

**Clarifications (stage 2).** "Yields at least four groups" counts groups *after* the last-group rule, so a
dropped fragment does not count. An explicit `blocks_per_group` or journal size is accepted whenever the
journal fits in group `G / 2`'s data area (the `2 × journal_blocks` rule only guides the default).
`reserved_blocks` is the percentage of all groups' data-area blocks, the journal not counted. `mkfs.kfs`
gives the root exactly one directory leaf, at the first data block of group 0, holding the single entry
`lost+found` (inode 11, which itself is empty and has no blocks); root has `links` 3 and `lost+found` 2.

**Pedagogical notes.** The layout is ext2's, minus thirty years. Groups exist for three reasons that
are all visible here: a bitmap block covers exactly one group, so allocation inside a group touches one
block; an inode's data can be kept near its inode; and each group can be locked on its own. The backup
copies are spread across the disk so that one bad region cannot take every copy.

---

## 5. The superblock

The primary superblock is block 1. Its meaningful bytes are the first 512 (one sector, so it is
written atomically); the rest of the block is zero.

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | `magic` | `0x5353_464B`, the bytes "KFSS" |
| 4 | 4 | `version` | 1 |
| 8 | 4 | `block_size` | 4096 |
| 12 | 2 | `state` | bit 0 `CLEAN`, bit 1 `ERROR`; other bits 0 |
| 14 | 2 | `reserved0` | 0 |
| 16 | 8 | `blocks` | the filesystem's end: one past its last block (block 0 counted) |
| 24 | 4 | `group_count` | |
| 28 | 4 | `blocks_per_group` | |
| 32 | 4 | `inodes_per_group` | |
| 36 | 2 | `inode_size` | 256 |
| 38 | 2 | `first_ino` | 11 |
| 40 | 8 | `inode_count` | `group_count × inodes_per_group` |
| 48 | 8 | `free_blocks` | exact only while `CLEAN` is set (below) |
| 56 | 8 | `free_inodes` | exact only while `CLEAN` is set |
| 64 | 8 | `reserved_blocks` | blocks only uid 0 may allocate |
| 72 | 8 | `root_ino` | 2 |
| 80 | 8 | `journal_start` | first block of the journal |
| 88 | 4 | `journal_blocks` | |
| 92 | 4 | `max_transaction` | blocks; `journal_blocks / 4` |
| 96 | 8 | `orphan_head` | the first inode on the orphan list; 0 = empty |
| 104 | 4 | `gdt_blocks` | `ceil(group_count / 64)` |
| 108 | 1 | `hash_algorithm` | 1 = SipHash-1-3 (section 10) |
| 109 | 1 | `csum_algorithm` | 1 = CRC32C |
| 110 | 2 | `reserved1` | 0 |
| 112 | 16 | `hash_seed` | random at format time; the directory hash key |
| 128 | 16 | `uuid` | random at format time |
| 144 | 32 | `label` | UTF-8, NUL-padded |
| 176 | 4 | `feature_compat` | section 21 |
| 180 | 4 | `feature_ro_compat` | |
| 184 | 4 | `feature_incompat` | |
| 188 | 4 | `mount_count` | mounts since the last `fsck.kfs` |
| 192 | 8 | `format_time` | seconds since the epoch |
| 200 | 8 | `mount_time` | |
| 208 | 8 | `write_time` | when the superblock was last written outside the journal |
| 216 | 8 | `check_time` | the last `fsck.kfs` that found or left it clean |
| 224 | 4 | `csum_seed` | `C(0xFFFF_FFFF, uuid)` |
| 228 | 280 | `reserved2` | 0 |
| 508 | 4 | `crc` | `C(0xFFFF_FFFF, bytes 0 … 507)` |

**Two kinds of writes reach the superblock, and they never overlap.**

- **Journaled**: `orphan_head` changes inside transactions (section 16), so block 1 is ordinary
  journaled metadata while the filesystem is mounted.
- **Direct**: `state`, `mount_count`, the times and the free totals are written in place only while
  no transaction is running and the journal is empty — at mount, after recovery and before the first
  transaction; and at unmount, after the last checkpoint.

**`CLEAN`** is cleared, directly, at mount and set, directly, at a clean unmount together with exact
`free_blocks` and `free_inodes`. The authoritative free counts are always the group descriptors'
(journaled); the superblock's totals are a convenience for tools and are exact only when `CLEAN` is
set. Mount recomputes the totals from the descriptors whatever `CLEAN` says. **`ERROR`** means the
kernel detected an inconsistency (a checksum failure, an impossible structure); a filesystem with
`ERROR` set mounts read-only until `fsck.kfs` clears it.

**An error found while mounted is recorded in the journal superblock, not here.** Block 1 is journaled
metadata while mounted, so a direct write of it then could carry an uncommitted `orphan_head` home, and
a later replay of an older journaled copy of block 1 would clear the bit again. Instead the kernel
**aborts the journal** (the running transaction is discarded, nothing more commits, the filesystem
becomes read-only) and sets the journal superblock's `error` field (section 14), a direct write of one
sector that no transaction ever logs. Mount copies a non-zero `error` into `ERROR` after replay (section
15), so the bit survives any crash. Unmount of an aborted filesystem writes `ERROR` directly once the
journal is idle.

**Backups.** The backup superblocks (first block of groups 1 and G − 1) and descriptor tables are
written by `mkfs.kfs` and by any tool that changes a static field (label, features). Their free counts
and times are stale by design; only their static fields are used. If block 1 fails its check,
`fsck.kfs` finds a backup by trying block `1 + b` for each allowed `blocks_per_group` `b` (2048, 4096,
…, 32768) and taking the first that passes; the kernel refuses to mount (it does not guess).

---

## 6. Group descriptors

The descriptor table is `gdt_blocks` blocks of 64-byte descriptors, 64 per block, descriptor `g` at
offset `(g % 64) × 64` of table block `g / 64`. The primary table follows the primary superblock.

| offset | size | field | notes |
|---|---|---|---|
| 0 | 8 | `block_bitmap` | block number |
| 8 | 8 | `inode_bitmap` | block number |
| 16 | 8 | `inode_table` | first block |
| 24 | 4 | `free_blocks` | in this group |
| 28 | 4 | `free_inodes` | |
| 32 | 4 | `directories` | inodes in this group that are directories |
| 36 | 4 | `flags` | bit 0 `COPY_GROUP` (holds a superblock and table copy); others 0 |
| 40 | 4 | `block_bitmap_crc` | `C(csum_seed, group as u32 LE ‖ the block bitmap)` |
| 44 | 4 | `inode_bitmap_crc` | `C(csum_seed, group as u32 LE ‖ the inode bitmap)` |
| 48 | 12 | `reserved` | 0 |
| 60 | 4 | `crc` | `C(csum_seed, group as u32 LE ‖ bytes 0 … 59)` |

Table entries past `group_count` in the last table block are zero and unchecked.

**A bitmap's checksum covers the whole 4096-byte bitmap block**, not just the bits its group uses: every
bit past the group's last block (block bitmap) or last inode (inode bitmap) is set, so no byte of the
block is unspecified.

**The descriptor is where the bitmaps' checksums live**, so a bitmap needs no room of its own for one;
every change to a bitmap changes a free count anyway, so the descriptor is in the same transaction.

**Pedagogical note.** Free counts live in the descriptors, journaled with the bitmap they summarise,
so they are always exact after recovery. The superblock's totals would make every allocation on every
core write one shared block; instead they are derived (section 20).

---

## 7. Bitmaps

**Block bitmap**: bit `i` of the group's block bitmap describes block `group_first + i`: set while the
block is in use. Bit `i` is bit `i % 8` of byte `i / 8`. Every metadata block of the group (superblock
copy, table copy, both bitmaps, the inode table), every journal block in it, and every block past the
end of the filesystem is set.

**Inode bitmap**: bit `i` describes inode `g × inodes_per_group + i + 1`: set while it is allocated.
Bits past `inodes_per_group` are set. In group 0 the bits for inodes 1–10 (reserved) are set.

A block freed by a transaction is not free for allocation until that transaction commits (section 17).
The on-disk bitmap shows it free as soon as the freeing transaction commits.

---

## 8. Inodes

### Numbering

Inodes are numbered from 1; 0 means "none". Inode `n` is in group `(n − 1) / inodes_per_group` at index
`i = (n − 1) % inodes_per_group`, in block `inode_table + i / 16` at offset `(i % 16) × 256`.

| inode | use |
|---|---|
| 1 | reserved, never allocated (so an off-by-one is never the root) |
| 2 | the root directory |
| 3–10 | reserved for future use; allocated, mode 0 |
| 11 … | ordinary; `mkfs.kfs` makes `/lost+found` the first |

### Layout (256 bytes)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 2 | `mode` | POSIX `st_mode`: type and permission bits (below) |
| 2 | 2 | `reserved0` | 0 |
| 4 | 4 | `links` | names referring to it; a directory counts its own name, its synthesized `.`, and its subdirectories' `..` |
| 8 | 4 | `uid` | |
| 12 | 4 | `gid` | |
| 16 | 4 | `flags` | below |
| 20 | 4 | `generation` | incremented each time the inode is allocated |
| 24 | 8 | `size` | bytes |
| 32 | 8 | `blocks` | 4 KiB blocks held: data, extent-tree nodes, and the xattr block; `st_blocks` is 8 × this |
| 40 | 8 | `atime_sec` | signed seconds since the epoch |
| 48 | 8 | `mtime_sec` | |
| 56 | 8 | `ctime_sec` | change of the inode itself |
| 64 | 8 | `crtime_sec` | creation; never changes |
| 72 | 4 | `atime_nsec` | 0 … 999,999,999 |
| 76 | 4 | `mtime_nsec` | |
| 80 | 4 | `ctime_nsec` | |
| 84 | 4 | `crtime_nsec` | |
| 88 | 8 | `orphan_next` | next inode on the orphan list; 0 = last (section 16) |
| 96 | 8 | `xattr_block` | 0 = none (section 13) |
| 104 | 8 | `parent` | for a directory, its parent directory (the root's is itself); otherwise 0 |
| 112 | 128 | `data` | a union selected by type and flags (below) |
| 240 | 12 | `reserved1` | 0 |
| 252 | 4 | `crc` | `C(csum_seed, inode number as u64 LE ‖ bytes 0 … 251)` |

**Mode.** The type is one of `S_IFREG 0o100000`, `S_IFDIR 0o040000`, `S_IFLNK 0o120000`,
`S_IFCHR 0o020000`, `S_IFBLK 0o060000`, `S_IFIFO 0o010000`, `S_IFSOCK 0o140000`; below it set-user-ID
`0o4000`, set-group-ID `0o2000`, sticky `0o1000`, and `rwx` for owner, group and other. Any other value
is corrupt. A free inode has mode 0.

**Flags.**

| bit | name | meaning |
|---|---|---|
| 0 | `INLINE` | a symlink whose target is in `data` |
| 1 | `INDEXED` | a directory whose block 0 is a hash-index root |
| 2 | `ORPHAN` | the inode is on the orphan list |
| 3–31 | | reserved, 0 |

**The `data` union.**

| inode | `data` holds |
|---|---|
| regular file, directory, symlink without `INLINE` | the extent-tree root (section 9) |
| symlink with `INLINE` | the target, `size` bytes, the rest zero |
| character or block device | `major: u32` at 0, `minor: u32` at 4, the rest zero |
| FIFO, socket | zero |

**Generation.** When an inode is freed, its slot is zeroed except `generation`, which survives; the
next allocation of the slot increments it. Extent-tree blocks, directory blocks and xattr blocks carry
their owner's number *and generation* (section 9's tail), so a block left over from a previous
occupant of the same inode number fails its check.

**Times.** 64-bit signed seconds plus nanoseconds: there is no 2038 problem and no 2106 problem. A
time before 1970 is a negative `_sec` with a non-negative `_nsec`.

**Invariants** (`fsck.kfs` checks each):

- `links` is 0 only for an inode on the orphan list (`ORPHAN` set).
- No *written* extent maps a block at or beyond `ceil(size / 4096)`; only unwritten (preallocated)
  extents may lie past the end of the file, and only for a regular file.
- **The tail of the last block is zero**: bytes from `size` to the end of the block holding byte
  `size − 1` read as zero on the disk. Truncation zeroes them (section 18); writes never put bytes
  there.
- A device, FIFO or socket has `size` 0 and `blocks` 0.

---

## 9. The extent tree

A regular file's, directory's or long symlink's blocks are mapped by an **extent tree** rooted in the
inode's `data` field. An **extent** maps a run of logical blocks to a run of physical blocks. A logical
block that no extent maps is a **hole**: it reads as zeros and holds no space. There is no "sparse"
extent; absence is the hole.

### Entries (16 bytes)

**Leaf entry** — an extent:

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | `logical` | first logical block |
| 4 | 2 | `len` | blocks, 1 … 32768; 0 is corrupt |
| 6 | 2 | `flags` | bit 0 `UNWRITTEN`: allocated, reads as zeros; others 0 |
| 8 | 8 | `physical` | first physical block |

An extent lies wholly inside one group's data area (the group metadata between data areas guarantees
this), which is why 16 bits of length are enough.

**Index entry** — a pointer to a child node:

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | `logical` | the lowest logical block the child may map |
| 4 | 4 | `reserved` | 0 |
| 8 | 8 | `child` | block number of the child node |

### Node header (16 bytes)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | `magic` | `0x5853_464B`, "KFSX" |
| 4 | 2 | `count` | entries in use |
| 6 | 2 | `max` | capacity of this node: ≤ 7 in the inode, ≤ 254 in a block |
| 8 | 2 | `depth` | 0 = entries are extents; *d* > 0 = entries point to nodes of depth *d* − 1 |
| 10 | 6 | `reserved` | 0 |

**The root** is the inode's 128-byte `data`: the header, then up to 7 entries. Its checksum is the
inode's.

**A node block** is the header, up to 254 entries (offsets 16 … 4079), and a **block tail**:

### Block tail (16 bytes, the last 16 bytes of every owned metadata block)

| offset from block start | size | field |
|---|---|---|
| 4080 | 8 | `owner` — the inode that holds this block |
| 4088 | 4 | `generation` — the owner's generation |
| 4092 | 4 | `crc` — `C(csum_seed, bytes 0 … 4091)` |

Extent-tree nodes, directory blocks and xattr blocks all end in this tail. `owner` and `generation`
are inside the checksummed bytes, so a valid block from another inode — or from a previous life of
this one — fails its check.

### Rules

- Entries in a node are sorted by `logical`, strictly increasing; extents do not overlap.
- Every entry of the child of index entry *i* maps logical blocks in `[logical_i, logical_{i+1})` (the
  last entry's range is open above).
- All leaves are at the same depth; the root's `depth` is the tree's height, at most
  `EXTENT_MAX_DEPTH`. A tree of depth 4 with full nodes maps 7 × 254⁴ ≈ 2.9 × 10¹⁰ extents, more than
  the 2³² blocks a file can have, so no file can outgrow it.
- **`max` may be smaller than the capacity.** Implementations write the capacity; the host tests
  format trees with small `max` so that depth-4 trees are exercised on a RAM disk. `fsck.kfs` accepts
  any `max` from 1 to the capacity.
- Adjacent extents that are contiguous both logically and physically, with equal flags, may be merged;
  the format does not require it.

### Operations

- **Lookup** of logical block `l`: from the root, at each index level take the last entry with
  `logical ≤ l`; at the leaf, the last extent with `logical ≤ l`, which maps `l` iff
  `l < logical + len`. O(depth × log fan-out).
- **Insert** an extent: descend as for lookup; if the leaf is full, split it into two halves, insert
  the new node's first key into the parent, and so on up; if the root is full, move its entries into a
  new block and make the root one index entry pointing to it (the tree grows from the top, so all
  leaves stay at one depth).
- **Remove a range** `[a, b)`: extents wholly inside are removed; an extent straddling `a` or `b` is
  shortened, and one straddling both is split in two (which is an insert). An empty node is freed and
  its entry removed from the parent; a root left with one index entry whose child fits in the root
  absorbs the child and the tree shrinks.
- **Unwritten → written**: writing into an unwritten extent splits it into at most three (unwritten,
  written, unwritten) after the data is on the disk (section 18).

**Pedagogical notes.** An extent turns "one pointer per block" into "one pointer per run", and the tree
turns "a flat list of runs" into O(log n) lookup. Holes cost nothing because they are not there. The
root inside the inode means a file of up to seven runs needs no extra block at all — most files.

---

## 10. Directories

A directory is a file of 4 KiB **directory blocks**, mapped by its extent tree. It is either
**linear** — one leaf at block 0 — or **indexed** (`INDEXED` set) — block 0 is the root of a hash
index whose leaves hold the entries. An **empty directory has no blocks at all** (`size` 0).

**`.` and `..` are not stored.** `readdir` synthesizes `.` (the directory itself) and `..` (its
`parent` field). Lookup answers `.` and `..` from the inode without reading a block. Rename of a
directory changes the moved directory's `parent` field — an inode it already rewrites for `ctime` —
rather than a block inside it.

### The hash

`hash(name) = clamp(S(name) & ~1, 8, 2⁶⁴ − 4)`, where `S` is **SipHash-1-3** with output 64 bits, keyed
by the superblock's `hash_seed` (k0 = bytes 0–7 LE, k1 = bytes 8–15 LE), over the name's bytes. Every
hash is therefore even and lies in [8, 2⁶⁴ − 4].

- **Keyed**: the seed is random per filesystem, so no one can choose names that collide on a disk
  they did not format; collisions are only chance, about one pair in 2⁶³.
- **Bit 0 is free** for the index's continuation mark (below).
- **The clamp reserves directory positions** 0–3 and 2⁶³ − 1 (below).

`hash_algorithm` 1 names this function; no other value is defined.

### Leaf block

| offset | size | field |
|---|---|---|
| 0 | 4 | `magic` — `0x4453_464B`, "KFSD" |
| 4 | 2 | `count` — live entries |
| 6 | 2 | `reserved` — 0 |
| 8 | 4072 | records, packed |
| 4080 | 16 | block tail (section 9) |

A **record**:

| offset | size | field | notes |
|---|---|---|---|
| 0 | 8 | `ino` | 0 = a free record |
| 8 | 2 | `rec_len` | bytes to the next record; a multiple of 8, at least 16 |
| 10 | 1 | `name_len` | 1 … 255 |
| 11 | 1 | `file_type` | `DT_*` (below) |
| 12 | `name_len` | `name` | any bytes but `/` and NUL; never `.` or `..` |

Records tile the area exactly: the first starts at 8 and the last ends at 4080. A record's live bytes
are `12 + name_len` rounded up to 8; the rest of its `rec_len` is slack. A removed record's space is
added to the previous record's `rec_len`; if it is the first record, its `ino` becomes 0. An empty leaf
is one free record of `rec_len` 4072. Records are not sorted on the disk.

**`file_type`** repeats the inode's type, Linux's `DT_*` numbering, so `getdents64` needs no inode
read:

| value | name | type |
|---|---|---|
| 0 | `DT_UNKNOWN` | never written |
| 1 | `DT_FIFO` | FIFO |
| 2 | `DT_CHR` | character device |
| 4 | `DT_DIR` | directory |
| 6 | `DT_BLK` | block device |
| 8 | `DT_REG` | regular file |
| 10 | `DT_LNK` | symbolic link |
| 12 | `DT_SOCK` | socket |

### Index block

Block 0 of an indexed directory is the **root**; deeper index blocks are **interior**. Both:

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | `magic` | `0x4853_464B`, "KFSH" |
| 4 | 2 | `count` | entries |
| 6 | 2 | `max` | capacity, ≤ 254 (tests may use less) |
| 8 | 1 | `level` | 0 = children are leaves; *k* = children are index blocks of level *k* − 1 |
| 9 | 7 | `reserved` | 0 |
| 16 | 16 × n | entries | |
| 4080 | 16 | block tail | |

An **index entry** is `key: u64` at 0, `child: u32` (a logical block of this directory) at 8,
`reserved: u32` at 12. Entries are sorted by `key`, strictly increasing as unsigned 64-bit numbers.
**The root's entry 0 has key 0; an interior block's entry 0 has the same key as the entry in its
parent that points to it**, so every leaf's key is simply its entry's key in its own parent. The
root's `level` + 1 is the number of index levels, at most `DIR_MAX_LEVELS`.

**A new directory block** takes the lowest logical block that is a hole in the directory, else the
block just past its end.

### Keys, ranges and the continuation mark

A key is a hash with bit 0 as a mark: `key = h` or `key = h | 1`. Child *i* of an index block holds
entries whose hashes lie in `[key_i & ~1, key_{i+1} & ~1]` — closed above **only** when `key_{i+1}` has
bit 0 set, and open above otherwise. **`key_{i+1} = h | 1` means: the run of entries with hash `h`
began in child *i* (or earlier) and continues into child *i* + 1.** This is the one way equal hashes
can span a leaf split, and it is precise:

- **Probe** for hash `h` (even): at each index block take the last entry with `key ≤ h`, comparing all
  64 bits. Because `h | 1 > h`, a continuation entry is never chosen; the probe lands on the **first**
  leaf that can hold `h`.
- **Lookup** scans that leaf for the name; if absent, and the **next leaf in key order** has key
  `h | 1`, it scans that leaf too, and so on. "Next leaf in key order" is the in-order successor in the
  tree, which may sit under a different index block: the probe keeps its path to find it.
- **Insert** probes for `h`; if the name exists, `EEXIST`. It inserts into the probed leaf if the
  record fits (in a free record, in slack, or by splitting slack); else into the next leaf of the
  same run (key `h | 1`) that has room; else it splits the last leaf it visited.
- **Split** a leaf: take its live entries plus the new one, ordered by hash. Let *s* be the first entry
  whose cumulative live bytes reach half. If entries *s* − 1 and *s* have different hashes, split
  there: the new leaf gets entries *s* … and its key is `hash_s`. Otherwise move *s* to the nearest
  index (fewest entries moved, ties toward the end) where adjacent hashes differ and both halves fit; if
  there is none (every entry has one hash), split at *s* and the new key is `hash_s | 1`. The lower
  half stays in place; the new leaf's entry goes immediately after the old leaf's in its parent.
- **Split** an index block: halves by count; the new block's key in the parent is its first entry's
  key, mark and all (which its entry 0 keeps). A full root moves its entries into a new index block and becomes a single entry
  pointing to it, one level higher; past `DIR_MAX_LEVELS`, insert answers `ENOSPC`.

**Converting** a linear directory: when an entry will not fit in block 0, two new leaves are
allocated, the entries (and the new one) are split between them by the rule above, block 0 is
rewritten as a root of level 0 with two entries, and `INDEXED` is set — all in one transaction.

### Shrinking

Removing an entry may shrink the directory, in the same transaction:

1. **A leaf left empty** is freed (its block becomes a hole, or the directory's size shrinks if it was
   the last block) and its entry leaves its parent: entry *i* > 0 is simply removed — its range joins
   child *i* − 1; when entry 0 is removed, entry 1 takes entry 0's key (and, if entry 1's child is an
   index block, so does that block's entry 0, down to the leaf). An index block left with no entries
   is removed from its parent the same way.
2. **A leaf and its right sibling** (same parent) whose live bytes together fit in half a leaf are
   merged: the sibling's records move into the leaf and the sibling is freed as in 1.
3. **A root with one entry** whose child is an index block absorbs that child (one level fewer).
4. **The last entry removed** from a directory frees every block it holds; it becomes empty (no blocks,
   `INDEXED` clear, `size` 0).

Holes in a directory are never visited: lookup and `readdir` follow the index, not block order.

### readdir positions

`getdents64` reports, and `lseek` accepts, a **position** per entry: `.` is at 1, `..` at 2, and an
entry with hash `h` at `h >> 1`, which lies in [4, 2⁶³ − 2]; the end of the directory is 2⁶³ − 1
(position 0 is the start, 3 is unused). `readdir` from position *p* returns, in hash order, every
entry whose position is ≥ *p* (leaves in key order, each leaf's records sorted in memory), and the
`d_off` of an entry is the position of the entry after it.

Because positions are hashes, they survive any insertion, removal, split or merge: an entry present
for a whole scan is returned **exactly once**; one added or removed during it may or may not be. Two
names with one hash share a position, so **one `getdents64` call never ends between them**; if their
whole run does not fit the caller's buffer the call answers `EINVAL` (with a keyed 64-bit hash, a run
longer than one is a chance of 2⁻⁶³ per pair).

**Pedagogical notes.** Linear search is fine for a few dozen names and hopeless for a million; the
hash index turns lookup into one block per level. Hashing makes a directory order-free, which is why
`readdir` positions are hashes rather than byte offsets. Keying the hash closes the denial-of-service
door that a fixed hash (FNV-1a, half-MD4) leaves open: with a fixed hash, an attacker can make every
name in a directory collide.

---

## 11. Symbolic links

A symlink's target is `size` bytes, 1 to 4095, with no terminator. A target of at most 128 bytes lives
in the inode's `data` (`INLINE` set, no blocks); a longer one in one data block (`INLINE` clear, one
extent of one block, the bytes past `size` zero). The mode is `S_IFLNK | 0o777`; permission bits are
ignored. `readlink` returns the bytes; path resolution follows at most 40 links (`ELOOP`).

---

## 12. Device nodes, FIFOs and sockets

A device node, FIFO or socket is an inode with no blocks; only its type and, for a device, its numbers
mean anything to the filesystem.

- **Character and block devices** store `major` and `minor` (each `u32`) in `data`. `stat` answers
  `st_rdev` in Linux's encoding:
  `(minor & 0xff) | ((major & 0xfff) << 8) | ((minor & ~0xff) << 12) | ((major & ~0xfff) << 32)`.
  Opening one hands the open to the kernel's device table, keyed by `(major, minor)`; the filesystem's
  part ends at lookup and permission checks.
- **FIFOs**: opening one attaches to an in-memory pipe keyed by the inode; nothing is written to the
  disk by I/O on it.
- **Sockets**: the inode is the name a socket is bound to; the filesystem stores nothing more.

keel's `/dev` uses Linux's numbers:

| name | type, major, minor | mode |
|---|---|---|
| `/dev/null` | c 1 3 | 0666 |
| `/dev/zero` | c 1 5 | 0666 |
| `/dev/full` | c 1 7 | 0666 |
| `/dev/random` | c 1 8 | 0666 |
| `/dev/urandom` | c 1 9 | 0666 |
| `/dev/tty` | c 5 0 | 0666 |
| `/dev/console` | c 5 1 | 0600 |

`mknodat` creates any of the four types (devices require uid 0, as on Linux).

---

## 13. Extended attributes

**Defined, minimal.** An inode may have one **xattr block** (`xattr_block`), which it owns alone (no
sharing between inodes) and which counts in its `blocks`. There are no in-inode attributes.

| offset | size | field |
|---|---|---|
| 0 | 4 | `magic` — `0x4153_464B`, "KFSA" |
| 4 | 2 | `count` |
| 6 | 2 | `used` — bytes of records |
| 8 | … | records, packed, sorted by (`namespace`, `name`) |
| 4080 | 16 | block tail |

A record: `namespace: u8` (1 `user.`, 2 `trusted.`, 3 `security.`, 4 `system.`), `name_len: u8`
(1 … 255, the name without its prefix), `value_len: u16`, then the name, then the value, padded to 4
bytes. All attributes of an inode together are at most 4072 bytes; a `setxattr` that would exceed it
answers `ENOSPC`. Removing the last attribute frees the block. `system.posix_acl_*` values are stored
and returned but not enforced.

Xattrs are the ro_compat feature `XATTR` (section 21): set by the first `setxattr`, so a driver that
does not know them can read the filesystem but must not write it (it would leak xattr blocks when
freeing inodes).

---

## 14. The journal

KFS journals **metadata** — superblock, descriptors, bitmaps, inode-table blocks, extent nodes,
directory blocks, xattr blocks — and writes **data** to its home location **before** the commit of the
transaction that makes metadata point to it (data=ordered). After a crash, metadata is exactly as of
some committed transaction, and no metadata points at a block whose data was never written.

### Transactions and handles

- Operations run inside **handles**. A handle is opened with a **credit** count: an upper bound on the
  number of metadata blocks it may add to the transaction. Opening a handle may wait (below).
- At most one transaction is **running** — the one new handles join — and at most one is
  **committing**. Every handle belongs to the running transaction it joined.
- A transaction closes to new handles when its reserved credits would exceed `max_transaction` minus
  its overhead (descriptor, revoke and commit blocks), when it has been open `COMMIT_INTERVAL`, or when
  `fsync`/`sync`/unmount asks; it commits when its last handle closes.
- **`max_transaction = journal_blocks / 4`**, so the log always has room for the committing
  transaction plus the next, and checkpointing the oldest transaction always frees enough room to
  proceed. `mkfs.kfs` refuses a journal whose `max_transaction` is below `MAX_OP_CREDITS` plus
  overhead (`JOURNAL_MIN_BLOCKS` guarantees it).

### Credits

The unit costs, with `D_X = EXTENT_MAX_DEPTH = 4` and `D_H = DIR_MAX_LEVELS = 3`:

| name | blocks | what it covers |
|---|---|---|
| `C_INODE` | 1 | one inode-table block |
| `C_GROUP` | 2 | one group's bitmap and its descriptor-table block |
| `C_EXT` | 4 × (D_X + 1) = 20 | one extent insertion or split, with every node it may split or add and their groups |
| `C_ALLOC` | `C_GROUP + C_EXT` = 22 | allocate one physical run and map it |
| `C_DIR_ADD` | D_H + 1 + (D_H + 1) × `C_ALLOC` = 92 | insert a name: index path, leaf, and up to D_H + 1 new blocks |
| `C_DIR_DEL` | D_H + 2 + (D_H + 1) × (`C_GROUP` + `C_EXT`) + 1 = 94 | remove a name, with merges and freed blocks and their revoke |
| `C_SB` | 1 | the superblock (orphan list head) |

Per operation (section 18 says what each writes):

| operation | credits |
|---|---|
| create, `mknodat`, `mkdirat`, inline `symlinkat` | `C_GROUP + 2·C_INODE + C_DIR_ADD` = 96 |
| `symlinkat` with a block | 96 + `C_ALLOC` = 118 |
| `linkat` | `2·C_INODE + C_DIR_ADD` = 94 |
| `unlinkat`, `rmdir` | `3·C_INODE + C_DIR_DEL + C_SB` = 98 |
| `renameat2` (any form) | `4·C_INODE + C_DIR_ADD + C_DIR_DEL + C_SB` = 191 |
| one write chunk (≤ 8 extents allocated) | `C_INODE + 8·C_ALLOC` = 177 |
| one truncate step (≤ 16 extents, ≤ 4 nodes freed) | ≤ 64 |
| one punch step (a truncate step that may split one extent) | ≤ 64 + `C_EXT` + `C_GROUP` = 86 |
| `setattr` (mode, owner, times, size up) | `C_INODE` = 1 |
| `setxattr`/`removexattr` | `C_INODE + C_GROUP + 2` = 5 |

**Credits are checked, not trusted.** A handle counts the distinct buffers it adds to the
transaction; adding one past its credits is a bug in the credit table and stops the kernel (and fails
the host test) at once, naming the operation. The tests of every stage run with this check, so a
credit undercount shows up as a failure instead of as a transaction that overflows the log.

`MAX_OP_CREDITS` = 192 bounds every operation that is never split. **Operations whose worst case is
unbounded** — writing a large buffer, truncating or deleting a large fragmented file, preallocating a
large range — are split into steps of bounded credits, each its own handle; section 18 says how each is
made crash-safe across its steps.

### Region and log layout

The journal is `journal_blocks` blocks at `journal_start`. Journal block 0 is the **journal
superblock**; blocks 1 … `journal_blocks − 1` are a circular **log**. Every log block written as a
header begins with the 16-byte **journal header**:

| offset | size | field |
|---|---|---|
| 0 | 4 | `magic` — `0x4A53_464B`, "KFSJ" |
| 4 | 4 | `type` — 1 journal superblock, 2 descriptor, 3 revoke, 4 commit |
| 8 | 8 | `sequence` — the transaction's number (0 in the journal superblock) |

Sequence numbers are 64-bit and increase by one per transaction; they never wrap.

### Journal superblock (journal block 0; meaningful bytes in its first sector)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 16 | header | type 1 |
| 16 | 4 | `journal_blocks` | must equal the superblock's |
| 20 | 4 | `first` | 1 — the first log block |
| 24 | 4 | `start` | log block of the oldest transaction not yet checkpointed; **0 = the log is empty** |
| 28 | 4 | `error` | 0, or the first error the kernel found while mounted (an `errno` value, e.g. `EIO` 5); section 5 |
| 32 | 8 | `start_seq` | the sequence at `start`; when empty, the next sequence to use |
| 40 | 16 | `uuid` | the filesystem's |
| 56 | 4 | `max_transaction` | must equal the superblock's |
| 60 | 4 | `features` | 0 |
| 64 | 444 | `reserved1` | 0 |
| 508 | 4 | `crc` | `C(csum_seed, bytes 0 … 507)` |

### A transaction in the log

```
descriptor  copy  copy  …  [descriptor  copy …]  [revoke …]  commit
```

in consecutive log blocks, wrapping from the last log block to block 1.

**Descriptor block**:

| offset | size | field |
|---|---|---|
| 0 | 16 | header, type 2 |
| 16 | 4 | `count` — tags in this block, 1 … 254 |
| 20 | 4 | `reserved` |
| 24 | 16 × count | tags |
| 4092 | 4 | `crc` — `C(csum_seed, bytes 0 … 4091)` |

A **tag** describes the copy that follows the descriptor in the same position as the tag:

| offset | size | field |
|---|---|---|
| 0 | 8 | `block` — the home block number |
| 8 | 4 | `flags` — bit 0 `ESCAPED` |
| 12 | 4 | `crc` — `C(csum_seed, sequence as u64 LE ‖ the copy as logged)` |

**Escaping.** If a copy's first four bytes equal the journal magic, they are logged as zero and the tag
is marked `ESCAPED`; replay restores them. A log block that begins with the magic is then always a
header, so a reader that has lost its place (the dumper, `fsck.kfs`'s log audit) can resynchronise.

**Revoke block**:

| offset | size | field |
|---|---|---|
| 0 | 16 | header, type 3 |
| 16 | 4 | `count` — 1 … 508 |
| 20 | 4 | `reserved` |
| 24 | 8 × count | home block numbers revoked |
| 4092 | 4 | `crc` — `C(csum_seed, bytes 0 … 4091)` |

**Commit block** (meaningful bytes in its first sector, so it is written atomically):

| offset | size | field |
|---|---|---|
| 0 | 16 | header, type 4 |
| 16 | 8 | `commit_sec` |
| 24 | 4 | `commit_nsec` |
| 28 | 4 | `blocks` — log blocks of this transaction before the commit block |
| 32 | 476 | reserved, 0 |
| 508 | 4 | `crc` — `C(csum_seed, bytes 0 … 507)` |

### Revocation

**A metadata block freed in transaction T gets a revoke record in T.** Without one, this happens: a
directory block *b* is journaled in T1; the directory is removed in T2 and *b* freed; *b* is
reallocated as file data in T3 and the data written straight to *b*; the system crashes before T1 has
left the log. Replay writes T1's copy of the directory block over the file's data. With the revoke,
replay skips every copy of *b* in a transaction no newer than T2.

- **Journaling a block cancels its revoke in the same transaction** (a block freed and reused as
  metadata in one transaction is logged, not revoked).
- A revoke record is only needed while some committed transaction still in the log holds a copy of
  the block; an implementation may simply revoke every freed metadata block.
- A revoke lives in a transaction newer than every copy it cancels, and the log is checkpointed oldest
  first, so a revoke leaves the log after the copies it cancels — never before.
- Data blocks never appear in the log and are never revoked.

### Commit (the write ordering)

1. **Lock** T: no handle may join it, and a handle being opened **waits** (it joins no transaction
   yet); wait until T has no open handle. Then open T + 1 to the waiting handles.
2. **Ordered data**: write every dirty data block of every inode on T's ordered list (inodes whose
   size or mapping T changes); wait for completion.
3. **Freeze on write**: from step 1 on, the first time a handle of T + 1 asks to modify a buffer that
   belongs to T, a **frozen copy** of the buffer's bytes is taken for T (under the buffer's lock)
   before the change. Without this, T's log would carry T + 1's half-done changes.
4. Write T's descriptor blocks, escaped copies (the frozen copy where one exists, the live buffer
   otherwise) and revoke blocks to the log; wait for completion.
5. **Flush.**
6. Write the commit block; wait; **flush**.
7. T is committed: blocks T freed become allocatable (section 17); T's buffers join T's checkpoint list
   unless a newer transaction has modified them since (then they join that one's, when it commits).

The flush in step 5 is what makes the commit block a promise: without it the device could make the
commit durable before the descriptors and data it vouches for. The flush in step 6 makes the commit
itself durable before `fsync` returns.

### Checkpoint

Checkpointing writes committed metadata to its home locations so that log space can be reused:

1. For the oldest committed transaction(s): write each buffer on its checkpoint list to its home
   block. **A buffer that a running or committing transaction has modified is never written home**;
   if the log space is needed, that transaction is committed first, after which the buffer belongs to
   its checkpoint list.
2. Wait; **flush**.
3. Set the journal superblock's `start` and `start_seq` to the oldest transaction still in the log
   (`start = 0` if none); write it; flush.

**Log space is reserved when a handle joins, never when a transaction commits.** A handle may join
the running transaction only if the log's free space (between the head and `start`) covers the
committing transaction (if any) plus the running transaction's reserved credits and overhead
including this handle's. Otherwise the opener — which belongs to no transaction yet — waits while the
committer checkpoints the oldest transactions, committing the running transaction first if a buffer
it must write home is held by it (that commit needs no new space: it was reserved). So a commit never
waits for space, and nothing in the cycle "commit needs a checkpoint needs a commit" can close. A
transaction never exceeds `max_transaction`, a quarter of the log, so checkpointing every older
transaction always makes room. Unmount commits, checkpoints everything, and leaves `start = 0`.

**The buffer-cache rule that follows**: a metadata buffer modified by a transaction that has not
committed is **pinned** — it may not be written to its home block or evicted. A committed,
not-yet-checkpointed buffer may be written home at any time (that is a checkpoint write) and then
evicted.

**Pedagogical notes.** The commit block is the whole trick of write-ahead logging: everything before
it is a proposal, and its presence makes the proposal law. Revoke records are the subtle part — a log
replays the past, and some of the past must stay dead. Sequence numbers in every block are what tell
this lap of the circular log from the last.

---

## 15. Recovery

Mount runs recovery when the journal superblock's `start` is not 0. `fsck.kfs` runs the same code.

1. **Read the journal superblock**; check its magic, type, checksum, `uuid` and sizes. Any failure:
   refuse to mount (`fsck.kfs` can rebuild an empty journal, losing the log).
2. **Scan.** From `pos = start`, `seq = start_seq`: read the block at `pos`. If it is a descriptor of
   sequence `seq` with a valid checksum, skip its `count` copies; if a revoke block of `seq` with a
   valid checksum, continue; if a commit block of `seq` with a valid checksum and `blocks` equal to the
   blocks seen since the transaction began, transaction `seq` is complete — `seq += 1`. Anything else
   ends the scan. Positions advance modulo the log (block `journal_blocks − 1` is followed by block 1).
   The last complete transaction is `end_seq − 1`; an incomplete one after it is ignored.
3. **Revoke pass.** For every revoke record in transactions `start_seq … end_seq − 1`, record
   `revoked[b] = max(revoked[b], seq)`.
4. **Replay pass.** For every transaction in order, for every tag: if `revoked[b] ≥ seq`, skip it;
   otherwise read the copy, check the tag's `crc` (a mismatch in a committed transaction is media
   corruption: stop, set `ERROR`, refuse a read-write mount), unescape it, and write it to block `b`.
5. **Flush.** Then set `start = 0`, `start_seq = end_seq`; write the journal superblock; flush.
6. **Load** the superblock and descriptors afresh (replay may have rewritten them) and recompute the
   free totals. If the journal superblock's `error` is non-zero, set `ERROR` in the superblock (a direct
   write, now safe: the log is empty) and clear `error`.

Recovery is **idempotent**: a crash during it leaves the journal superblock unchanged until step 5, and
replaying a copy twice writes the same bytes twice.

**Orphan processing is not part of recovery.** Mount runs it after its own direct superblock write
(section 18, "Mount and unmount"), because it runs in ordinary transactions and direct writes of block 1
are allowed only while the log is empty. A crash during it is an ordinary crash: the list is still on
the disk and the next mount resumes it.

---

## 16. The orphan list

An **orphan** is an inode that must have work finished on it even if the system crashes now:

- **links = 0** — unlinked (or renamed over, or `rmdir`ed) and not yet freed, typically because it is
  still open; its blocks and slot are freed when the last reference goes, or at the next mount;
- **links > 0** — a **truncation in progress**: its `size` is already the new size, and blocks past it
  are still being freed in steps.

The list is singly linked through the disk: the superblock's `orphan_head` names the first, each
member's `orphan_next` the next, the last has 0. Every member has `ORPHAN` set; no non-member has it.

- **Add** (in the transaction that makes the inode an orphan): `inode.orphan_next = orphan_head;
  orphan_head = inode; set ORPHAN`. Cost: the inode and the superblock.
- **Remove** (in the transaction that finishes the work): the predecessor's `orphan_next` (or
  `orphan_head`) takes the inode's `orphan_next`; `ORPHAN` is cleared. The kernel keeps the list
  doubly linked in memory, built at mount, so finding the predecessor is O(1).
- **At mount**, after journal replay, walk the list (at most `inode_count` steps; a cycle or an
  unallocated member is corruption): an inode with links 0 is truncated to 0 in steps and freed; one
  with links > 0 has every block past `ceil(size / 4096)` freed in steps; each is then removed.

**Pedagogical note.** The orphan list is the answer to "Unix lets you delete an open file — what does
the disk do?" The name goes at once; the inode lives on the list until nobody holds it. The same list
makes any multi-step job restartable: record the intent in the first transaction, finish it in later
ones, and let the next mount finish whatever a crash interrupted.

---

## 17. Allocation

### Inodes

- A **directory** whose parent is the root goes to the group with the fewest directories among those
  with at least the average free inodes and free blocks (spreading top-level trees); any other
  directory to its parent's group if that group has at least the average free inodes, else as for the
  root's children.
- **Any other inode** goes to its parent directory's group, else the next group (round the groups)
  with a free inode.
- In the chosen group, the lowest clear bit of the inode bitmap. Allocation sets the bit, decrements
  `free_inodes` (and increments `directories` for a directory), increments the slot's `generation`,
  and initialises the inode (all times now, `crtime` included).
- **Exhaustion**: when no group has a free inode, the operation answers `ENOSPC` and `statfs` reports
  `f_ffree = 0`. Inodes are not created dynamically (section 24). The default of one inode per 8 KiB
  costs 3.1% of the disk and runs out first only if the average file is under 8 KiB; small files,
  empty directories, inline symlinks and device nodes cost an inode and no block.

### Blocks

- **The goal** is the block after the file's last mapped physical block (so files stay contiguous);
  for a file with no blocks, the first data block of its inode's group; for an extent-tree node, its
  inode's group.
- From the goal, search the goal's group's bitmap for a free run of the wanted length; failing that,
  take the longest free run found in the group; failing that, the next group (round the groups). One
  request allocates one run, which becomes one extent.
- **Blocks freed by a transaction are not allocatable until it commits.** Each group keeps a
  per-transaction list of blocks freed in the running and committing transactions; they return to the
  allocatable pool at commit (step 7). Without this rule, a crash before the freeing transaction
  commits would leave the old owner pointing at a block already overwritten by its new owner's data.
- **Reserve**: an allocation that would leave fewer than `reserved_blocks` free answers `ENOSPC`
  unless the caller is uid 0.
- Every allocation and free updates the bitmap, the descriptor's `free_blocks`, both bitmap checksums
  and the descriptor's checksum, in the same handle.

**Pedagogical note.** "Allocate near the last block" is most of what keeps an extent-based filesystem
unfragmented; delayed allocation (deciding at write-back time, when the whole size is known) is the
next step and needs no change to the format.

---

## 18. Operations: write ordering and crash analysis

Every operation below is one transaction unless it says it is split. **A transaction is atomic**: a
crash before its commit block is durable leaves the disk as before it; after, as after it, because
recovery replays it. So for a single-transaction operation the crash analysis reduces to: *are all of
its changes in one transaction, and is every data block it makes reachable written before the commit?*
For a split operation the analysis is per step plus the orphan list.

"Dir add/remove" means the directory changes of section 10 including splits, conversions and merges;
"times" means `ctime` (and `mtime` for directories and written files) set to now.

### Create a file, device node, FIFO or socket (`openat O_CREAT`, `mknodat`)

One transaction: allocate the inode (bitmap, descriptor); write it (mode, owner from the caller,
`links` 1, times, `crtime`, device numbers); dir add in the parent; parent times.
**Crash**: either no name and a free inode, or the name and the inode. Never a name to a free inode
(both in one transaction), never an allocated inode with no name.

### `mkdirat`

As create, plus: `parent` = the parent's number, `links` 2, no blocks; the parent's `links` + 1; the
group's `directories` + 1. `EMLINK` if the parent is at `LINK_MAX`.

### `symlinkat`

As create, plus the target: inline in the inode, or — longer than 128 bytes — in one newly allocated
block, which is written as **ordered data** of the transaction (before its commit). **Crash**: a
symlink never points at an unwritten block.

### `linkat`

One transaction: the target's `links` + 1 and `ctime`; dir add in the new parent. Directories cannot
be linked (`EPERM`). **Crash**: links and names always agree.

### `unlinkat` (a non-directory)

One transaction: dir remove in the parent; parent times; the inode's `links` − 1 and `ctime`; if
`links` reaches 0, **add it to the orphan list** in the same transaction.
Then, **if links is 0 and nothing holds the inode open or mapped**, free it at once by the *delete*
sequence below; otherwise the last close runs it.
**Crash**: after the first transaction, a zero-link inode is always on the orphan list, so the next
mount frees it. Unlinking an open file is legal: the name is gone, the open descriptors keep working.

### `unlinkat AT_REMOVEDIR` (`rmdir`)

The directory must be empty (`ENOTEMPTY`) and not the root or a mount point (`EBUSY`); `.` as the
last component is `EINVAL`. One transaction: dir remove in the parent; parent `links` − 1 and times;
the victim's `links` = 0 and `ctime`; the group's `directories` − 1; orphan add. Then delete as for
unlink. An empty directory owns no blocks, so its delete is one step. A process whose cwd it was keeps
a reference; lookups in it answer `ENOENT`.

### Delete (free a zero-link inode)

Split: truncate to 0 in steps (below), then one final transaction: free the xattr block (with revoke),
zero the inode except `generation`, clear its bitmap bit, `free_inodes` + 1, orphan remove.
**Crash** at any step: the inode is still on the orphan list with links 0, so the next mount resumes.

### `renameat2` (flags 0, `RENAME_NOREPLACE`, `RENAME_EXCHANGE`)

Checks first (under the locks of section 19): the source exists; for flags 0 and NOREPLACE, a
destination of the same inode as the source is a successful no-op; a directory may not move into its
own subtree (`EINVAL` — walk `parent` from the destination directory to the root); a directory may
replace only an empty directory, a non-directory only a non-directory (`EISDIR`/`ENOTDIR`,
`ENOTEMPTY`); NOREPLACE with an existing destination is `EEXIST`.

**One transaction holds all of it:**

- If the destination name exists: its record's `ino` and `file_type` are rewritten in place (the name
  never disappears — the atomic replace POSIX promises); the **victim** gets `links` − 1 (a directory
  victim: `links` = 0 and the new parent `links` − 1) and `ctime`, and if it reaches 0 goes on the
  orphan list. Otherwise: dir add in the new parent.
- Dir remove of the source name in the old parent.
- If a **directory** moves between parents: its `parent` field becomes the new parent; the old
  parent's `links` − 1, the new parent's + 1 (`EMLINK` checked first).
- Times: both parents' `mtime`/`ctime`; the moved inode's `ctime`.
- `RENAME_EXCHANGE`: both records rewritten in place with each other's inode and type; directories'
  `parent` fields and the parents' link counts adjusted if they differ.

**Crash**: either nothing happened (both names as before) or all of it did (the source name gone, the
destination name referring to the source, the victim's count down and the victim on the orphan list
if it reached 0). The destination name is never missing and the source is never reachable by both
names, because every one of those changes is in the same transaction. Delete of the victim follows as
for unlink.

### `read`

No transaction, except an `atime` update: with the default **relatime** policy, `atime` is written
(one handle, `C_INODE`) only when it is older than `mtime` or `ctime` or more than 24 hours old.
Holes and unwritten extents read as zeros.

### `write` (and `pwrite`, `writev`)

The caller passes the offset; with `O_APPEND` the VFS takes the inode lock and uses the current `size`
as the offset, so appends from several descriptors never interleave or overwrite. A write past 16 TiB
answers `EFBIG`; one past the end leaves the gap as a hole.

**Split into chunks**, each one handle, each allocating at most 8 extents:

1. Map the chunk's blocks; for every hole, allocate a run (bitmap, descriptor, extent tree, `blocks`).
2. Copy the bytes into the blocks' buffers. A newly allocated block not wholly covered is zeroed
   first; the tail of a new last block past the new `size` is zero.
3. Set `size` if the chunk extends the file; `mtime`, `ctime`; clear set-user-ID and set-group-ID for
   a non-root writer; put the inode on the transaction's ordered list.
4. For a range that was **unwritten**: the data is written first; the extent conversion goes in a
   handle that joins a transaction whose ordered list carries the inode, so the conversion commits only
   after the data is durable.

**Crash**: each chunk is atomic, and its new blocks' data reach the disk before the commit that maps
them, so a file never maps garbage and `size` never covers unwritten blocks. A crash during a large
`write` may keep a prefix of its chunks (POSIX promises nothing without `fsync`). Overwriting blocks
already mapped and written is not ordered: after a crash such a block may hold old or new bytes (or,
torn, some sectors of each) — the price of data=ordered over data=journal.

### `truncate`, `ftruncate`

- **Growing** (`new ≥ size`): one transaction setting `size`, `mtime`, `ctime`. The new range is a
  hole. The tail-zero invariant (section 8) is what makes this safe: the old last block's bytes past
  the old size are already zero.
- **Shrinking**:
  1. First transaction: `size = new`; `mtime`, `ctime`; if `new` is not block-aligned, zero the bytes
     of the new last block from `new` to its end, as ordered data of this transaction; **orphan add**.
  2. Steps: each removes at most 16 extents (and the nodes they empty) from the end of the file
     toward `ceil(new / 4096)`, frees their blocks (revoking freed nodes), decreases `blocks`.
  3. Last step's transaction: orphan remove.
  **Crash**: after step 1, the inode is an orphan with links > 0, so the next mount frees what remains
  past `size`; the file is never seen with its old size again.

### `fallocate` (mode 0, `FALLOC_FL_KEEP_SIZE`, `FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE`)

- **Preallocate**: split into chunks as for `write`, allocating holes as **unwritten** extents; mode 0
  also sets `size` in the last chunk. **Crash**: a prefix of the chunks; unwritten extents past `size`
  are legal (section 8).
- **Punch a hole**: zero the partial blocks at the edges (ordered data of the first step), then remove
  the whole blocks in steps as for truncate. Unlike a truncation, a punch strictly inside one extent
  splits it in two, which is an insert and may need a new extent-tree node; so a punch step carries
  `C_EXT` + `C_GROUP` credits on top of a truncate step's, and a punch may answer `ENOSPC` when that
  node cannot be allocated (Linux's `fallocate` may too). Truncation and delete only ever cut the
  last extents and never need a new block. A punch does not use the orphan list: a crash part-way
  leaves the range partly freed (reading as zeros) and partly still mapped (reading as the old bytes).
  The filesystem is consistent either way — every step is a transaction — and a call that never
  returned promised nothing about its range.

### `fsync`, `fdatasync`, `sync`

- **`fsync(fd)`**: write the inode's dirty data blocks and wait; if the inode has metadata in the
  running or committing transaction, force that transaction to commit and wait for its commit; else
  flush the device. Commits are whole transactions, so a new file's directory entry — in the same
  running transaction — is made durable too.
- **`fdatasync(fd)`**: as `fsync`, but a transaction is forced only if it changed `size` or the
  inode's mapping (not only times).
- **`fsync` of a directory**: commits the transaction holding its last change.
- **`sync`**: commit the running transaction and wait; checkpointing is not needed for durability.

### `chmod`, `chown`, `utimensat`, xattrs

One transaction each: the inode (and the xattr block). `utimensat` sets `atime`/`mtime` (`UTIME_NOW`,
`UTIME_OMIT` honoured) and `ctime` = now.

### Mount and unmount

**Mount**, in this order: read and check the superblock and descriptors; refuse unknown incompat
features; recover (section 15: replay, then `error` → `ERROR`); read-only for unknown ro_compat
features, for `ERROR`, or for a device without flush; recompute totals; clear `CLEAN`, `mount_count`
+ 1, `mount_time` (direct write, flush) — the last direct write before transactions start; then process
the orphan list (section 16) in ordinary transactions. A read-only mount skips the last two (the orphan
list waits for a read-write mount or `fsck.kfs`).
**Unmount**: commit, checkpoint everything (`start = 0`), write the superblock with exact totals and
`CLEAN` (direct write, flush).

---

## 19. Concurrency on a multi-core kernel

keel runs four cores and sleeping locks (`keel.sched.Mutex`, which may be held across disk I/O) as well
as spinning ones (`IrqLock`). KFS replaces the current single filesystem-wide lock with these:

| lock | kind | protects |
|---|---|---|
| `rename_lock` | sleeping, one per filesystem | the shape of the directory tree during a rename that moves a directory or crosses directories |
| `inode.lock` | sleeping, per in-memory inode | the inode's fields, its extent tree, its directory blocks, its open count |
| `orphans.lock` | sleeping | the in-memory orphan list and the on-disk links of the list |
| `group[g].lock` | sleeping, per group | the group's bitmaps (their buffers), descriptor fields, pending-free lists, allocation cursor |
| `buffer.lock` | sleeping, per buffer | a buffer's bytes and its journal state |
| `journal.lock` | spinning | the running and committing transaction records: handle counts, credits, buffer lists |
| `cache.lock` | spinning | the buffer cache's hash and LRU |
| `icache.lock` | spinning | the in-memory inode table |

**Lock order** (take only in this order; release in any):

1. `rename_lock`
2. directory `inode.lock`s — **ancestor before descendant**; two directories neither of which is an
   ancestor of the other, in increasing inode number
3. non-directory `inode.lock`s, in increasing inode number
4. **open a journal handle** (may sleep waiting for room or for a commit)
5. `orphans.lock`
6. `group[g].lock` — **never two at once**: allocation tries a group, releases it, tries the next
7. `buffer.lock`
8. `journal.lock`, `cache.lock`, `icache.lock` — leaves: no sleeping, no other lock, while held

A handle is opened **after** the inode locks and never before: the committer waits for every handle
of its transaction to close, so a handle holder waiting for an inode lock held by someone waiting to
open a handle in a closed transaction would deadlock.

**Path lookup** locks one directory at a time, never hand over hand: lock, find the name, unlock, then
take the next component. An operation that changes a directory re-checks the name after taking its
locks. `unlinkat`/`rmdir`: parent then victim (rule 2 or 3). `renameat2`: `rename_lock` when a
directory moves or the parents differ; both parents (rule 2); then the moved inode and the victim (rule
2 if both are directories, else rule 3); the subtree check runs under `rename_lock`, so no concurrent
rename can invalidate it.

**The journal serialises commits, not operations.** Handles from every core join the running
transaction concurrently, each under its own inode and group locks; `journal.lock` is held only to
adjust counts and lists. One **commit thread** (a kernel thread, `kfs-commit`) performs commits and
checkpoints; it takes no inode lock — it writes ordered data by buffer, under buffer locks — so it
cannot be blocked by an operation.

**Per-group locks are the scaling story**: two creates in different directories in different groups
touch disjoint bitmaps and descriptors, and only meet in `journal.lock` for a few instructions. The
descriptor-table block holding both descriptors is one buffer; both handles add it to the same
transaction under `buffer.lock` in turn.

**The superblock's totals**: each group's counts change under its lock; the kernel also keeps two
atomic 64-bit counters, `free_blocks_total` and `free_inodes_total`, adjusted by the same deltas under
the same group lock. `statfs` reads the atomics (no lock). They equal the sum of the descriptors
whenever no group lock is held, and the superblock is written from them at unmount.

**What a read takes**: `read` takes the inode lock for the extent lookup and the size (a later
reader-writer lock can make concurrent readers of one file parallel; keel's scheduler has none yet).

**A split operation holds its inode's lock from its first step to its last** — a large `write`, a
shrinking `truncate`, a punch, a delete — opening and closing one handle per step inside it. Otherwise
a `write` between two truncation steps could map blocks past the new `size` that the next step then
frees, or a second truncate could start on an inode already on the orphan list. Holding an inode lock
across handles is allowed by the order above (inode locks come before handles); the commit thread
takes no inode lock, so a step waiting for a commit cannot deadlock against it.

**`orphan_next` and `ORPHAN` belong to `orphans.lock`, not to the inode's lock.** Removing an inode
from the list rewrites its *predecessor's* `orphan_next`, an inode some other core may be changing
under its own lock at that moment. So both fields live in the in-memory list, not the in-memory inode,
and whoever serialises an inode into its inode-table block copies them from the list under
`orphans.lock` (rule 5, before `buffer.lock`); no other code writes them. Without this, an inode
written back from a stale in-memory copy would silently drop a list link and leak every inode behind
it.

---

## 20. statfs

| field | value |
|---|---|
| `f_type` | `0x4B465331` |
| `f_bsize`, `f_frsize` | 4096 |
| `f_blocks` | data blocks: `blocks` − 1 − every group's metadata − the journal |
| `f_bfree` | `free_blocks_total` |
| `f_bavail` | `f_bfree − reserved_blocks`, at least 0 |
| `f_files` | `inode_count − (first_ino − 1)` |
| `f_ffree` | `free_inodes_total` |
| `f_fsid` | the first 8 bytes of `uuid` |
| `f_namelen` | 255 |
| `f_flags` | `ST_RDONLY` when read-only, `ST_RELATIME` |

Blocks freed by a transaction that has not committed are not yet in `f_bfree`; a `statfs` right after
deleting a large file may lag by up to `COMMIT_INTERVAL`.

---

## 21. Feature flags and forward compatibility

Three bitmasks in the superblock, as ext4 has:

| mask | a driver that does not know a set bit… |
|---|---|
| `feature_compat` | mounts read-write anyway (the feature changes nothing it would write) |
| `feature_ro_compat` | mounts read-only only |
| `feature_incompat` | refuses to mount |

Defined bits:

| mask | bit | name | meaning |
|---|---|---|---|
| ro_compat | 0 | `XATTR` | some inode may have an xattr block |

Every other bit is reserved and must be 0 in version 1. Examples of what would take a bit, so the next
format change has a place to go: a reflink/shared-block refcount (incompat), 64-bit logical blocks
(incompat), in-inode xattrs (ro_compat), dynamic inode allocation (incompat), a directory-hash
algorithm other than 1 (incompat, with a new `hash_algorithm` value).

`version` changes only for a change no feature bit can express; a driver refuses a version it does not
know. `fsck.kfs` refuses to repair a filesystem with unknown incompat or ro_compat bits.

---

## 22. The integrity model

| structure | protection |
|---|---|
| superblock | CRC32C over its first sector; backups in groups 1 and G − 1 |
| group descriptor | CRC32C with the group number; backups beside each backup superblock |
| block and inode bitmaps | CRC32C with the group number, stored in the descriptor |
| inode | CRC32C with the inode number |
| extent node, directory block, xattr block | CRC32C over the block including its owner and owner's generation (the block tail); a magic |
| journal superblock, descriptor, revoke, commit | CRC32C; the commit's in its first sector |
| journaled copy | CRC32C with the transaction's sequence, in its tag |
| file data | **none** |

A checksum failure in the kernel makes the call answer `EIO`, aborts the journal, records the error in
the journal superblock (section 5 says why not in the superblock) and makes the filesystem read-only.
Detection is not correction: only `fsck.kfs` repairs.

**What remains undetected**: corruption of file data; a stale but valid block written back by a
misbehaving device to the same owner at the same generation; and simultaneous loss of a structure and
every copy of it.

---

## 23. fsck

`fsck.kfs <image>` checks; `-y` repairs what it can repair safely; `--rebuild-corrupt` additionally
clears structures whose checksum fails (destructive); `-n` opens read-only and never writes. Exit
codes follow e2fsck: 0 clean, 1 errors corrected, 4 errors left uncorrected, 8 operational error. It
reports every problem it finds in one run, not just the first.

**Pass 0 — superblock, descriptors, journal.** Magic, version, checksum, geometry arithmetic (group
count, last-group rule, table sizes, journal inside group `G / 2`'s data area), unknown features.
Primary superblock bad: find a backup (section 5) and, with `-y`, restore the primary from it. Every
descriptor's checksum and its recorded locations against the computed ones. Journal superblock; if the
log is not empty, **run recovery** (section 15) first, and audit the log (every header found by
resynchronising agrees with the scan).

**Pass 1 — inodes and their blocks.** For every inode whose bitmap bit is set (except 1 and 3–10):
checksum; a valid type; flags valid for the type; device/FIFO/socket with no blocks; inline symlinks
`1 ≤ size ≤ 128`; `parent` non-zero only for directories. Walk each extent tree: magic, checksum,
`owner`, `generation`, `depth` consistency, `count ≤ max ≤ capacity`, sorted non-overlapping keys,
child ranges within the parent's, `len ≥ 1`, every extent inside one group's data area, no extent past
2³² blocks, no written extent past `ceil(size / 4096)`; the xattr block. Record every claimed block;
**a block claimed twice** goes to pass 1b. Check `blocks` against the count. Free inodes (bit clear)
should have mode 0 (a warning).

**Pass 1b — duplicate claims.** Report every inode sharing each doubly claimed block. With `-y`, copy
the block for each later claimant (as e2fsck does), so each file keeps its bytes.

**Pass 2 — directories.** Every directory block: magic, checksum, `owner`, `generation`. Leaves: records
tile the block, `rec_len` multiple of 8 and ≥ 16, `name_len` fits, no `/` or NUL, never `.` or `..`,
`ino` allocated and not reserved, `file_type` equals the inode's type, `count` equals the live records,
names unique in the directory. Indexed directories: block 0 is a root; `level`s descend by one; keys
sorted, the root's entry 0's key 0 and every interior block's entry 0's key its parent entry's; every leaf's hashes lie in its range by the rule of section 10 (the
continuation mark set exactly where a run is split); every leaf and index block reachable exactly once;
no block of the directory unreachable. Linear directories: one leaf, block 0. Empty: no blocks.

**Pass 3 — connectivity.** Every directory is reached from the root by exactly one name; its `parent`
names the directory that holds that name; the root's `parent` is itself; no cycles. With `-y`: a
directory reached by no name is linked into `/lost+found` as `#<ino>` and its `parent` set; a wrong
`parent` is corrected.

**Pass 4 — link counts.** Count every name of every inode (for a directory, 2 + subdirectories, the
root's own name counting as the first). With `-y`: correct `links`; an inode with `links > 0` and no
name goes to `/lost+found`; one with `links = 0` not on the orphan list is put on it and processed.

**Pass 5 — bitmaps, counts, orphans.** Rebuild both bitmaps from pass 1 (metadata, journal, claimed
blocks, allocated inodes) and compare: report every leaked block and every claimed-but-free block, every
inode bit wrong; compare `free_blocks`, `free_inodes`, `directories`, both bitmap checksums, and the
superblock's totals. Walk the orphan list: no cycle, every member allocated with `ORPHAN` set, no
`ORPHAN` outside the list. With `-y`: rewrite bitmaps, counts and checksums; process the orphan list
as mount would.

**Repaired automatically with `-y`**: bitmaps and counts and their checksums; link counts;
`lost+found` reconnection (creating `/lost+found` if missing, reusing it otherwise); `parent` fields;
the orphan list; duplicate claims; the primary superblock and descriptors from backups. **Never
repaired without `--rebuild-corrupt`**: an inode, extent node, directory block or xattr block that
fails its checksum (`--rebuild-corrupt` clears it, removes names pointing to a cleared inode, and
reconnects what becomes unreachable). `fsck.kfs` clears `ERROR` and sets `check_time` and
`mount_count = 0` only when it leaves the filesystem clean.

**Pedagogical note.** With a journal, fsck is not needed after a crash — recovery is. It exists for
what a journal cannot fix: bugs and bad media. Its passes are the filesystem's invariants written as a
program, which is why each one has a test that breaks exactly that invariant.

---

## 24. Design tradeoffs

| decision | cost | why |
|---|---|---|
| data=ordered, not data=journal | an overwrite of existing blocks may be torn or lost on a crash | data written once instead of twice; ext4's default |
| no data checksums | bit rot in file data is not detected | checksumming data needs either copy-on-write or a journal of data to keep a checksum and its block in step; both are large |
| no copy-on-write, no snapshots | no atomic snapshot, no cheap clone | update-in-place plus a journal is the design one team can build, test and teach in stages |
| no compression, no encryption | | out of scope; neither changes what the rest of the design must do |
| fixed inode count | can run out before blocks do (`ENOSPC` with free space) | a fixed table is O(1) to locate; dynamic inodes (XFS) need an inode B-tree |
| no inline data for small files | a 10-byte file costs a block | one less inode union arm; ext4's inline data is an incompat feature added late |
| allocation at write time | less contiguity than delayed allocation | simpler; the format permits delayed allocation later |
| synchronous commit (flush before the commit block) | two flushes per commit | per-block tag checksums would allow dropping the first (jbd2's async commit); it is left as an option because the reasoning is harder |
| one xattr block, unshared | ≤ 4 KiB of attributes per inode; identical ACLs are not deduplicated | nothing in keel needs more |
| 32-bit logical blocks | 16 TiB files | 16-byte extents; widening is an incompat bit |
| 4 KiB blocks only | no tuning for tiny or huge files | the page size; every structure is sized for it |
| no bad-block list | | a virtual disk remaps its own bad sectors |

---

## 25. Changes from SFS, and why

**Kept**: a superblock-described layout with backups; metadata journaling with data written before
commit; the commit block as the atomicity point; hash-indexed directories; directory blocks bound to
their owner by a checksummed tail; nanosecond timestamps and `crtime`; inline symlinks; preallocation
by unwritten extents; sparse files; `fsck`; pedagogical notes.

**Changed, as asked**:

| change | why |
|---|---|
| **Revoke records** | a freed metadata block reused for data would otherwise be overwritten by replay of an old copy (section 14 has the exact sequence) |
| **The orphan list** | makes unlink of an open file and rename-over of an open file legal and crash-safe, and makes multi-step truncation restartable |
| **Extents carry their logical block, in a B-tree rooted in the inode** | O(log n) lookup instead of a linear walk of positional extents; holes are absent ranges, so `SPARSE` is gone; `UNWRITTEN` stays |
| **64-bit seconds** | 32-bit seconds end in 2038 |
| **SipHash-1-3, keyed per filesystem, 64-bit; a continuation mark** | an unkeyed FNV-1a lets anyone force collisions; the mark gives an exact rule for runs of equal hashes across splits |
| **CRC32C** | aarch64 has instructions for it, and its polynomial has better error-detection properties than CRC32's at the lengths checked here, at equal cost |
| **Allocation groups** | locality, and per-group locks so cores do not serialise on one bitmap; totals derived, not shared |
| **Inode density per group, 1 per 8 KiB default** | SFS's 1 per 2 MiB ran out long before the disk filled with ordinary files |
| **Device nodes, FIFOs, sockets, `DT_*` numbering** | `/dev/null` and friends; `getdents64` answers Linux's values directly |

**Changed by this specification's own review**:

| change | why |
|---|---|
| Blocks freed by a transaction are not reusable until it commits | otherwise a crash before that commit leaves the old owner pointing at the new owner's data |
| The device model is explicit: sector-atomic writes, torn blocks, reordering until flush | the superblock, journal superblock and commit block keep their meaning in one sector, so their in-place updates are atomic; everything else is journaled or is data |
| Explicit flushes in commit and checkpoint; `VIRTIO_BLK_F_FLUSH` required for read-write | without a barrier the commit block can overtake what it commits |
| Per-copy checksums in descriptor tags; descriptor and revoke blocks checksummed individually | SFS's single transaction CRC could not say which block was bad, nor check a block before replaying it |
| A buffer modified by an uncommitted transaction is never written home (pinned), and checkpoint forces a commit when it must | checkpointing T1 must not write T2's uncommitted changes in place |
| Frozen copies: T + 1's first change to a buffer of a committing T copies T's bytes first | otherwise T's log could carry T + 1's half-done changes |
| Log space is reserved when a handle joins, never at commit | a commit that waited for a checkpoint that waited for a commit would deadlock |
| Credits per handle, `max_transaction = journal / 4`, `MAX_OP_CREDITS`, and splitting of unbounded operations with the orphan list | SFS's 1024-block cap did not say what happens to an operation larger than a transaction |
| Truncation sets `size` first and is an orphan until done; the tail-zero invariant | a truncate interrupted by a crash finishes at the next mount, and growing a file never exposes old bytes |
| `.` and `..` synthesized; directories carry `parent`; an empty directory has no blocks | no special root-block layout; a directory rename touches the moved inode instead of a block inside it; `mkdir` allocates nothing |
| `readdir` positions are hashes, with the run rule and `EINVAL` | `telldir`/`seekdir` and resumed `getdents64` stay correct while the directory changes |
| Directories shrink: empty leaves freed (holes), siblings merged, root collapse, empty directory releases everything | SFS directories only grew |
| Checksums are seeded by the UUID and bound to identity (group, inode number, owner + generation) | a block from another filesystem, another group, another inode or a previous life of the same inode is detected |
| The generation survives free and is checked in every owned block's tail | stale blocks from a deleted file cannot pass as the new file's |
| Bitmaps are checksummed through their group descriptor | SFS did not protect bitmaps |
| Backups at groups 1 and G − 1, discoverable without the primary; block 0 left to the boot block | SFS's backup at block 1 sat beside the primary |
| 64-bit inode numbers in directory entries; 64-bit block numbers throughout | no 2³² ceiling anywhere a disk can reach |
| `max` in extent and index headers may be below capacity | lets host tests exercise deep trees on small RAM disks; costs nothing on disk |
| Xattrs defined (one owned block, ro_compat `XATTR`) | SFS reserved a pointer without a format |
| Feature masks compat / ro_compat / incompat | a defined path for every future format change |
| fsck: duplicate-claim pass, connectivity via `parent`, orphan-list audit, exit codes, a destructive mode kept separate | SFS's fsck was report-mostly |
| Dropped: the bad-block inode; triple-indirect extent blocks; `SPARSE` | a virtual disk remaps; the B-tree replaces the indirect tiers; holes are absence |
| An error found while mounted aborts the journal and is recorded in the journal superblock, copied into `ERROR` at the next mount | a direct write of the journaled block 1 could carry uncommitted changes home, and replaying an older copy of block 1 would clear the bit |
| Orphan processing runs after mount's direct superblock write, not inside recovery | direct writes of block 1 are only safe while the log is empty |
| A split operation holds its inode lock across all its steps | a write between truncation steps could map blocks the next step frees |
| `orphan_next`/`ORPHAN` are owned by `orphans.lock` and copied in when an inode is serialised | a predecessor's link is rewritten by whoever removes its successor, possibly while another core writes that predecessor |
| Handles check their credits | a credit undercount fails a test instead of overflowing the log |
| A punch step carries split credits and may answer `ENOSPC` | punching inside an extent splits it, which may need a new tree node |

---

## 26. Pedagogical notes

- **Every crash rule names its crash.** Revoke records, free-after-commit, the tail-zero invariant,
  pinning uncommitted buffers, orphans for truncation: each exists because a specific interleaving of
  writes and a power cut would otherwise corrupt something, and each section spells that interleaving
  out. The crash-injection tests (section 27) are those interleavings, run.
- **Atomicity comes from one sector.** A 4 KiB write can tear; a 512-byte one cannot. Everything
  KFS updates in place without a journal fits in one sector; everything else gets its atomicity from
  the commit block, which itself fits in one sector.
- **Identity in checksums** turns "is this block intact?" into "is this the block I meant?" —
  misdirected writes and stale blocks are corruption a plain CRC cannot see.
- **Names and inodes are separate**, so hard links, rename and unlink-while-open all fall out of one
  idea; `links` and the orphan list are the bookkeeping.
- **Groups are a scaling technique twice over**: for the disk (locality) and for the processor
  (locks). The totals that would undo the second are derived instead of stored.
- **The extent tree and the directory index are the same idea** — a B-tree keyed by something the
  operation already knows (a logical block, a name's hash) — applied to two different questions.

---

## 27. Implementation plan

Thirteen stages in dependency order. Each is one bounded deliverable, ends with a commit on its own
branch fast-forwarded into `dev` after `scripts/boot-test.sh` and the host suites are green, and
leaves the old filesystem working until stage 12 retires it.

**Where things live.**

- `kfs/` — workspace member `kfs`, module `keel.kfs`: the format and every algorithm, compiled into
  both the kernel (aarch64-freestanding) and the host tools, as `ondisk/` is today. It is written
  against a small **platform** interface supplied by its user: a block device (`read`, `write`,
  `flush`, `blocks`), a clock, the locks of section 19, and a hook to run the committer. On the host
  the locks are trivial and the committer runs inline; in the kernel they are `keel.sched`'s. Its
  tests are `@tests` files run by `sysl test kfs` on the host.
- `kfs/SPEC.md` — this document.
- `tools/mkfs-kfs`, `tools/fsck-kfs`, `tools/kfsdump` — host programs built to `scratch/mkfs.kfs`,
  `scratch/fsck.kfs`, `scratch/kfsdump`.
- `kernel/keel/kfs/` — the kernel's glue: the buffer cache with pinning, the commit thread, the inode
  cache, the VFS-facing calls.

**Test fixtures shared by every stage from 5 on.**

- **`RamDevice`** — a block device over a heap buffer, counting reads, writes and flushes, with a log
  of every write in order.
- **`CrashDevice`** wrapping a `RamDevice`, in three modes: **drop after N** (every write after the
  Nth is lost); **tear the Nth** (the Nth write persists only a chosen subset of its eight sectors, the
  rest lost); **reorder** (at the crash, an arbitrary seeded subset of the writes since the last flush
  is lost). After the crash, the test remounts the surviving image, which must recover, and runs the
  package's **checker**, which must report clean.
- **The checker** (`keel.kfs.check`) is fsck's read-only half. Stage 5 creates it with passes 0 and 5
  (superblock, descriptors, journal, bitmaps and counts) and pass 1 for what exists then; each later
  stage adds the checks for the structures it introduces, in the same commit, so the crash harness
  always checks everything that exists. Stage 9 adds repair, the host program, and the corruption
  tests.
- **The every-write loop**: for an operation, count its writes W on a clean run; then for N = 1 … W,
  and for each mode, crash at N, recover, check clean, and check the visible state is exactly the
  state before the operation or exactly the state after it (for a split operation: before, after, or
  a documented intermediate state with the orphan list finishing it).

### Stage 1 — Codecs and structures

Builds: constants; `C(seed, bytes)` CRC32C (table-driven); SipHash-1-3; the directory hash with its
clamp; every on-disk structure of sections 5–14 as a struct with `@assert`ed size and field offsets;
seal/verify helpers for each checksum; `RamDevice`.
Tests: CRC32C vectors (`"123456789"` → 0xE3069283, the iSCSI vectors of RFC 3720 B.4); SipHash-1-3
reference vectors; `hash` never odd, never below 8 or above 2⁶⁴ − 4 (fed names whose raw SipHash is
forced to the edges by a test hook); every structure round-trips; every checksum changes when one bit
of its covered bytes, its identity, or its seed changes, and does not depend on any byte outside its
coverage; the LE assertion.

### Stage 2 — Geometry and format (`mkfs.kfs`, `kfsdump`)

Builds: geometry selection (section 4) and refusal; writing groups, descriptor tables and backups,
bitmaps with metadata and the journal marked, the inode table zeroed, reserved inodes, the root (2,
empty, no blocks) and `/lost+found`, an empty journal. `kfsdump` prints the superblock, descriptors,
any inode, any extent tree and any directory. `mkfs.kfs` takes the filesystem size and, optionally, a
host directory and a **spec file** (lines `<path> <type> <mode> <uid> <gid> [<major> <minor> | <target>]`)
for owners, device nodes and symlinks — filled in stage 10 once files can be written.
Tests: geometry at the edges (just under and over 2 × 2048 blocks, a last group of 63 and 64 data
blocks, the largest `blocks_per_group`, a journal that does not fit); every computed location equals
the descriptor's; the bitmap bits of metadata and past-the-end blocks; corrupting block 1 → the backup
is found by the search of section 5.

### Stage 3 — Allocation

Builds: the inode and block allocators of section 17 over the platform's group locks; pending-free
lists released by a commit callback; the reserve; the totals.
Tests: inode placement (top-level directories spread, files beside their parent); contiguity from a
goal; fallback to other groups; exhaustion answers `ENOSPC` for inodes and for blocks; uid 0 may use
the reserve and others may not; a block freed in an uncommitted transaction is not handed out until
the commit callback; counts and bitmap checksums always agree with the bitmaps.

### Stage 4 — The extent tree

Builds: lookup, insert with splits and growth, range removal with splits, shrink and root collapse,
unwritten conversion, optional merging.
Tests: a model-based property test — random inserts, removals and conversions against a reference map
from logical to physical blocks, with node `max` set small (2–4) so trees reach depth 4; holes read as
unmapped; the 2³² limit; node checksums, owner and generation verified on every read (a node from
another inode, or from a previous generation, is refused); every node's entries sorted and inside its
parent's range after every step.

### Stage 5 — The journal and recovery

Builds: handles with credits; the running/committing transaction; descriptor, copy, revoke and commit
writing with escaping; the commit ordering of section 14 with its two flushes; frozen copies;
pinning; log-space reservation at handle start; checkpoint; wrap; recovery's scan, revoke and replay
passes; `CrashDevice` and the every-write loop; the checker's first passes.
Tests: a transaction that exactly fills `max_transaction`; one more credit refused; log wrap with a
transaction straddling the end; recovery stops at a missing commit, a commit of the wrong sequence, a
stale transaction from the previous lap, a gap in sequence; a bit flip in a copy, a descriptor, a
revoke block, or the commit's first sector each rejects that transaction; every journal magic escaped
and restored; a 600-copy transaction spanning three descriptors; the same block journaled three times
in one transaction logs once; **revoke**: a block journaled in T1, revoked in T2, rewritten as data in
T3, crash — replay leaves the data; a block revoked then re-journaled in the same transaction is
replayed; revoke ordering across a wrap; double recovery (crash during recovery, recover again) gives
the same bytes; recovery after commit but before checkpoint makes the change visible; **ordering**: the
write log shows ordered data before the descriptors, a flush, then the commit, then a flush; a pinned
buffer is never written home before its commit; a buffer changed by T + 1 while T commits is logged
in T with T's bytes only (the frozen copy); a log kept nearly full by a stream of handles never
deadlocks (handles wait, checkpoints run, every handle finishes).

### Stage 6 — Files: I/O, truncation, the orphan list

Builds: the inode layer (read, write and free of inodes, generation, times); `read` with holes and
unwritten extents; chunked `write` with ordered data and append; `truncate` both ways with the tail
zeroing; `fallocate` (preallocate, keep-size, punch); `fsync`/`fdatasync`; the orphan list with its
in-memory double links and mount-time processing; delete; the checker's inode, extent-tree and
orphan-list checks.
Tests: holes and `st_blocks`; writes across extent boundaries and into unwritten ranges; the tail-zero
invariant after shrink-then-grow; `EFBIG`; crash loops (every write, three modes) over: appending a
chunk, writing into an unwritten range, growing truncate, **shrinking truncate of a 20,000-extent
file** (many steps — after every crash the file has its new size and the next mount frees the rest),
delete of an open (orphaned) inode, punching; `fdatasync` forces no commit for a times-only change.

### Stage 7 — Directories

Builds: leaves, linear directories, conversion, the hash index with probe, lookup, insert and split
under the continuation rule, removal with the four shrinking rules, `readdir` positions; the checker's
pass 2.
Tests: a hash hook that forces collisions (every name one hash; names in two hash values alternating)
— runs spanning two, three and many leaves, across index-block boundaries, lookup finds every name,
the continuation mark set exactly at run splits; splits keep every name reachable; promotion from
linear to one index level and to three; interior index splits; delete then reinsert after promotion;
slack reuse; names of 1 and 255 bytes and non-ASCII UTF-8; drain to empty frees every block and the
directory accepts entries again; thousands of alternating inserts and removals against a model;
`readdir` while inserting and removing returns every untouched name exactly once; a collision run that
does not fit the buffer answers `EINVAL`; leaf and index blocks with a bad checksum, wrong magic, or
another directory's owner are refused.

### Stage 8 — Namespace operations

Builds: create, `mknodat` (all four types), `mkdirat`, `symlinkat` (inline and block), `readlinkat`,
`linkat`, `unlinkat`, `rmdir`, `renameat2` (0, NOREPLACE, EXCHANGE), `setattr`, xattrs, `statfs`,
mount and unmount; the checker's passes 3 and 4.
Tests: every error of section 18 (`EEXIST`, `ENOTEMPTY`, `EISDIR`, `ENOTDIR`, `EINVAL` for a move into
the own subtree, `EMLINK`, `EPERM` on linking a directory); link counts of directories through mkdir,
rmdir and rename between parents; rename onto a hard link of itself is a no-op; rename-over of an open
file keeps it readable through its descriptor and frees it at close; the every-write crash loop over
**each** operation, the visible state being exactly before or after; statfs counts move by the right
amounts after commit.

### Stage 9 — fsck

Builds: repair for every pass of section 23 (`-y`, `--rebuild-corrupt`, `-n`, exit codes, pass 1b,
`lost+found`) on top of the checker the earlier stages grew, in the package, with the thin host
program `fsck.kfs`.
Tests: one corruption per check, each producing its named diagnostic and (where repairable) a clean
second run after `-y`: primary superblock bad (backup restores it); a descriptor's checksum; a bitmap
bit set for no owner (leak) and clear for an owned block; an inflated link count; an inode with no
name (→ `lost+found`, created, then reused for a second); a directory with a wrong `parent`; an
unreachable directory; a dangling entry; a duplicate claim (copied); an orphan-list cycle; a zero-link
inode not on the list; a wrong continuation mark; checksum failures **not** repaired without
`--rebuild-corrupt`; several problems reported in one run; a freshly formatted image and an image after
stage 8's workload are clean.

### Stage 10 — Populating images on the host

Builds: `mkfs.kfs` fills an image from a host directory and the spec file through the package's own
operations (so the image is made by the same code the kernel runs); `kfsdump get <image> <path>
<host-file>` and `kfsdump put`. 
Tests: an image built from a tree with nested directories, a 200 KiB file, symlinks, device nodes and
an owners line is `fsck`-clean and reads back byte for byte; `boot-test.sh`'s fsroot builds into a
32 MiB image.

### Stage 11 — KFS in the kernel, beside the old filesystem

Builds: virtio-blk flush (`VIRTIO_BLK_F_FLUSH`, request type 4), multi-sector requests, and more than
one device (the driver drives the first disk today); the kernel
platform (the buffer cache with pinning and eviction rules, `keel.sched` locks, the `kfs-commit`
thread, the inode cache with open counts); a second disk attached by `boot-test.sh` holding a KFS image,
mounted by a selftest while the old filesystem stays the root.
Tests (boot-test lines): `keel: virtio-blk flush ok`; `keel: kfs mounted, 8192 blocks, 4 groups,
journal 1024`; `keel: kfs smp ok` (four threads on four cores creating, writing and removing files in
four directories at once, then a host `fsck.kfs` of the second image is clean);
`keel: kfs replayed <n> transactions` on a boot that follows one ended on purpose after a commit and
before its checkpoint, with the files present and the host `fsck.kfs` clean.

### Stage 12 — The switch

Builds: keel's file calls (`openat`, `read`/`write`/`pread64`/`readv`/`writev`, `lseek`,
`newfstatat`/`fstat` with times and `st_blocks`, `getdents64` with positions and all `DT_*`,
`mkdirat`, `unlinkat`, `renameat` and `renameat2`, `linkat`, `symlinkat`, `readlinkat`,
`truncate`/`ftruncate`, `fallocate`, `fsync`/`fdatasync`/`sync`, `utimensat`, `fchmodat`,
`fchownat`, `statfs`/`fstatfs`, the xattr calls) on KFS, with path resolution following symlinks; the
root mounted from KFS; `boot-test.sh` building the disk with `mkfs.kfs` (32 MiB) and checking it with
`fsck.kfs` after each boot; the old `kernel/keel/fs/`, `ondisk/` and `tools/mkfs` removed.
Tests (boot-test lines, besides every existing line still passing):
`keel: kfs root mounted, 8192 blocks`; `user: unlinked file still readable` (open, unlink, write,
read back, the name absent from `ls`); `user: rename replaced an open file`; `busybox ln -s
/hello.txt /l` then `cat /l` printing the file and `busybox readlink /l` printing `/hello.txt`;
`busybox stat -c %Y /hello.txt` printing the format time and a `touch` moving it; on boot 2,
`user: persisted across reboot` and `keel: kfs orphans 0`; `fsck.kfs: clean` after each boot.

### Stage 13 — Device nodes and `/dev`

Builds: `mknodat` in the kernel; opening a character device routed to a `(major, minor)` device table
serving null, zero, full, random, urandom, tty and console (the console's descriptor kind, so termios
and job control work through `/dev/tty`); FIFOs opened by name attached to a pipe keyed by the inode;
`st_rdev`; `/dev` built by `mkfs.kfs` from the spec file; some busybox applets installed as symlinks to
`/bin/busybox`.
Tests (boot-test lines): `echo x > /dev/null`; `busybox head -c 4 /dev/zero | busybox od -An -tx1`
printing ` 00 00 00 00`; `echo x > /dev/full` failing with `ENOSPC`; `ls -l /dev/null` showing
`crw-rw-rw-` and `1, 3`; `busybox mkfifo /tmp/f`, a writer in the background and a reader through the
name; an applet run through its symlink (`md5sum /hello.txt`); a non-root `mknod` of a device refused
(`EPERM`).
