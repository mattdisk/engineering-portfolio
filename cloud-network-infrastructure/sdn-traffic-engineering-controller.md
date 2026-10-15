# Adaptive SDN Traffic Engineering Controller

An OpenFlow 1.3 controller, built on OS-Ken and Mininet, for a network with heterogeneous link bandwidth and latency. It discovers topology dynamically and offers two static routing objectives — minimum-latency shortest-path routing and a max-min widest-path adaptation that maximizes bottleneck bandwidth — plus two adaptive modes that extend widest-path routing with live per-flow bandwidth monitoring: proactive per-flow rerouting and periodic redistribution of the most bandwidth-dominant flows. Path recomputation is coordinated with topology changes, live monitoring, and flow-table updates across the switches. A companion test harness drives concurrent, JSON-scheduled iperf traffic and link-fault scenarios against a Mininet-emulated topology.

This differs from a basic learning-switch controller, which forwards using learned MAC-to-port state and flooding, without topology-wide metric optimization or traffic-aware path recomputation.

**Stack:** OS-Ken (OpenFlow controller framework) · Mininet · Open vSwitch (OpenFlow 1.3) · NetworkX · Python · iperf

Deeper detail on each of the following sections is in the project's `docs/` directory, which is part of the private source repository.

## 1. Overview

This repository implements an adaptive OpenFlow controller and its supporting infrastructure: dynamic topology discovery, two path-selection algorithms, traffic-aware proactive routing, periodic flow redistribution, per-flow/per-port monitoring, and a JSON-driven concurrent traffic-generation and test harness. It is designed to run against a Mininet-emulated network of OpenFlow 1.3 (Open vSwitch) switches.

## 2. Key Capabilities

- Dynamic topology discovery via OS-Ken topology events, maintained as a live NetworkX graph.
- Latency-minimizing shortest-path routing via standard latency-weighted Dijkstra.
- Bottleneck-bandwidth-maximizing widest-path routing via a max-min widest-path adaptation of the same traversal.
- Proactive routing that adjusts path selection using live per-flow bandwidth monitoring.
- Periodic recomputation of known flows' paths, processed in descending order of observed average bandwidth.
- Per-port and per-flow statistics monitoring via periodic OpenFlow stats requests.
- A threaded, JSON-configured test harness generating concurrent, time-scheduled iperf traffic (at configured target send rates) and link up/down fault injection.

## 3. Architecture

The controller is built from a `BaseController` OS-Ken application composed with three mixins — configuration loading, topology-graph maintenance, and monitoring — plus MAC/host learning, a debounced topology-recomputation mechanism, and explicit locking around the topology graph and flow installation. See `docs/architecture.md` for the full breakdown, including exactly which shared state is and is not protected by an explicit lock.

## 4. Controller Modes

Two static routing baselines, each extending the base controller independently:

- **Shortest path** (`controller_shortest_path.py`) — routes on minimum accumulated link latency.
- **Widest path** (`controller_widest_path.py`) — routes on maximum bottleneck bandwidth along the path.

Two adaptive modes, both extending the widest-path controller:

- **Proactive traffic-aware routing** (`controller_proactive.py`) — recomputes widest paths against a bandwidth-adjusted graph snapshot reflecting recently-observed per-flow usage.
- **Periodic redistribution** (`controller_redistribute.py`, extends the proactive controller) — every `T2` seconds, recomputes paths for known flows in descending order of observed average bandwidth.

## 5. Routing Algorithms

Shortest-path routing uses standard latency-weighted Dijkstra. Widest-path routing uses a max-min widest-path adaptation of the same traversal: relaxation takes the minimum of a path's running bottleneck and the next edge's bandwidth instead of a sum, and the priority queue orders by descending bottleneck width instead of ascending distance. Both traversals track a *set* of optimal predecessors per node (so ties are preserved rather than resolved during the traversal), with tie-breaking deferred to random sampling at path-reconstruction time. Full explanation is in `docs/routing-algorithms.md`.

## 6. Monitoring and Flow Redistribution

A monitoring mixin periodically polls flow and port statistics from every known switch (interval `T1`) and maintains a rolling per-flow byte-history (window size `S1`). This feeds two adaptive behaviors: proactive path selection, which subtracts recent utilization from link capacity before computing a widest path per flow, and periodic redistribution, which every `T2` seconds recomputes paths for known flows in descending order of observed average bandwidth, reinstalling the most bandwidth-dominant flows' paths first. See `docs/architecture.md` and `docs/routing-algorithms.md`.

## 7. Traffic Generation and Test Scenarios

`test_service.py` is a threaded harness that reads a JSON scenario file describing concurrent `iperf` flows (each with a scheduled start/end time and a target UDP send rate — offered load, not a guarantee of achieved throughput) and `control_link` up/down events, and drives them against a running Mininet network. Five bundled scenario files live under `tests/`. Full format and per-mode scenario intent is documented in `docs/testing.md`.

## 8. Repository Structure

```
.
├── README.md
├── base_controller.py                # core OS-Ken app: packet-in handling, flow install, mode extension points
├── base_controller_mixin/
│   ├── config_mixin.py                # link-configuration loading
│   ├── monitor_mixin.py               # flow/port stats polling, redistribution trigger
│   └── topology_mixin.py              # topology-event handling, debounced recompute
├── controller_shortest_path.py       # shortest-path (latency) mode
├── controller_widest_path.py         # widest-path (bottleneck bandwidth) mode
├── controller_proactive.py           # proactive traffic-aware mode
├── controller_redistribute.py        # periodic redistribution mode
├── sdn_topo.py                       # Mininet topology built from a link-config JSON file
├── sdn_graph.py                      # NetworkX graph twin of the Mininet topology
├── utils.py                          # OpenFlow stats parsing helpers
├── test_service.py                   # concurrent, scheduled iperf/link-fault test harness
├── controller.conf                   # T1/T2/S1 and link-config parameters
├── link_config*.json                 # topology/link fixtures
├── tests/                            # bundled test scenario files
├── helpers/                          # operational ovs-vsctl/ovs-ofctl scripts
├── start_mininet.sh / start_controller.sh / start_test.sh
├── visualize_topo.ipynb              # NetworkX/Matplotlib topology visualization
└── docs/
    ├── architecture.md
    ├── routing-algorithms.md
    ├── testing.md
    ├── limitations.md
    └── attribution.md
```

## 9. Environment Prerequisites and Execution Workflow

> The commands and scenario outcomes described here are based on the original development and course demonstration workflow. They were not freshly re-executed during the portfolio documentation pass, and exact version compatibility of the technologies listed below was not freshly revalidated.

### Environment prerequisites

Based on the imports and commands actually used in this repository:

- Python 3
- [OS-Ken](https://opendev.org/openstack/os-ken) — the OpenFlow controller framework the controller modes are built on
- [Mininet](http://mininet.org/) — the network emulation environment the topology and test harness run against
- Open vSwitch, configured for OpenFlow 1.3 — the switches controlled by the topology and the `helpers/` scripts
- [NetworkX](https://networkx.org/) — topology graph representation and routing computation
- [iperf](https://iperf.fr/) — traffic generation used by `test_service.py`
- Jupyter and Matplotlib — required only for the optional `visualize_topo.ipynb` notebook

The original development environment used the following command as a starting point. It is retained here as a historical note, not a complete or independently verified setup procedure:

```
sudo apt-get install python3-networkx
```

### Interactive topology inspection

For manually exploring a topology and issuing ad hoc Mininet CLI commands (`pingall`, `iperf`, `link ... down`/`up`, etc.):

1. Terminal A (Mininet): `./start_mininet.sh`
2. Terminal B (controller): `./start_controller.sh <controller ID>` — `1` = shortest path, `2` = widest path, `3` = proactive, `4` = redistribution

### Automated scenario harness

For running one of the bundled, time-scheduled JSON scenarios (see `docs/testing.md` and section 10 below):

1. Terminal A (controller): `./start_controller.sh <controller ID>`
2. Terminal B (scenario harness): `./start_test.sh <scenario name> <link config file>`

`start_test.sh` runs `test_service.py`, which creates and starts its own Mininet network from the given link-configuration file before running the scenario. `start_mininet.sh` should **not** be run separately for this workflow — starting a second, independent Mininet network would conflict with the one `test_service.py` already manages.

Topology visualization: open `visualize_topo.ipynb` and edit the `link_config` variable to render a different topology fixture.

## 10. Documented Scenarios

> As with the previous section, the steps and expected outcomes below are the originally documented workflow, not results reproduced during this documentation pass. See `docs/testing.md` for the full scenario-format explanation and additional caveats.

**Shortest path** — Terminal A: `./start_controller.sh 1`; Terminal B: `./start_test.sh shortest link_config_demo.json`.
Steps (per `tests/shortest.json`): iperf h1↔h2 → bring the `s1`-`s2` link down → bring it back up → iperf h1↔h2 again.
Documented expectation: initial paths route through `s2`; after the link goes down, traffic reroutes through `s3`; after the link is restored, the original paths return.

**Widest path** — Terminal A: `./start_controller.sh 2`; Terminal B: `./start_test.sh widest link_config_demo.json`.
Steps (per `tests/widest.json`): iperf h1↔h2 → bring the `s2`-`s3` link down → bring it back up → iperf h1↔h2 again. Note: this differs from the shortest-path scenario's link — see `docs/testing.md` for a note on an inconsistency between this project's original prose description and the actual scenario file. Documented expectation: the widest-path controller selects different paths than the shortest-path controller over the same topology, reflecting the different optimization metric.

**Proactive rule** — set `link_config` to `link_config_demo1.json` in `controller.conf`; Terminal A: `./start_controller.sh 3`; Terminal B: `./start_test.sh proactive link_config_demo1.json`.
Steps: concurrent iperf h1↔h2 and h3↔h4, with the configured target rate for each flow swapping which pair is dominant partway through the run.
Documented expectation: either flow may change path as the proactive controller reacts to shifting observed utilization.

**Redistribution** — set `link_config` to `link_config_demo1.json` in `controller.conf`; Terminal A: `./start_controller.sh 4`; Terminal B: `./start_test.sh redist link_config_demo1.json`.
Steps: background flows at a configured target rate of 10 Mbps across all hosts, with a shifting "dominant" pair across three phases (h1→h2 at 50 Mbps, then h3→h4 at 50 Mbps, then h2→h4 at 30 Mbps).
Documented expectation: every `T2` seconds the controller recomputes paths for known flows in descending order of observed average bandwidth, giving the currently-dominant pair its natural widest path first; other pairs may be rerouted to accommodate it.

## 11. Limitations

Mininet-emulated environment (not physical switching hardware); validation via Mininet-driven integration/demo scenarios rather than an automated unit-test suite; not freshly re-executed during this portfolio pass; explicit locking covers the topology graph and flow installation but not all shared controller state (see `docs/architecture.md`); configuration loading logs and falls back to defaults rather than failing hard. Full detail in `docs/limitations.md`.

## 12. Project Origin and Attribution

This project originated as a graduate-level SDN coursework implementation, built to a provided technical specification that is not reproduced here. No vendored upstream example controller code was found in this repository during static review; it depends on OS-Ken, Mininet, Open vSwitch, NetworkX, and iperf as external technologies. Full detail in `docs/attribution.md`. 

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
