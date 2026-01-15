# WAL-CPP Design

This document describes the **architecture, responsibilities, and invariants**
of the WAL-CPP system.

The goal of this design is to provide a **reusable, minimal, and correct**
write-ahead log suitable for high-performance and correctness-critical systems.

---

## 1. Overview

WAL-CPP is a **single-writer, append-only write-ahead log**.

Its responsibilities are strictly limited to:

- ordering
- durability
- crash-safe recovery
- deterministic replay

All business meaning, serialization, and state reconstruction are
**explicitly delegated to the user**.

---

## 2. Architecture

![Architecture](High-level-wal-architecture.png)

### Flow Description

1. The user or engine produces domain events
2. Events are serialized into binary payloads
3. A sequencer assigns a strictly increasing sequence number
4. Records are submitted to the WAL via a queue
5. A single WAL thread persists data and metadata
6. On restart, WAL replays records for user-controlled recovery

The WAL does not know what the data represents — only **that it exists and in what order**.

---

## 3. Single-Writer Model

The WAL operates with **exactly one writer thread**.

This thread exclusively owns:

- file descriptors
- segment rotation
- fsync operations
- metadata updates

This design:

- avoids locks
- guarantees deterministic ordering
- simplifies recovery logic

All concurrency happens **before** WAL submission.

---

## 4. WAL Internal Pipeline

![Internal Pipeline](WAL-Internals.png)

The WAL thread performs the following steps in strict order:

1. Drain ingress queue
2. Build a batch of records
3. Append batch to active segment file
4. fsync segment file
5. Update metadata with commit information
6. fsync metadata file

Only after step 6 is a batch considered **durably committed**.

---

## 5. Record Format

![Record Layout](WAL-Record-Layout.png)

Each WAL record consists of:

### WAL Header (WAL-owned)

- WAL format version
- Record type
- Schema version
- Sequence number
- Payload size
- Checksum

### User Payload (User-owned)

- Arbitrary binary data
- Any structure or encoding
- Never inspected by WAL

The sequence number is part of the **WAL header**, not the payload.

---

## 6. Sequence Numbers

Sequence numbers are:

- strictly monotonic
- globally ordered
- required for recovery and replay

WAL **validates** sequence ordering but does not assign meaning to it.

A separate sequencer component is responsible for generating sequence numbers.

---

## 7. Recovery Model

![Recovery Flow](Recovery&Truncation-Flow.png)

On startup, WAL performs:

1. Read `wal.meta`
2. Determine last committed sequence
3. Scan WAL segments sequentially
4. Validate headers and checksums
5. Stop on corruption or end-of-log

Recovered records are then provided to the user for replay.

WAL recovery is **self-contained** and does not require user logic.

---

## 8. Replay Model

Replay is **user-driven**.

The user controls:

- replay start sequence
- record filtering
- payload deserialization
- state reconstruction

WAL only guarantees ordered delivery of valid records.

---

## 9. Truncation and Retention

WAL does not decide when data is no longer relevant.

Truncation requires an explicit user-provided **safe sequence**, typically
after a snapshot is completed.

Rules:

- Only sealed segments may be deleted
- Partial segment deletion is forbidden
- WAL never guesses truncation safety

---

## 10. Design Invariants

The following invariants must always hold:

- Exactly one WAL writer thread
- Append-only log structure
- Total ordering by sequence number
- Payloads are opaque to WAL
- Metadata is the source of commit truth
- No truncation without explicit safety proof
- Deterministic recovery and replay

---

## 11. Non-Goals

WAL-CPP explicitly does NOT:

- manage schemas
- deserialize payloads
- provide messaging semantics
- perform business validation
- make retention decisions

---

## 12. Future Extensions (Out of Scope)

Potential future extensions include:

- multiple WAL instances per shard
- configurable fsync policies
- async durability modes
- tooling and inspection utilities

These extensions must not violate existing invariants.
