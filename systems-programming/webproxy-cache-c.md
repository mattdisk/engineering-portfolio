# Multithreaded Web Proxy and Shared Memory Cache in C

A two-part systems project implementing a multithreaded HTTP proxy and a companion cache daemon that communicate via POSIX shared memory IPC.

---

## Overview

The project is split across two components, each in its own directory:

- **`server/`** — A multithreaded proxy server. Accepts requests using a custom GetFile protocol, fetches the requested resource from an HTTP origin server via libcurl, and streams the response back to the client in chunks. Uses a boss-worker thread pool.

- **`cache/`** — An extended version of the proxy paired with a standalone cache daemon (`simplecached`). Instead of fetching from the web, the proxy checks whether the file is available locally by communicating with the cache daemon over POSIX IPC (message queues + shared memory + semaphores).

Architecture diagrams are in `docs/part1.png` and `docs/part2.png`.

---

## Key Technical Features

**Part 1 — Proxy (`server/`)**

- Boss-worker multithreaded server using the provided `gfserver` framework
- libcurl integration via the "easy" C interface
- HEAD-first design: sends an initial request with `CURLOPT_NOBODY` set to probe for file existence and retrieve content length before committing to a full download, allowing early error reporting without downloading body content
- Range-based chunked streaming using `CURLOPT_RANGE` for segmented transfer; avoids loading the full file into memory
- 404 mapped to `GF_FILE_NOT_FOUND`; other curl failures mapped to `GF_ERROR`
- Signal handling for `SIGINT` and `SIGTERM` with clean shutdown

**Part 2 — Proxy + Cache (`cache/`)**

- Separate proxy process and cache daemon communicate without sockets, using only POSIX IPC
- **Command channel**: a named POSIX message queue (`/main_mq`) carries request metadata from proxy workers to the cache daemon's boss thread
- **Data channel**: named POSIX shared memory segments carry file content; one segment per proxy worker thread
- **Synchronization**: a pair of named POSIX semaphores per segment (`sem_write` / `sem_read`) enforces producer-consumer ordering between the cache worker writing a chunk and the proxy worker reading it
- **PID-scoped naming**: all IPC objects are named with the proxy's PID (e.g., `/proxy<pid>_shm_<n>`) so that multiple simultaneous proxy instances do not collide
- **Per-segment response queues**: each proxy worker has its own named message queue for receiving the cache's response, avoiding any need to lock the main command queue during a transfer
- **Pool of shared segments**: the proxy pre-allocates `nsegments` shared memory segments at startup and manages them through a thread-safe steque (stack-and-queue); workers acquire a segment, perform a transfer, then return it to the pool
- IPC resource cleanup on `SIGTERM`/`SIGINT`
- Proxy retries connecting to the cache daemon's command queue if it is not yet available, so the two processes can start in either order

---

## Technical Skills Demonstrated

- C systems programming (POSIX, Linux)
- POSIX threads (`pthreads`): mutex, condition variable, boss-worker pattern
- POSIX IPC: `shm_open`, `mmap`, `sem_open`, `mq_open`/`mq_send`/`mq_receive`
- libcurl easy interface: `CURLOPT_NOBODY`, `CURLINFO_RESPONSE_CODE`, `CURLINFO_CONTENT_LENGTH_DOWNLOAD`, `CURLOPT_RANGE`, `CURLOPT_WRITEFUNCTION`
- Signal handling: `SIGINT`, `SIGTERM`, graceful IPC cleanup
- Inter-process communication design: separating command channel from data channel, ensuring one-to-one worker pairing, PID-scoped unique naming

---

## Repository Layout

```
.
├── docs/
│   ├── part1.png          # Part 1 architecture diagram
│   └── part2.png          # Part 2 architecture diagram
├── server/                # Part 1: curl-backed web proxy
│   ├── Makefile
│   ├── handle_with_curl.c # libcurl request handler (core implementation)
│   ├── webproxy.c         # main: argument parsing, thread pool setup, signal handling
│   ├── proxy.h            # shared header (stub)
│   ├── handle_with_file.c # provided reference implementation using local files
│   ├── gfserver.h         # provided GetFile server framework header
│   ├── gfserver.o         # provided pre-compiled framework object
│   ├── gfserver_noasan.o  # provided pre-compiled framework object (no AddressSanitizer)
│   ├── steque.[ch]        # provided stack-and-queue data structure
│   ├── gfclient_download  # provided workload generator (downloads files)
│   ├── gfclient_measure   # provided workload generator (measures throughput)
│   └── workload.txt       # sample file paths for workload generators
└── cache/                 # Part 2: proxy + shared-memory cache daemon
    ├── Makefile
    ├── handle_with_cache.c # proxy-side IPC handler (core implementation)
    ├── webproxy.c          # main: segment pool setup, IPC init, signal handling
    ├── simplecached.c      # cache daemon: boss-worker, message queue, shared memory writes
    ├── ipc_types.h         # shared type definitions for IPC structs
    ├── shm_channel.[ch]    # IPC channel stubs (compiled as part of build)
    ├── simplecache.[ch]    # provided static cache implementation
    ├── gfserver.h          # provided framework header
    ├── gfserver.o          # provided pre-compiled framework object
    ├── gfserver_noasan.o   # provided pre-compiled framework object (no AddressSanitizer)
    ├── steque.[ch]         # provided stack-and-queue data structure
    ├── gfclient_download   # provided workload generator
    ├── gfclient_measure    # provided workload generator
    ├── cached_files/       # sample files served by the cache daemon
    ├── locals.txt          # maps request path keys to local file paths for simplecache
    └── workload.txt        # sample request paths for workload generators
```

---

## Dependencies and Environment

This project targets Linux. It will not build or run on macOS or Windows without modification.

| Dependency | Notes |
|---|---|
| GCC | C99 or later (`--std=gnu99`) |
| libcurl | Required for Part 1. Install via `apt install libcurl4-openssl-dev` or equivalent. |
| pthreads | Standard on Linux (`-lpthread`) |
| POSIX real-time library | `-lrt`, standard on Linux |
| AddressSanitizer | Used in the default build targets; requires a GCC/Clang build with asan support |

The provided `gfserver.o` and `gfserver_noasan.o` objects are pre-compiled ELF 64-bit Linux binaries. They cannot be recompiled from source (source is not included) and must be present for the linker to succeed.

---

## Build

Each part builds independently from its own directory.

**Part 1 (`server/`):**

```sh
cd server
make
```

Produces `webproxy` (with AddressSanitizer) and `webproxy_noasan`.

To build only the non-asan variant:

```sh
make noasan
```

**Part 2 (`cache/`):**

```sh
cd cache
make
```

Produces `webproxy` and `simplecached` (with AddressSanitizer), plus `webproxy_noasan` and `simplecached_noasan`.

---

## Run

### Part 1 — Proxy

Start the proxy, pointing it at an HTTP file server:

```sh
./webproxy -p <port> -t <threads> -s <server-url-prefix>
```

| Flag | Description | Default |
|---|---|---|
| `-p` | Listen port | 16642 |
| `-t` | Worker thread count (1–80) | 8 |
| `-s` | URL prefix for origin server | GitHub image data repo |

Then use the provided workload generator to send requests:

```sh
./gfclient_download -p <port> -w <threads> -r <requests> workload.txt
./gfclient_measure  -p <port> -w <threads> -r <requests> workload.txt
```

### Part 2 — Proxy + Cache Daemon

The cache daemon and proxy are separate processes that communicate via IPC. They can be started in either order.

**Start the cache daemon:**

```sh
cd cache
./simplecached -t <threads> -c locals.txt
```

| Flag | Description | Default |
|---|---|---|
| `-t` | Worker thread count (1–175) | 10 |
| `-c` | Path to the cache map file | `locals.txt` |
| `-d` | Artificial delay in microseconds (0–2500000) | 0 |

**Start the proxy:**

```sh
./webproxy -p <port> -t <threads> -n <segments> -z <segment-size>
```

| Flag | Description | Default |
|---|---|---|
| `-p` | Listen port | 25462 |
| `-t` | Worker thread count (1–200) | 8 |
| `-n` | Number of shared memory segments in the pool | 8 |
| `-z` | Size of each shared memory segment in bytes (min 824) | 5712 |

**Send requests:**

```sh
./gfclient_download -p <port> -w <threads> -r <requests> workload.txt
```

The paths in `workload.txt` are sample request keys (e.g., `/files/paraglider.jpg`). For Part 2, each key must have a matching entry in `locals.txt` that maps it to a file under `cached_files/`. For Part 1, when using an external origin server, the origin must expose paths that match what is in `workload.txt`.

---

## Notes and Limitations

- **Platform**: Requires Linux. POSIX shared memory (`shm_open`) and POSIX message queues (`mq_open`) are used throughout; these are not available on macOS or Windows without compatibility layers.
- **Message queue limits**: Linux caps `/proc/sys/fs/mqueue/msg_max` at 10 by default. The implementation uses `mq_maxmsg = 5` for the main command queue and `mq_maxmsg = 1` for per-segment response queues, staying within system defaults. Exceeding these limits produces unpredictable behavior.
- **Provided objects**: `gfserver.o` and `gfserver_noasan.o` (in both `server/` and `cache/`) are pre-compiled Linux ELF objects implementing the GetFile server framework. They are required for linking and are tracked in the repository because no source is available to rebuild them.
- **Part 2 does not fall back to the web**: the `cache/` proxy only queries the local cache daemon; it does not forward cache misses to an HTTP server. Files not mapped in `locals.txt` result in a `GF_FILE_NOT_FOUND` response.
- **IPC object naming**: all shared memory, semaphore, and message queue names are scoped to the proxy's PID (e.g., `/proxy12345_shm_1`). Ungraceful termination (e.g., `kill -9`) may leave IPC objects in `/dev/shm` and `/dev/mqueue`; these can be cleaned up manually or by rebooting.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
