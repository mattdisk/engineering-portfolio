# Distributed Systems

Multi-process systems that coordinate over RPC: task scheduling across workers, request fan-out and aggregation, and cache consistency across clients and a server.

**Skills demonstrated:** C++ with gRPC and Protocol Buffers, asynchronous RPC with completion queues, thread pools, partial-failure handling, and consistency and locking protocols.

| Project | Summary | Source |
|---|---|---|
| [cpp-grpc-mapreduce](cpp-grpc-mapreduce.md) | MapReduce runtime in C++ in which a master schedules map and reduce tasks across workers over asynchronous gRPC, with input sharding, hash partitioning, and deadline-based task re-queuing. | Available on request |
| [grpc-price-aggregator](grpc-price-aggregator.md) | Asynchronous gRPC store service that fans each query out to multiple vendor services in parallel, built on a completion-queue event loop and a custom thread pool. | Available on request |
| [grpc-distributed-file-system](grpc-distributed-file-system.md) | AFS-inspired distributed file system in C++14: gRPC streaming file service, whole-file client caching, single-writer locks, and asynchronous server-push synchronization. | Available on request |

[Back to portfolio overview](../README.md)
