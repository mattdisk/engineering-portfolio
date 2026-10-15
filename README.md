# Engineering Portfolio

A collection of engineering projects spanning systems programming, distributed systems, high-performance computing, cloud and network infrastructure, security, embedded systems, and machine learning. Each project page reproduces that project's own README, covering its design, implementation, testing, and known limitations.

## Source Code Access

Source code for selected projects is publicly available through their respective GitHub repositories, linked below. Other projects are maintained in a private source archive, with project pages describing their design and implementation.

Recruiters and hiring managers interested in reviewing the implementations are welcome to contact me to request read-only access to the archive, subject to applicable sharing restrictions.

## Projects by area

### [Systems Programming](systems-programming/)

Software that works directly against operating-system and hypervisor interfaces: processes and job control, POSIX threads, sockets, inter-process communication, and host-side resource management.

| Project | Summary | Source |
|---|---|---|
| [AiShell](systems-programming/AiShell.md) | Interactive Unix shell in C99 with a recursive-descent parser, AST executor, and job control, plus a human-confirmed AI command-proposal mode backed by an embedded MCP server. | [Public repository](https://github.com/mattdisk/AiShell) |
| [multithreaded-file-server-c](systems-programming/multithreaded-file-server-c.md) | TCP socket programs, a custom Getfile file-transfer protocol library, and a boss-worker multithreaded client and server using POSIX threads. | On request |
| [webproxy-cache-c](systems-programming/webproxy-cache-c.md) | Multithreaded HTTP proxy built on libcurl, paired with a cache daemon that communicates over POSIX message queues, shared memory, and semaphores. | On request |
| [vm-resource-coordinator](systems-programming/vm-resource-coordinator.md) | Host-side daemons that re-pin KVM vCPUs to balance physical-core load and adjust guest memory through ballooning, driven by libvirt statistics. | On request |

### [Distributed Systems](distributed-systems/)

Multi-process systems that coordinate over RPC: task scheduling across workers, request fan-out and aggregation, and cache consistency across clients and a server.

| Project | Summary | Source |
|---|---|---|
| [cpp-grpc-mapreduce](distributed-systems/cpp-grpc-mapreduce.md) | MapReduce runtime in C++ in which a master schedules map and reduce tasks across workers over asynchronous gRPC, with input sharding, hash partitioning, and deadline-based task re-queuing. | On request |
| [grpc-price-aggregator](distributed-systems/grpc-price-aggregator.md) | Asynchronous gRPC store service that fans each query out to multiple vendor services in parallel, built on a completion-queue event loop and a custom thread pool. | On request |
| [grpc-distributed-file-system](distributed-systems/grpc-distributed-file-system.md) | AFS-inspired distributed file system in C++14: gRPC streaming file service, whole-file client caching, single-writer locks, and asynchronous server-push synchronization. | On request |

### [High-Performance Computing](high-performance-computing/)

Parallel algorithms and synchronization for shared-memory, distributed-memory, and GPU execution, with attention to communication and memory-hierarchy costs.

| Project | Summary | Source |
|---|---|---|
| [openmp-mpi-barriers](high-performance-computing/openmp-mpi-barriers.md) | Sense-reversing, tournament, dissemination, and MCS-style barrier algorithms in OpenMP, MPI, and a hybrid MPI/OpenMP form, with scaling experiments. | On request |
| [high-performance-algorithms](high-performance-computing/high-performance-algorithms.md) | Parallel list ranking (OpenMP, CUDA), MPI collectives built from point-to-point messages, a TeraSort-style distributed sort, and a cache-oblivious funnel sort. | On request |

### [Cloud & Network Infrastructure](cloud-network-infrastructure/)

Control planes and cloud pipelines that program infrastructure: software-defined networking, containerized network functions, and an event-driven serverless scanning service.

| Project | Summary | Source |
|---|---|---|
| [nfv-service-chain-orchestrator](cloud-network-infrastructure/nfv-service-chain-orchestrator.md) | Single-node NFV control plane that deploys containerized Firewall and NAT chains, programs Open vSwitch over OpenFlow, and balances connections across instances with connection affinity. | On request |
| [sdn-traffic-engineering-controller](cloud-network-infrastructure/sdn-traffic-engineering-controller.md) | OpenFlow 1.3 controller with shortest-path and widest-path routing, plus adaptive per-flow rerouting and redistribution driven by live bandwidth monitoring. | On request |
| [vulnscan](cloud-network-infrastructure/vulnscan.md) | Event-driven vulnerability-scanning pipeline for GitHub repositories on Azure Service Bus and Function Apps, running Semgrep, Trivy, and Gitleaks and storing results in PostgreSQL. | On request |

### [Security & Reverse Engineering](security-reverse-engineering/)

Low-level security work on native binaries: designing an exploitation challenge, developing exploits across vulnerability classes and architectures, and building program-analysis tools for malware.

| Project | Summary | Source |
|---|---|---|
| [toupper-pwn](security-reverse-engineering/toupper-pwn.md) | 32-bit Linux binary-exploitation challenge designed and built for a CTF competition, with containerized deployment and a reference exploit. | [Public repository](https://github.com/mattdisk/toupper-pwn) |
| [catch_the_flag](security-reverse-engineering/catch_the_flag.md) | Exploit development across x86, x86-64, ARM, and RISC-V targets: reverse engineering, shellcode, ROP, format-string, heap, and logic and race-condition vulnerabilities. | On request |
| [malware-program-analysis](security-reverse-engineering/malware-program-analysis.md) | Static (Ghidra, P-code) and dynamic (Intel Pin, NetworkX) analysis tools that produce control-flow, data-dependence, and control-dependence graphs for a Windows malware sample. | On request |

### [Embedded Systems & Edge AI](embedded-edge-ai/)

Software and hardware designed around physical device limits: an FPGA peripheral with a bare-metal driver, and a quantized neural network running on a microcontroller camera board.

| Project | Summary | Source |
|---|---|---|
| [FPGA_crc8_accelerator](embedded-edge-ai/FPGA_crc8_accelerator.md) | Custom VHDL CRC-8 peripheral on AXI4-Lite, integrated with a MicroBlaze soft processor on an Artix-7 board and validated against a software reference in bare-metal C. | [Public repository](https://github.com/mattdisk/FPGA_crc8_accelerator) |
| [nicla-vision-waste-classifier](embedded-edge-ai/nicla-vision-waste-classifier.md) | Six-class waste image classifier quantized to int8 TensorFlow Lite and run live on an Arduino Nicla Vision under OpenMV MicroPython. | [Public repository](https://github.com/mattdisk/nicla-vision-waste-classifier) |

### [Machine Learning](machine-learning/)

Empirical machine-learning studies that compare model families under controlled conditions and analyze why they behave as they do.

| Project | Summary | Source |
|---|---|---|
| [machine-learning-experiments](machine-learning/machine-learning-experiments.md) | Empirical studies in supervised, unsupervised, and reinforcement learning, with leakage-aware preprocessing, controlled hyperparameter sweeps, and analysis of unexpected results. | On request |

## Public repositories

- [AiShell](https://github.com/mattdisk/AiShell): [project page](systems-programming/AiShell.md)
- [toupper-pwn](https://github.com/mattdisk/toupper-pwn): [project page](security-reverse-engineering/toupper-pwn.md)
- [FPGA_crc8_accelerator](https://github.com/mattdisk/FPGA_crc8_accelerator): [project page](embedded-edge-ai/FPGA_crc8_accelerator.md)
- [nicla-vision-waste-classifier](https://github.com/mattdisk/nicla-vision-waste-classifier): [project page](embedded-edge-ai/nicla-vision-waste-classifier.md)
