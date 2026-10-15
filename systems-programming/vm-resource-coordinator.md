# VM Resource Manager — Dynamic vCPU Scheduling and Memory Ballooning for KVM/libvirt

This project implements two host-side daemons that manage the resources of KVM guest virtual machines through `libvirt`: a **vCPU scheduler** that periodically re-pins virtual CPUs to physical cores to balance load across the host, and a **memory coordinator** that grows or shrinks each guest's memory via the balloon driver based on observed demand. Both run in host user space, polling hypervisor statistics on a fixed interval and issuing `libvirt` calls to adjust CPU affinity and memory allocation.

## What This Project Demonstrates

- Systems programming in C against a hypervisor API (`libvirt`), including connection management, domain enumeration, and statistics collection.
- Resource-scheduling logic driven by runtime statistics rather than static configuration (per-pCPU load balancing, gradual memory reclamation/allocation).
- Hypervisor/OS concepts: vCPU-to-pCPU affinity pinning, memory balloon drivers, and the tradeoffs of polling-interval-based control loops.
- Python tooling for VM lifecycle automation (`libvirt` Python bindings) and scenario-based integration testing against live KVM guests.

## Architecture

```
                 +------------------+          +------------------+
   host          |  vCPU Scheduler  |          | Memory Coordinator|
   (user space)  |  (cpu/src)       |          | (memory/src)      |
                 +--------+---------+          +---------+--------+
                          |  libvirt API                  |  libvirt API
                          v                                v
                 +------------------------------------------------+
                 |            libvirt / KVM hypervisor             |
                 +------------------------------------------------+
                          |                                |
                    guest vCPUs                     guest memory (balloon)
                     pinned to                        grown/shrunk per
                     physical cores                    guest demand
```

A reference architecture diagram is included at res/project1_architecture.png.

### vCPU Scheduler (`cpu/src/vcpu_scheduler.c`)

On each interval, the scheduler:

1. Reads cumulative vCPU time for every running domain and derives per-vCPU utilization since the last sample.
2. Aggregates utilization by the pCPU each vCPU currently occupies.
3. Evaluates whether the host is balanced using the mean and standard deviation of per-pCPU load (a standard-deviation threshold determines whether rebalancing is needed), with a special case to avoid needless churn when every overloaded pCPU already holds only one vCPU.
4. If rebalancing is warranted, sorts vCPUs by utilization (heaviest first) and greedily reassigns each to the currently least-loaded pCPU, then applies the new pinning with `virDomainPinVcpu`.

This is a greedy, statistics-driven heuristic intended to converge to a balanced, stable pinning without needing to know the number of vCPUs or pCPUs in advance.

### Memory Coordinator (`memory/src/memory_coordinator.c`)

On each interval, the coordinator:

1. Enables periodic balloon-memory statistics collection per domain (`virDomainSetMemoryStatsPeriod`) and establishes a baseline of allocated vs. used memory.
2. Compares current usage against the previous sample for each guest.
3. Gradually reclaims memory from guests whose usage has stayed flat and which retain more than a configured minimum free buffer.
4. Gradually grows allocation for guests whose usage is increasing, capping growth per interval so allocation changes smoothly rather than in large jumps.
5. Before granting additional memory, checks host free memory so the host itself is never starved.

The result is a control loop that shifts memory between guests (and the host) in small, bounded steps rather than reacting to instantaneous demand spikes.

### Supporting Tooling

- vm/vm.py — a thin `libvirt` Python wrapper (`VMManager`) for starting/stopping/destroying guests, pinning vCPUs, and setting memory limits, used by the CLI scripts in vm/ (`startallvm.py`, `shutdownallvm.py`, `setallmaxmemory.py`, etc.).
- `cpu/test/` and `memory/test/` — scenario-based integration tests. Each includes small C workload generators (`iambusy.cpp`, `run.cpp`) that synthetically load CPU or memory inside a guest, a `monitor.py` poller that prints per-interval utilization/mapping state, and `plot_graph_*.py` for visualizing a test run.

## Repository Structure

```
.
├── cpu/
│   ├── src/        vCPU scheduler implementation, Makefile, algorithm write-up
│   └── test/       CPU scheduling test scenarios and monitoring tools
├── memory/
│   ├── src/        Memory coordinator implementation, Makefile, algorithm write-up
│   └── test/       Memory coordinator test scenarios and monitoring tools
├── vm/             libvirt-based VM lifecycle tooling and environment setup notes
├── res/            Architecture diagram and reference output samples
└── docs/           Design notes (this portfolio pass)
```

## Build Instructions

Each component builds independently with `make` and links against `libvirt` (`libvirt-dev`/`libvirt-bin` must be installed):

```bash
cd cpu/src && make        # produces ./vcpu_scheduler
cd memory/src && make     # produces ./memory_coordinator
```

## Running

Both programs take a single argument: the polling interval in seconds. Each requires a working connection to a local KVM/libvirt hypervisor (`qemu:///system`) with guest VMs already running.

```bash
./vcpu_scheduler 2         # rebalance vCPU pinning every 2 seconds
./memory_coordinator 2     # rebalance guest memory every 2 seconds
```

See vm/EnvironmentSetup.md for provisioning a KVM-capable host (cloud VM or local Vagrant/VirtualBox environment) suitable for running and testing both components.

## Testing

Each component ships with three scenario-based integration tests that exercise a specific load pattern against real libvirt-managed guests (e.g., already-balanced pinning, all-load-on-one-pCPU, mixed heavy/light load for the scheduler; single-guest growth, uniform multi-guest growth, and shifting demand for the memory coordinator). Test procedures and expected behavior are documented in cpu/test/HowToDoTest.md and memory/test/HowToDoTest.md.

**Verification status**: these tests require a KVM hypervisor with multiple running guest VMs and were originally exercised in that environment. They were not re-run as part of this documentation pass — this pass consisted of a static review of the existing source and test harnesses, not fresh test execution.

## Design Notes

See docs/design-notes.md for the reasoning behind the scheduling and memory-management approach, known tradeoffs, and limitations.

## Limitations

- The scheduling and memory heuristics use fixed-size static arrays (indexed up to 1024 domains/vCPUs/pCPUs) rather than dynamic sizing.
- Both control loops assume the ordering of domains returned by `libvirt`'s list calls is stable between polling intervals, since per-domain state is tracked by array index rather than by domain UUID.
- The vCPU scheduler balances purely on measured utilization; it has no awareness of NUMA topology, memory locality, or per-guest priority.
- Testing depends on a live KVM/libvirt environment with multiple guest VMs; there is no mocked or unit-level test path that runs without a hypervisor.

## Future Improvements

- Track per-domain state by UUID instead of array position to remove the ordering assumption.
- Replace fixed-size arrays with dynamically sized structures.
- Add a lightweight mock of the `libvirt` API to allow the core balancing logic to be exercised in CI without a hypervisor.
- Extend the vCPU scheduler to consider NUMA locality when selecting a target pCPU.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
