# Distributed MapReduce Runtime in C++ and gRPC

A from-scratch MapReduce execution engine in C++: a master process schedules map and reduce work across a pool of worker processes over gRPC, splitting input files by byte offset, partitioning intermediate data by key, and producing sorted, per-partition output files. It's a small, self-contained example of the kind of problem backend/distributed-systems work involves: coordinating multiple processes over RPC, partitioning and ordering data across them, and handling partial failure without a single component doing everything.

## What this demonstrates

- Designing a master/worker distributed task-scheduling loop around a gRPC service contract, using asynchronous RPCs and a completion queue instead of blocking calls.
- Splitting large inputs into shards by byte offset while respecting record (newline) boundaries, without physically copying the input files.
- A two-phase (map, then reduce) pipeline with disk-based intermediate storage and hash-based partitioning between the phases.
- Basic fault handling: per-RPC deadlines, worker-state tracking, and task re-queuing when a worker misses its deadline or reports failure.
- A pluggable user-code interface (`BaseMapper` / `BaseReducer`) so the runtime is decoupled from any specific map/reduce job.

## Technical highlights

| Area | Implementation |
| --- | --- |
| RPC layer | Single `WorkerControl.AssignTask` unary RPC (protobuf `oneof` carries either a `MapTask` or `ReduceTask` payload); see src/masterworker.proto |
| Scheduling | Master issues `AsyncAssignTask` calls and polls a `grpc::CompletionQueue`, dispatching to any worker in `AVAILABLE` state; see src/master.h |
| Sharding | Shard size derived from a `map_kilobytes` config value; shard boundaries snapped to the next `\n` so records are never split; see src/file_shard.h |
| Partitioning | Intermediate keys are routed to one of R partitions via `std::hash<string>(key) % R`; see `BaseMapperInternal::emit` in src/mr_tasks.h |
| Reduce ordering | Each reduce task accumulates key/value pairs in a `std::map`, so its output file is sorted by key; see `BaseReducerInternal` in src/mr_tasks.h |
| Fault handling | Worker states are `AVAILABLE` / `BUSY` / `DEAD`; a 6-second RPC deadline plus a failed reply mark a worker `DEAD` and re-queue its task; see `Master::run` in src/master.h |
| Cleanup | After the reduce phase, the master deletes every file in the output directory that isn't a final reduce output; see the cleanup loop in `Master::run` |

## Architecture overview

```
                 ┌─────────────────────┐
                 │  User program       │
                 │  (mrdemo)           │
                 └─────────┬───────────┘
                           │ MapReduce::run(config)
                 ┌─────────▼───────────┐
                 │  Master              │
                 │  - reads config      │
                 │  - shards input      │
                 │  - schedules M map   │
                 │    + R reduce tasks  │
                 └─────────┬───────────┘
                    gRPC (AssignTask)
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ Worker  │        │ Worker  │        │ Worker  │   (mr_worker processes,
   │ process │        │ process │        │ process │    one per ip:port)
   └─────────┘        └─────────┘        └─────────┘
```

The master is not a separate process; it runs inside the `mrdemo` binary and is driven by `MapReduce::run()`. Workers are separate `mr_worker` processes, each hosting a gRPC server that the master calls into.

**Map phase.** The master computes `M` file shards from the configured `map_kilobytes` size and dispatches one `MapTask` per shard to an available worker. The worker reads its byte range, splits it into records on `\n`, and feeds each record to the registered `BaseMapper::map()`. The user's `emit()` calls are buffered in memory and, on finalize, written to up to `R` intermediate files (one per partition, named `map_<task_id>_<partition_id>_<worker_addr>`), skipping any partition that received no keys.

**Reduce phase.** Once every map task has completed, the master dispatches `R` reduce tasks, each carrying the list of intermediate files assigned to that partition. The worker reads and groups those files by key, calls the registered `BaseReducer::reduce()` once per key, and writes a single sorted output file (`reduce_<task_id>_0_<worker_addr>`).

**Cleanup.** Intermediate and final files are written into the same output directory. After the reduce phase, the master removes every file in that directory that isn't one of the recorded final reduce outputs.

## Repository structure

```
project4/
├── CMakeLists.txt          top-level build: gRPC/Protobuf/Threads, subdirectories below
├── external/                gRPC-facing public interface exposed to user code
│   └── include/              mapreduce.h, mr_task_factory.h
├── src/                     runtime implementation
│   ├── masterworker.proto    gRPC service + message definitions
│   ├── file_shard.h          byte-offset sharding aligned to newlines
│   ├── mapreduce_spec.h      config file parsing/validation
│   ├── master.h               scheduling loop, worker pool, fault handling
│   ├── worker.h                gRPC service implementation run by each worker process
│   ├── mr_tasks.h              mapper/reducer internal buffering, partitioning, output
│   ├── mapreduce.cc / .h, mapreduce_impl.cc / .h   thin entry points wiring the above together
│   ├── mr_task_factory.cc      user mapper/reducer registration
│   └── run_worker.cc           `mr_worker` process entry point
├── test/                    sample job: word count over three text inputs
│   ├── config.ini             job configuration (workers, inputs, shard size, etc.)
│   ├── user_tasks.cc           word-count `BaseMapper` / `BaseReducer` implementation
│   ├── main.cc                 `mrdemo` process entry point
│   └── input/                  sample input text files
└── docs/
    └── design-notes.md      engineering write-up: tradeoffs, failure modes, limitations
```

## Build instructions

Requirements: CMake 3.10+, a C++17 compiler, Protobuf, and gRPC (with the `grpc_cpp_plugin` protoc plugin) discoverable via `find_package`.

Out-of-source build:

```bash
cmake -S . -B build
cmake --build build
```

This produces two binaries under `build/bin/`: `mrdemo` (the user-facing driver/master) and `mr_worker` (the worker process). A symlink to `test/config.ini` is installed alongside them as `build/bin/config.ini`.

> The project also supports the traditional in-source `cmake .` / `make` flow; out-of-source
> is recommended to keep generated files out of the source tree.

## Run instructions

```bash
cd build/bin
cp -r <repo-root>/test/input .        # sample input files
mkdir -p output

# start one worker per configured ip:port (config.ini defaults to 6 local ports)
./mr_worker localhost:50051 &
./mr_worker localhost:50052 &
# ... one per address listed in config.ini's worker_ipaddr_ports

./mrdemo config.ini
```

`config.ini` controls the job: input files, number of workers and their addresses, number of output partitions (`n_output_files`), and the approximate shard size in KB (`map_kilobytes`). The sample job in `test/` is a word count over `test/input/*.txt`.

On completion, `output/` contains one `reduce_*` file per partition, each holding `key value` lines sorted by key. Kill the worker processes once `mrdemo` exits.

## Testing / verification status

This repository was statically reviewed for portfolio presentation. It was not freshly built or run in this cleanup pass because the required C++/CMake/gRPC/Protobuf toolchain was unavailable.

- There is no automated test suite in the repository; `test/` is a runnable sample job, not a unit/integration test harness.
- Anyone building this should treat the build/run instructions above as derived from static inspection of `CMakeLists.txt` and the source, not as freshly confirmed working commands.

## Limitations

- **Partition-file assumption.** A map worker always reports `n_output_files` intermediate filenames to the master, one per partition, even for partitions that received no keys (in which case the file is never actually written — see `BaseMapperInternal::finalize` in src/mr_tasks.h). A reduce worker that is handed a filename for an empty partition will fail to open it. This is unlikely to trigger on reasonably sized inputs but is a latent correctness gap for small inputs or a large partition count. Not exercised by any test in this repository.
- **Worker failure is permanent for the job.** A worker that misses its 6-second RPC deadline once, for any reason (including transient slowness), is marked `DEAD` and never reused for the rest of the run — there is no recovery path back to `AVAILABLE`.
- **No merge across reduce partitions.** Each reduce task's output file is sorted internally; the runtime does not merge the `R` output files into one globally sorted file.
- **Single-machine deployment model.** Workers are meant to run as separate local processes on different ports, per the original project scope, rather than on physically separate hosts — reachability is whatever `ip:port` values are supplied in `config.ini`.
- **Symlinked config on non-POSIX filesystems.** The CMake config-file install step creates a symlink; this can require elevated permissions or developer-mode symlink support on Windows.

## Possible future improvements

- Allow a `DEAD` worker to be re-probed and returned to the pool instead of being permanently excluded.
- Have the reduce phase tolerate a missing intermediate file for an empty partition instead of failing.
- Add an automated test harness that runs a small job end-to-end and checks output against expected key/value pairs.
- Optionally merge per-partition reduce outputs into a single sorted file for jobs where that is more convenient than per-partition files.

## References

- Dean, J. and Ghemawat, S., ["MapReduce: Simplified Data Processing on Large Clusters"](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf), OSDI 2004 — the paper this project's simplified model is based on.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
