# Distributed File System Case Study

## Summary

This project implements a distributed file system (DFS) in C++14 on top of gRPC and Protocol Buffers. It is built in two layers:

- **Part 1** — a synchronous RPC file service that streams files between a client and a server (fetch, store, list, stat, delete), with explicit handling of RPC deadlines and failure statuses.
- **Part 2** — a weakly consistent distributed file system built on top of that RPC layer, inspired by the Andrew File System (AFS): clients cache whole files locally, a single-writer lock protocol coordinates concurrent writes, and an asynchronous server-push channel keeps multiple clients and a server in sync as files are added, modified, or deleted.

This README and the accompanying design notes summarize the system architecture, engineering decisions, and testing approach.

## What This Project Demonstrates

- **Systems programming in C++14** — multithreaded client and server processes built around a formal RPC interface rather than an ad-hoc protocol.
- **RPC and protocol design** — gRPC and Protocol Buffers, including streaming RPCs for file transfer and explicit handling of deadlines and failure statuses.
- **Concurrency** — a filesystem watcher thread and an asynchronous network thread cooperating on the client; server-side state (locks, change tracking) synchronized across concurrent requests.
- **Distributed-systems reasoning** — cache consistency, write-lock coordination, and race conditions across independent client and server processes, plus the ability to explain the resulting tradeoffs (whole-file caching, checksum-based change detection, weak vs. strict consistency).
- **OS-level integration** — driving application behavior from filesystem events via Linux `inotify`.
- **Build tooling** — Makefiles and protobuf/gRPC code generation.

## Architecture Overview

Both parts follow a client/server model, communicating exclusively over gRPC:

```
        gRPC (sync + streaming)             gRPC (async, server-push)
Client(s) <───────────────────────► Server        Client(s) <──────────────► Server
   Part 1: request/response file operations           Part 2 adds: cache, locks, live sync
```

Part 2 layers two additional client-side threads and a server-side lock/notification mechanism on top of the Part 1 RPC surface — the sections below describe what each part implements, and docs/design-notes.md covers the underlying design reasoning and concurrency edge cases in depth.

## Part 1: Synchronous RPC File Service

Defines and implements a gRPC service exposing:

- **Store** — stream a file's contents from client to server
- **Fetch** — stream a file's contents from server to client
- **List** — list all files known to the server, with modification times
- **Stat** — return size and timestamp metadata (modification and metadata-change time) for a file
- **Delete** — remove a file from the server

The client is responsible for recognizing and handling gRPC deadline-exceeded responses, so that a slow or unresponsive server does not hang the client indefinitely.

## Part 2: Weakly Consistent Client Cache, Write Locks, and Async Server-Push Coordination

Extends the Part 1 service with:

- **Whole-file client-side caching** — clients operate on local copies of files and only communicate with the server when a file changes locally or a server-side change is detected.
- **Single-writer lock protocol** — a client must request a write lock from the server before storing a file; the server tracks which client (if any) currently holds the lock per file.
- **CRC-based change detection** — file identity/staleness is compared using a CRC checksum plus modification time, rather than transferring full file contents to detect drift.
- **Asynchronous server-push notifications** — clients maintain a standing async RPC so the server can proactively notify them of remote changes, instead of clients polling on a fixed interval.
- **Multi-client synchronization** — the design keeps the server and every connected client's mount directory converged, including coordinated handling of client-initiated deletions.

## Repository Structure

```
part1/            Part 1 implementation (synchronous RPC file service) and its Makefile
part2/            Part 2 implementation (DFS: caching, locks, async sync) and its Makefile
docs/             Design/reference material, including design-notes.md
mnt/, bin/, tmp/  Runtime/build scratch directories (mount points, build output, object files)
attributions.md   Third-party asset credits
```

## Build / Run / Test Notes

The project builds with `make` (root `Makefile` delegates to `part1/` and `part2/`), and depends on a C++14 toolchain plus gRPC and Protocol Buffers (via `pkg-config`). Protobuf/gRPC sources are generated from `.proto` definitions with `make protos`.

Each part produces a client and a server executable that communicate over gRPC on the local machine or over a network; the Part 2 client additionally mounts a local directory and watches it via `inotify`.

Testing during development was scenario-based: individual RPC methods were exercised directly (including failure cases such as a missing file, a mid-transfer interruption, and large-file transfer), and the Part 2 synchronization logic was exercised with a single client, then with multiple concurrent clients, verifying convergence between client and server state. These results reflect the original development record, not a fresh run — this documentation pass did not rebuild or execute the project.

## Limitations

- CRC32 is used for change detection; it is adequate for detecting incidental drift between a client and server copy but is not a cryptographic integrity guarantee.
- Locking is coarse-grained (a single lock table keyed by filename) rather than per-file lock objects — simpler to reason about, at the cost of some avoidable contention under heavy concurrent write load.
- Whole-file caching means any change to a file requires re-transferring the entire file, which is inefficient for large files with small edits.
- The system assumes a single server; there is no replication or failover if the server process goes down.
- Testing was scenario-based and manual rather than an automated regression suite.

## Future Improvements

- Block-level or delta-based caching instead of whole-file transfer.
- Finer-grained (per-file) locking instead of a single shared lock table.
- Server replication or a leader-election scheme for fault tolerance.
- An automated test suite (unit + integration) runnable in CI.
- Transport security (TLS) and authentication on the gRPC channel.

## References / Influences

- [gRPC C++ documentation](https://grpc.github.io/grpc/cpp/index.html)
- [Protocol Buffers language guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)
- [gRPC deadlines](https://grpc.io/blog/deadlines/)
- [Andrew File System (AFS)](https://en.wikipedia.org/wiki/Andrew_File_System) — the primary influence on the Part 2 weak-consistency / whole-file-caching design

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
