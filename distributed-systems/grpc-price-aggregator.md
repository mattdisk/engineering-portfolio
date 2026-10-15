# gRPC Price Aggregation Service

A distributed price-aggregation service built on asynchronous gRPC. A client queries a store for a product; the store fans the query out to multiple vendor services in parallel, waits for their bids, aggregates the results, and returns them to the client. The store itself is a fully asynchronous gRPC server backed by a custom thread pool, so it can accept and process many concurrent client requests without blocking its main event loop.

## What This Project Demonstrates

- Designing an asynchronous gRPC server around a completion-queue event loop, rather than the simpler synchronous request/response model
- Building an asynchronous gRPC client that fans a single incoming request out to many downstream services and collects all responses before replying
- Writing a thread pool from scratch (task queue, condition-variable signaling, clean shutdown) and integrating it into a request-handling pipeline
- Coordinating concurrent, stateless request handling across client-facing and downstream RPC boundaries

## Technical Highlights

- **Language:** C++
- **RPC / IDL:** gRPC and Protocol Buffers, with `.proto`-driven code generation
- **Server concurrency model:** asynchronous server-side RPC using a `grpc::ServerCompletionQueue` and an explicit per-request state machine (`CREATE` → `PROCESS` → `FINISH`)
- **Client concurrency model:** asynchronous client-side RPC (`grpc::CompletionQueue`) used to issue parallel calls to every vendor and block only until all of them complete
- **Concurrency primitive:** a custom thread pool (`std::thread`, `std::mutex`, `std::condition_variable`) that owns request processing so the server's polling thread is never blocked by vendor round-trips
- **Build system:** CMake, with a shared proto-generation module used by the store, test, and vendor targets

## Architecture Overview

Three roles are involved:

- **Client** (`test/client.cc`, driven by `test/run_tests.cc`) — sends product queries to the store over a synchronous stub and prints the aggregated bids it gets back.
- **Store** (`src/store.cc`, `src/store_service.*`) — the async gRPC server at the center of the system. It accepts client queries, hands each one to a thread pool, and that worker thread acts as an async gRPC *client* to the vendors.
- **Vendor** (`test/vendor.cc`, launched by `test/run_vendors.cc`) — a minimal gRPC server that returns a deterministic, hash-derived price for a given product name. Multiple vendor instances run independently, each on its own address.

```
client  --(store.proto: getProducts)-->  store  --(vendor.proto: getProductBid)-->  vendor 1
                                            |                                       vendor 2
                                            |------------------------------------->  vendor N
```

## Request Flow

1. The store's async server accepts a `ProductQuery` on its completion queue. A `CallData` object tracks the request through a `CREATE → PROCESS → FINISH` state machine, and a new `CallData` is immediately created to accept the next incoming request.
2. On `PROCESS`, the request is handed to the thread pool (`ThreadPool::Enqueue`) so the server's polling loop is free to keep accepting new connections.
3. The worker thread constructs a `VendorAsyncClient`, issues an async `getProductBid` call to every configured vendor in parallel, and blocks on its own completion queue until every vendor has responded (or failed).
4. Successful vendor replies are aggregated into a single `ProductReply` (vendor ID + price pairs).
5. The worker finishes the async response writer, which pushes the reply back onto the server's completion queue and completes the request (`FINISH`).

## Repository Structure

```
.
├── README.md
├── docs/
│   └── design-notes.md        # architecture and design rationale
├── CMakeLists.txt             # top-level build: protos, src, test
├── external/
│   ├── CMakeLists.txt         # generates C++ classes from .proto files
│   └── protos/
│       ├── store.proto        # client <-> store contract
│       └── vendor.proto       # store <-> vendor contract
├── src/
│   ├── CMakeLists.txt
│   ├── store.cc                    # store entry point (CLI parsing, startup)
│   ├── store_service.h/.cc         # async store server + per-request state machine
│   ├── vendor_async_client.h/.cc   # async fan-out client used against vendors
│   └── threadpool.h/.cc            # custom thread pool
└── test/
    ├── CMakeLists.txt
    ├── client.cc                 # synchronous store client used by run_tests
    ├── vendor.cc                 # minimal vendor server (hash-based pricing)
    ├── run_tests.cc              # concurrent client-load test harness
    ├── run_vendors.cc            # launches multiple vendor server instances
    ├── product_queries_util.h
    └── product_query_list.txt    # sample product names used by run_tests
```

## Build Instructions

This project requires a C++ toolchain, CMake (3.10+), and a locally installed gRPC + Protobuf (built with `gRPC_INSTALL=ON` so `find_package(gRPC CONFIG)` can locate it). See the [gRPC C++ quickstart](https://grpc.io/docs/languages/cpp/quickstart/) for building gRPC from source.

```bash
mkdir build && cd build
cmake ..
make -j4
```

This produces three binaries under `build/bin/`: `store`, `run_vendors`, and `run_tests`.

## Run / Demo Instructions

1. Create a vendor address file (one `ip:port` per line), e.g. `vendor_addresses.txt`.
2. Start the vendor servers, one thread per address in the file:
   ```bash
   ./bin/run_vendors vendor_addresses.txt &
   ```
3. Start the store, pointing it at the same vendor file, the address it should listen on for clients, and its thread pool size:
   ```bash
   ./bin/store vendor_addresses.txt 0.0.0.0:50053 8
   ```
4. Run the load-test client, which reads product names from `product_query_list.txt` and sends them concurrently:
   ```bash
   ./bin/run_tests 0.0.0.0:50053 4
   ```
   Output shows, for each queried product, the aggregated `(price, vendor_id)` pairs returned by every vendor that responded.

## Test Instructions

There is no separate unit-test suite. `run_tests` (`test/run_tests.cc`) is the project's test harness: it drives concurrent client requests against a running store instance and prints the aggregated results for manual inspection against the vendor servers' known (hash-derived) prices.

## Verification Status

- Static code review of all source, proto, and CMake files was completed as part of this cleanup pass.
- The build was **not** freshly compiled or run in this environment — the gRPC/Protobuf/CMake toolchain is not installed here. The build and run instructions above reflect the project's existing CMake configuration and command-line interface, not a freshly executed test run.
- No performance, scalability, or reliability numbers are reported anywhere in this repository; none should be inferred.

## Limitations

- The store blocks a worker thread until *all* vendors respond (or fail) to a given query; there is no per-vendor timeout or partial-result path, so one unresponsive vendor delays that request.
- No retry, circuit-breaking, or backoff logic for vendor RPC failures.
- No TLS/authentication — all channels use insecure credentials, which is appropriate for local/demo use only.
- Vendor pricing is a deterministic hash of vendor ID and product name, not real pricing data.
- Testing is limited to the manual `run_tests` harness; there is no automated assertion-based test suite.

## Future Improvements

- Add per-vendor timeouts and return partial results when some vendors don't respond in time.
- Add a real automated test suite (unit tests for the thread pool and aggregation logic, integration tests for the full request flow).
- Add TLS credentials and basic auth for non-local deployment.
- Add structured logging/metrics around request latency and thread pool queue depth.

## References

The asynchronous server/client patterns follow the general completion-queue model documented in gRPC's own C++ async examples; the thread pool is an original implementation for this project.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
