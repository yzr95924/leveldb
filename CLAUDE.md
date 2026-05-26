# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug .. && cmake --build . --parallel
```

Run tests:
```bash
ctest --verbose
# Single test: ./build/leveldb_tests --gtest_filter=DBTest.Empty
```

Format code:
```bash
clang-format -i --style=file <file.cc>
```

## Architecture Overview

LevelDB is a persistent ordered key-value store with a multi-level structure similar to Bigtable tablets.

### Storage Hierarchy

- **L0 (Young level)**: Newly flushed sstables from the memtable. Files here may have overlapping key ranges. When 4+ files accumulate, they compact with L1.
- **L1+**: Sorted sstables with non-overlapping key ranges. Each level is ~10x larger than the previous (L1=10MB, L2=100MB, etc.).
- **MemTable**: In-memory sorted structure (skiplist) holding recent writes. When it reaches ~4MB, it is flushed to L0 as an sstable.
- **Log file**: Write-ahead log. Each record is also inserted into the memtable. Log is converted to sstable during flush.

### Core Flow

```
Put(key, value) → Write to log → Insert into memtable
                            ↓ (when memtable is full)
                    Flush to L0 as sstable → Background compaction
                            ↓ (compaction moves data to higher levels)
```

### Key Files

| Directory | Purpose |
|-----------|---------|
| `db/` | Database engine core: `db_impl.cc` (DB interface impl), `version_set.cc` (L0-L6 management), `memtable.cc` (in-memory skiplist), `compaction.cc` (background compaction) |
| `table/` | SSTable format: `table.cc`, `block.cc`, `block_builder.cc`, `filter_block.cc` (Bloom filters) |
| `util/` | Utilities: `arena.cc` (memory allocator), `cache.cc` (LRU cache), `coding.cc` (fast binary encoding), `bloom.cc` |
| `port/` | Platform abstractions: `port.h` defines `port::Mutex`, `port::CondVar`, thread annotations |
| `include/leveldb/` | **Public API only**. Never include internal headers from here. |

### Internal Data Structures

- **MemTable** (`db/memtable.cc`): Skiplist of `MemTableInserter` records. Uses `Arena` for allocation.
- **Version/VersionSet** (`db/version_set.cc`): Represents the current state of all sstables across all levels. VersionEdit applies changes.
- **TableCache** (`db/table_cache.cc`): Caches open sstables and their index/filter blocks.
- **Block** (`table/block.cc`): Individual sstable data block with restart points for binary search.

### Key Classes

- `DBImpl`: Main database implementation. Owns `memtable_`, `imm_memtables_`, `versions_`.
- `VersionSet`: Manages manifest log and version lifecycle.
- `Compaction`: Background compaction state machine.
- `DBIterator`: Merges iterators across memtable + imm_memtables + current version's sstables.

### Recovery

On startup: read CURRENT → read MANIFEST → recover memtables from log files → open all sstables lazily (via TableCache).

### Thread Safety

- Uses `port::Mutex` and `port::CondVar` (not std::mutex).
- `GUARDED_BY(mutex_)` and `EXCLUSIVE_LOCKS_REQUIRED(mutex_)` annotations for static analysis.
- Immutable data structures can be accessed without locks.