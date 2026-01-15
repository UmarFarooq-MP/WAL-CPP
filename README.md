# WAL-CPP

WAL-CPP is a **single-writer, append-only Write-Ahead Log (WAL)** designed for  
**deterministic recovery, crash safety, and ordered replay**.

It is intended to serve as a **foundational infrastructure primitive** for systems
that require strict ordering and durable state transitions, including:

- order matching engines
- trading and financial systems
- event-sourced architectures
- low-latency, stateful services

WAL-CPP is intentionally scoped to **durability and ordering only**.  
All business logic, data interpretation, and state reconstruction are handled by
the user of the library.

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
- A source of truth for ordered events
- A deterministic replay mechanism

## What WAL-CPP Is Not

- A database
- A serializer or schema management system
- A business logic engine
- A message broker

---

## High-Level Architecture

<p align="center">
  <img src="docs/High-level-wal-architecture.png" width="600">
</p>

At a high level:

1. The user or engine serializes domain data into binary form
2. A sequencer assigns a strictly monotonic sequence number
3. Records are submitted via a queue
4. A single WAL thread persists records and metadata
5. Replay is performed by user logic using WAL-provided iteration

---

## WAL Internal Pipeline

<p align="center">
  <img src="docs/WAL-Internals.png" width="550">
</p>

The WAL thread performs the following steps:

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

- WAL is able to recover itself without user-provided logic
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

This repository currently follows a **design-first approach**.  
Public APIs and implementation will be added once the design is fully locked.
