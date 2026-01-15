# WAL-CPP Design

This document defines the **architecture, responsibilities, and invariants**
of WAL-CPP.

WAL-CPP is designed as a **reusable, minimal, correctness-first**
write-ahead log. Its purpose is to provide durable ordering and deterministic
recovery without assuming any application-level semantics.

---

## 1. Design Goals

WAL-CPP is designed to:

- provide a single, totally ordered log
- guarantee durability and crash safety
- enable deterministic replay
- remain reusable across domains
- avoid coupling to business logic

---

## 2. Architectural Boundary

![Architecture](High-level-wal-architecture.png)

WAL-CPP is designed and documented **in isolation**.

Any component that produces or consumes WAL records is considered external.
Such components are shown only to clarify the **contractual boundary** of the WAL.

WAL-CPP:

- accepts ordered binary records
- persists them durably
- replays them deterministically

WAL-CPP does not interpret, validate, or act upon record contents.

---

## 3. Single-Writer Model

WAL-CPP operates with **exactly one writer thread**.

This thread exclusively owns:

- file descriptors
- segment rotation
- fsync operations
- metadata updates

This model avoids locking, guarantees strict ordering,
and simplifies recovery semantics.

---

## 4. Internal Pipeline

![Internal Pipeline](WAL-Internals.png)

The WAL writer thread executes the following steps in strict order:

1. Drain the ingress queue
2. Build a batch of records
3. Append the batch to the active segment file
4. fsync the segment file
5. Update WAL metadata
6. fsync the metadata file

Only after step 6 is a batch considered **durably committed**.

---

## 5. Record Format

![Record Layout](WAL-Record-Layout.png)

Each WAL record consists of two parts:

### WAL Header (WAL-owned)

- WAL format version
- Record type
- Schema version
- Sequence number
- Payload size
- Checksum

### Payload (Externally-owned)

- Arbitrary binary data
- Any encoding or structure
- Never inspected by WAL

The sequence number is part of the **WAL header**, not the payload.

---

## 6. Sequence Numbers

Sequence numbers provide:

- total ordering
- replay boundaries
- recovery guarantees

WAL-CPP validates and persists sequence numbers but does not generate them.
Sequence generation is explicitly **outside WAL scope**.

---

## 7. Recovery Model

![Recovery Flow](Recovery&Truncation-Flow.png)

On startup, WAL-CPP performs:

1. Read WAL metadata
2. Determine last committed sequence
3. Scan WAL segments sequentially
4. Validate headers and checksums
5. Stop at corruption or end-of-log

Recovered records are made available for deterministic replay.

---

## 8. Replay Model

Replay is **consumer-controlled**.

External consumers decide:

- replay start sequence
- record filtering
- payload decoding
- state reconstruction

WAL-CPP guarantees ordering, integrity, and completeness only.

---

## 9. Truncation and Retention

WAL-CPP does not decide when data becomes irrelevant.

Truncation requires an explicit **safe sequence boundary**, typically provided
after external state has been safely materialized.

Rules:

- Only sealed segments may be deleted
- Partial segment deletion is forbidden
- WAL never infers truncation safety

---

## 10. Design Invariants

The following invariants must always hold:

- Exactly one WAL writer thread
- Append-only log structure
- Total ordering by sequence number
- Payloads are opaque to WAL
- Metadata defines commit truth
- No truncation without explicit safety proof
- Deterministic recovery and replay

---

## 11. Non-Goals

WAL-CPP explicitly does not:

- manage schemas
- serialize or deserialize data
- enforce business rules
- coordinate workflows
- provide messaging semantics

---

## 12. Future Extensions

Potential extensions include:

- multiple WAL instances per shard
- configurable fsync policies
- alternative durability modes
- tooling and inspection utilities

All extensions must preserve existing invariants.
