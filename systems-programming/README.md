# Systems Programming

Software that works directly against operating-system and hypervisor interfaces: processes and job control, POSIX threads, sockets, inter-process communication, and host-side resource management.

**Skills demonstrated:** C and POSIX APIs, concurrency with pthreads, socket and protocol design, IPC (message queues, shared memory, semaphores), and hypervisor control through libvirt.

| Project | Summary | Source |
|---|---|---|
| [AiShell](AiShell.md) | Interactive Unix shell in C99 with a recursive-descent parser, AST executor, and job control, plus a human-confirmed AI command-proposal mode backed by an embedded MCP server. | [github.com/mattdisk/AiShell](https://github.com/mattdisk/AiShell) |
| [multithreaded-file-server-c](multithreaded-file-server-c.md) | TCP socket programs, a custom Getfile file-transfer protocol library, and a boss-worker multithreaded client and server using POSIX threads. | Available on request |
| [webproxy-cache-c](webproxy-cache-c.md) | Multithreaded HTTP proxy built on libcurl, paired with a cache daemon that communicates over POSIX message queues, shared memory, and semaphores. | Available on request |
| [vm-resource-coordinator](vm-resource-coordinator.md) | Host-side daemons that re-pin KVM vCPUs to balance physical-core load and adjust guest memory through ballooning, driven by libvirt statistics. | Available on request |

[Back to portfolio overview](../README.md)
