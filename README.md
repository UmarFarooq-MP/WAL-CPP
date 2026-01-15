# WAL-CPP

WAL-CPP is a **single-writer, append-only Write-Ahead Log (WAL)** designed for  
**deterministic recovery, crash safety, and ordered replay**.

This repository provides a **pure WAL implementation**.  
It is intentionally limited to **durability, ordering, and recovery**.

WAL-CPP does not contain, depend on, or assume any business logic.

---

## Scope

This repository is strictly limited to the implementation of a write-ahead log.

WAL-CPP does **not**:

- generate sequence numbers
- serialize or deserialize data
- interpret payloads
- manage application state
- implement business workflows

Any component that produces or consumes WAL records is considered **external**
to this repository.

---

## Key Characteristics

- Exactly one writer thread (no concurrent writes)
- Append-only log structure
- Total ordering enforced via sequence numbers
- Batch-based writes with atomic commit semantics
- Crash-safe recovery backed by explicit metadata
- User-defined, opaque binary payloads
- Explicit, user-controlled truncation

---

## What WAL-CPP Is

- A durability and ordering layer
- A source of truth for ordered records
- A deterministic replay mechanism

## What WAL-CPP Is Not

- A database
- A serializer or schema management system
- A business logic engine
- A message broker

---

## WAL Architecture

<p align="center">
  <img src="docs/High-level-wal-architecture.png" width="600">
</p>

This diagram represents the WAL **in isolation**.

External inputs and outputs are shown only to define the **interface boundary**
of the WAL. WAL-CPP accepts ordered binary records, persists them durably, and
replays them deterministically. It does not interpret payloads or depend on
external system semantics.

At a high level:

1. Ordered binary records are submitted to the WAL
2. Each record carries a strictly monotonic sequence number
3. Records are enqueued for persistence
4. A single WAL writer thread persists records and metadata
5. WAL provides deterministic replay of persisted records

---

## WAL Internal Pipeline

<p align="center">
  <img src="docs/WAL-Internals.png" width="550">
</p>

The WAL writer thread performs the following steps:

1. Drain the ingress queue
2. Build record batches
3. Append records to the active segment
4. fsync the segment file
5. Update WAL metadata
6. fsync the metadata file

A batch is considered **committed only after metadata has been fsynced**.

---

## WAL Record Layout

<p align="center">
  <img src="docs/WAL-Record-Layout.png" width="450">
</p>

Each WAL record consists of:

- A WAL-owned header (sequence number, sizes, versions, checksums)
- A user-defined, opaque binary payload

The WAL **never interprets or deserializes** the payload.

---

## Recovery and Truncation

<p align="center">
  <img src="docs/Recovery&Truncation-Flow.png" width="550">
</p>

- WAL is able to recover itself without external logic
- Replay start position and filtering are fully user-controlled
- WAL never deletes data unless explicitly instructed via a safe sequence

---

## Design Documentation

Detailed design documentation is available here:  
[docs/design.md](docs/design.md)

---

## Design Invariants

The following invariants are non-negotiable:

- WAL has exactly one writer thread
- WAL records are totally ordered by sequence number
- Sequence numbers are WAL-owned, not user-defined
- User payloads are opaque to WAL
- WAL never truncates data without an explicit safe sequence
- Recovery is deterministic and repeatable

---

## Status

This repository follows a **design-first approach**.  
Public APIs and implementation will be added once the design is fully locked.
