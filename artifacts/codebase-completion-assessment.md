# FrankenSQLite Codebase Completion Assessment

**Date:** 2026-02-26
**Scope:** Deep code review of all 26 crates (545K lines Rust, 582 files)
**Branch:** claude/review-codebase-completion-1PhVz

---

## Executive Summary

FrankenSQLite is an exceptionally well-architected project with production-quality
components that are **not yet connected end-to-end**. By bead count, 90.8% of tasks
are closed. By remaining effort, roughly 60% of the work lies ahead -- because the
remaining 9.2% is integration work, which is the hardest and most unpredictable kind.

**Estimated time to persistent CRUD:** 5-8 weeks (optimistic)
**Estimated time to full feature parity:** 4-6 months (single senior dev)
**"Done in 2 weeks" verdict:** Not supported by the code.

---

## The Smoking Gun

**`connection.rs` lines 5-8:**
> Table storage currently uses the in-memory `MemDatabase` backend for execution,
> while a `PagerBackend` is initialized alongside for future Phase 5 sub-tasks
> (bd-1dqg, bd-25c6) that will wire the transaction lifecycle and cursor paths
> through the real storage stack.

**`engine.rs` lines 9-10:**
> Cursor-based opcodes (OpenRead, Rewind, Next, Column, etc.) are stubbed and
> will be wired to the B-tree layer in Phase 5.

Every INSERT, UPDATE, DELETE, and SELECT goes through `MemTable` -- a `Vec<MemRow>`.
The B-tree, pager, VFS, WAL, and MVCC all exist and work individually. They are not
connected to the SQL execution path.

---

## What's REAL (Component-Level Assessment)

### Storage Layer
| Component | Lines | Status | Quality |
|-----------|-------|--------|---------|
| UnixVfs | ~2,600 | REAL | Production: POSIX fcntl, inode coalescing, SHM |
| Pager (SimplePager) | ~3,100 | REAL | Production: page-aligned I/O, journal, WAL hookup |
| Page Cache (ARC) | ~3,700 | REAL | Production: adaptive replacement cache |
| Page Cache (S3-FIFO) | ~2,600 | REAL | Production: write-optimized eviction |
| B-tree Cursor | ~3,900 | REAL | Production: page traversal, binary search, overflow |
| B-tree Balance | ~2,400 | REAL | Production: page split/merge |
| TransactionPageIo | ~100 | REAL | Bridge adapter (exists but not exercised for DML) |

### WAL and Crash Recovery
| Component | Lines | Status | Quality |
|-----------|-------|--------|---------|
| WAL | ~3,000+ | REAL | SQLite-compatible frames, checksum chain |
| Crash Recovery | Tested | REAL | Subprocess abort() + verify: not mocked |
| Checkpointing | All 4 modes | REAL | PASSIVE/FULL/RESTART/TRUNCATE |
| RaptorQ FEC | ~3,000 | REAL | RFC 6330, tested at 5-10% corruption |
| Rollback Journal | ~774 | REAL | SQLite 3-format, pre-image capture |

### MVCC and Concurrency
| Component | Lines | Status | Quality |
|-----------|-------|--------|---------|
| Version Chains | ~4,000 | REAL | Lock-free CAS, generation-counted arena |
| SSI Detection | ~4,000 | REAL | Full T1/T2/T3 rules, 64-thread stress tested |
| Page-Level Locks | ~1,400 | REAL | Lock-free, 1M entry capacity |
| EBR / GC | ~1,500 | REAL | Crossbeam-epoch, incremental pruning |
| Cross-Process IPC | ~250+ | REAL | Unix socket coordinator, shared TxnSlots |

### Query Pipeline
| Component | Lines | Status | Quality |
|-----------|-------|--------|---------|
| Parser | ~2,400 | REAL | Hand-written recursive descent + Pratt |
| Query Planner | ~400+ | REAL | Cost-based with cardinality estimation |
| Codegen | ~2,000+ | REAL | Produces VDBE bytecode |
| VDBE Engine | ~8,200 | REAL | 118-152 of 191 opcodes handled |
| Vectorized Engine | ~5,000+ | REAL | Hash join, leapfrog trie, SIMD sort |
| Functions | ~1,600+ | REAL | 50+ scalar/aggregate builtins |

---

## What's NOT Connected

```
 SQL string
    |
    v
 Parser ------------ REAL
    |
    v
 Query Planner ----- REAL (cost-based)
    |
    v
 Codegen ----------- REAL (produces bytecode)
    |
    v
 VDBE Engine ------- REAL (118+ opcodes)
    |
    | cursor ops go here:
    | OpenRead/Write, Rewind, Next, Column, Insert, Delete
    |
    +---> MemTable (Vec<MemRow>)  <-- THIS IS THE ACTUAL PATH
    |     in-memory, no disk, no B-tree, no pages
    |
    +---> StorageCursor -> BtCursor -> TransactionPageIo -> Pager -> VFS
          THIS PATH EXISTS IN CODE but is NOT the default execution path
          for user DML. The adapter exists. BtCursor works. But connection.rs
          sends everything through MemDatabase.
```

---

## Why Integration Takes Longer Than 2 Weeks

### 1. MemTable to BtCursor semantic gap
MemTable stores `Vec<SqliteValue>`. B-tree stores serialized SQLite record format
(varint headers + type codes + packed payload) inside page cells with overflow chains.
Every cursor operation in 8K lines of engine.rs needs auditing for the difference.

### 2. Index persistence does not exist
MemDatabase uses in-memory data structures for indexes. Persistent indexes require
separate B-trees with consistent updates during every INSERT/UPDATE/DELETE.

### 3. Pager freelist is in-memory only
`pager.rs` line 45: "TODO: This is currently an in-memory freelist... pages freed
here are NOT persisted to the database file's freelist structure." DELETE leaks disk
space until restart.

### 4. Transaction opcodes are no-ops
Transaction, AutoCommit, Savepoint, Checkpoint are all stubs (lines 1903, 3181).
The pager has real transaction support, but it is not invoked from the execution path.

### 5. 72 unimplemented opcodes include essential ones
- Virtual table operations (11): needed for FTS5/R-tree
- Schema management (ParseSchema, DropTable, DropIndex): needed for DDL
- Foreign key enforcement (FkCounter, FkIfZero)
- Trigger execution (Program, Param)
- VACUUM, IntegrityCk

### 6. Cross-process concurrency is untested
The MVCC stack is tested with multi-threaded stress tests, but no test spawns two
OS processes hitting the same database file.

### 7. Test suite validates in-memory correctness
738+ tests run against MemDatabase. When storage wiring changes, many will break.

---

## Realistic Timeline Estimates

### Persistent CRUD (basic disk-backed SQL)
- StorageCursor wiring (bd-1dqg + bd-25c6): 2-3 weeks
- Transaction lifecycle through pager: 1 week
- Basic index persistence: 1-2 weeks
- Fixing cascading test failures: 1-2 weeks
- **Total: 5-8 weeks**

### All Remaining Beads (119 open/in-progress)
- Triggers, foreign keys, generated columns: 4-6 weeks each
- WITHOUT ROWID tables: 2-3 weeks
- Virtual table wiring (FTS5/R-tree): 3-4 weeks
- VACUUM: 2 weeks
- Cross-process MVCC validation: 2-4 weeks
- Performance parity: ongoing
- **Total: 4-6 months** (single senior dev)

---

## Code Quality Metrics

| Metric | Value |
|--------|-------|
| Total Rust LOC | 545,415 |
| Source files | 582 |
| Crates | 26 |
| TODO/FIXME markers | 7 |
| unimplemented!() macros | 0 |
| unsafe code blocks | 0 |
| Test functions | 738+ |
| Test files | 233 |
| VDBE opcode coverage | 62-80% (118-152 of 191) |
| Bead completion rate | 90.8% (1,175/1,294) |

---

## Conclusion

This is among the most impressive Rust database projects one could find. The
architecture is sound, the code quality is exceptional, and the individual components
are production-ready. But "production-ready components" does not equal
"production-ready database."

The remaining work is integration: threading the storage stack through the VDBE,
making transactions durable, persisting indexes, wiring cross-process coordination.
This is inherently serial work that defies parallelization and resists estimation.

Two weeks is not supported by the evidence in the code.
