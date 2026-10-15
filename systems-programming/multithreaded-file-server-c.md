# Multithreaded File Server in C

A set of C systems programming projects implementing TCP socket communication, a custom file transfer protocol (Getfile), and a multithreaded server and client using POSIX pthreads. Built with GCC and GNU Make targeting POSIX/Linux environments.

---

## Projects

### Echo Client/Server (`echo/`)

A TCP echo pair demonstrating the BSD socket API: `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`, and `getaddrinfo`. The server loops indefinitely, echoing each message back to the client. Supports both IPv4 and IPv6. Messages are limited to 15 bytes, and no null termination is assumed on either side.

### File Transfer (`transfer/`)

A TCP server that opens a file specified on the command line and streams its contents to a connected client in chunks, closing the socket when the transfer is complete. The client saves the received bytes to a local file. Implements a correct chunked `send`/`recv` loop that handles partial transfers without assuming a fixed message size or a pre-communicated length.

### Getfile Protocol Library (`gflib/`)

Implements a custom HTTP-inspired protocol over TCP called Getfile. A request has the form:

```
GETFILE GET <path>\r\n\r\n
```

A response has the form:

```
GETFILE <status> <length>\r\n\r\n<content bytes>
```

Valid status values: `OK`, `FILE_NOT_FOUND`, `ERROR`, `INVALID`.

The library is split into a client side (`gfclient.c`) and a server side (`gfserver.c`), each exposing a callback-based C API:

- **Client** (`gfclient.c`): callers register a write callback and optionally a header callback, then call `gfc_perform()`. The `gfcrequest_t` type is an opaque pointer, keeping internal state hidden from callers. Inspired by libcurl's easy interface.
- **Server** (`gfserver.c`): callers register a handler callback and call `gfserver_serve()`, which loops indefinitely. The handler uses `gfs_sendheader()` and `gfs_send()` to respond without any socket-level code in the handler itself. The server implementation in this library is a stub; `gfserver_main` in `gflib/` depends on a prebuilt handler object described in Limitations below.

### Multithreaded Getfile (`mtgf/`)

Extends the Getfile server and client with a boss-worker thread pool on both sides.

**Server**: the main (boss) thread accepts connections via `gfserver_serve()` and dispatches each request to a shared work queue. A configurable pool of worker threads (`-t`) blocks on the queue, picks up work items, resolves the requested path via the content library, and sends the file back using `gfs_sendheader()` and `gfs_send()`. The handler callback, work queue, mutex, and condition variable are wired together in `gfserver_main.c` and `handler.c`. The `gfserver_internal.h` header defines the `work_item_t` and `worker_args_t` structs shared between those two files.

**Client**: the boss thread reads request paths from `workload.txt` and enqueues them into a shared steque. A pool of worker threads (`-t`) consumes requests and performs downloads via `gfc_perform()`, saving each file locally. The boss signals all workers after all requests are enqueued and joins them before exiting.

Both sides use `pthread_mutex_t` and `pthread_cond_t` for synchronization and `steque` (`steque.c/h`) as the shared work queue.

---

## Key Technical Concepts

- POSIX socket API — TCP client/server, `getaddrinfo`, `SO_REUSEADDR`, chunked `send`/`recv`
- Custom application-layer protocol design and header parsing without assuming null termination
- POSIX pthreads — `pthread_create`, `pthread_join`, `pthread_mutex_t`, `pthread_cond_t`
- Boss-worker (producer-consumer) thread pool pattern
- Callback-based C library API design using opaque pointers and function pointers
- AddressSanitizer integration (`-fsanitize=address`) for memory safety validation
- GNU Make with separate ASAN and no-ASAN build targets (`make` vs `make all_noasan`)

---

## Repository Layout

```
multithreaded-file-server-c/
├── echo/
│   ├── echoclient.c              TCP echo client
│   ├── echoserver.c              TCP echo server (loops indefinitely)
│   └── Makefile
├── transfer/
│   ├── transferclient.c          Receives a file from the server and saves to disk
│   ├── transferserver.c          Streams a file to each connected client
│   └── Makefile
├── gflib/                        Single-threaded Getfile library
│   ├── gfclient.c                Client library implementation
│   ├── gfclient.h                Client library public API
│   ├── gfserver.c                Server library (stub — see Limitations)
│   ├── gfserver.h                Server library public API
│   ├── gf_shared.c/h             System includes shared by client and server
│   ├── gfclient_internal.h       Internal client header
│   ├── gfserver_internal.h       Internal server header
│   ├── gfclient_download.c       Single-threaded client driver
│   ├── gfserver_main.c           Server driver
│   ├── content.c/h               Maps virtual request paths to on-disk files
│   ├── workload.c/h              Reads request paths from workload.txt
│   ├── content.txt               Virtual-to-disk path mapping for the content library
│   ├── workload.txt              Request paths for the workload generator
│   ├── server_root/              Static file corpus served by the server
│   └── Makefile
├── mtgf/                         Multithreaded Getfile server and client
│   ├── handler.c                 Boss enqueues work; shared work queue and handler callback
│   ├── gfserver_main.c           Spawns worker pool and drives the server loop
│   ├── gfclient_download.c       Multithreaded client with boss-worker pool
│   ├── gfclient.h                Client library public API
│   ├── gfserver.h                Server library public API
│   ├── gf_shared.c/h             System includes shared by client and server
│   ├── gfclient_internal.h       Internal client header
│   ├── gfserver_internal.h       Shared types (work_item_t, worker_args_t) and declarations
│   ├── steque.c/h                Stack/queue used as the shared work queue
│   ├── content.c/h               Content mapping library
│   ├── workload.c/h              Workload generator library
│   ├── content.txt               Virtual-to-disk path mapping
│   ├── workload.txt              Request paths for the workload generator
│   ├── server_root/              Static file corpus served by the server
│   └── Makefile
└── illustrations/
    ├── gfserver.png              Server architecture diagram
    └── gftransfer.png            Protocol transfer flow diagram
```

---

## Environment and Dependencies

- **OS**: Linux (Ubuntu 20.04 x64 recommended; required for `mtgf/` prebuilt support objects — see Limitations)
- **Compiler**: GCC with C99/GNU99 support (`--std=gnu99`)
- **System libraries**: pthreads (`-lpthread`), AddressSanitizer (`-fsanitize=address`, `-static-libasan`)
- **Build tool**: GNU Make

The Makefiles detect Darwin and omit `-lpthread` and ASAN library flags automatically, but full ASAN support on macOS depends on the local toolchain.

---

## Building

Each component is built independently from its own subdirectory.

```sh
# Echo
cd echo && make

# File transfer
cd transfer && make

# Getfile library (single-threaded)
cd gflib && make

# Multithreaded Getfile
cd mtgf && make
```

The default `make` target builds with AddressSanitizer enabled. Use `make all_noasan` to produce binaries without ASAN (compatible with Valgrind).

The `clean` target removes built objects and binaries. In `gflib/`, the clean target deliberately preserves `handler.o` and `handler_noasan.o` when they exist. In `mtgf/`, it preserves the prebuilt protocol objects.

---

## Running

### Echo

```sh
# Terminal 1 — start server on port 48384
echo/echoserver -p 48384

# Terminal 2 — send a message
echo/echoclient -s localhost -p 48384 -m "hello"
```

### File Transfer

```sh
# Terminal 1 — serve a file
transfer/transferserver -p 38484 -f path/to/file.txt

# Terminal 2 — receive and save
transfer/transferclient -s localhost -p 38484 -o received.txt
```

### Getfile — single-threaded (`gflib/`)

```sh
# Terminal 1 — start server (default port 29384)
gflib/gfserver_main -p 29384 -m gflib/content.txt

# Terminal 2 — run client workload
gflib/gfclient_download -s localhost -p 29384 -n 14 -w gflib/workload.txt
```

### Getfile — multithreaded (`mtgf/`)

```sh
# Terminal 1 — start multithreaded server (default port 17394, default 14 worker threads)
mtgf/gfserver_main -p 17394 -t 14 -m mtgf/content.txt

# Terminal 2 — run multithreaded client (9 worker threads, 13 requests)
mtgf/gfclient_download -s localhost -p 17394 -t 9 -n 13 -w mtgf/workload.txt
```

The `content.txt` and `workload.txt` files reference paths under `server_root/content/static/filecorpus/`, which is included in the repository.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
