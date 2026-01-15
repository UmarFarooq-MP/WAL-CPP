# WAL-CPP Design

## Overview

WAL-CPP is a single-writer, append-only write-ahead log designed for
deterministic recovery and replay.

## Architecture

![Architecture](High-level-wal-architecture.png)

## Internal Pipeline

![Pipeline](WAL-Internals.png)

## Record Format

![Record Layout](WAL-Record-Layout.png)

## Recovery and Retention

![Recovery](Recovery&Truncation-Flow.png)
