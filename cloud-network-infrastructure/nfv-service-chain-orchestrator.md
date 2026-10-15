# NFV Service Chain Orchestrator

## Summary

This project designs and implements a single-node NFV (Network Function Virtualization) control-plane prototype. Administrators register service chains that connect an internal and an external endpoint through a fixed two-stage role model: an optional Firewall stage followed by an optional NAT stage. The control plane deploys the requested network functions as containers, attaches them to a software-defined data plane, and installs the OpenFlow rules needed for traffic between the endpoints to pass transparently through the chain. Additional instances of a network function can be deployed at runtime, and the system distributes new connections across those instances with round-robin selection — decided independently for each role by the switch that hosts it — while preserving connection affinity for traffic already in progress.

The project's source repository (private; available on request) contains both a curated copy of that system's implementation, under `implementation/`, and the architecture, design, evolution, testing, and attribution documentation that explains it, under `docs/`. See "Repository Structure" and "Code Guide" below for how the two fit together, and "Environment and Execution Status" for exactly what has and has not been verified.

Architecture diagram showing the administrator/API client, REST interface, NFVI manager, SDN controller, shared cluster state, Docker-based NF instances, and the Open vSwitch data plane

## Project Highlights

- REST-based control-plane interface for registering service chains and requesting network function instances
- Container-based deployment of Firewall and NAT network functions on demand
- Software-defined data plane (Open vSwitch, OpenFlow) with controller-managed ARP resolution and forwarding
- Round-robin distribution of new connections across multiple instances of a network function
- Bidirectional connection affinity, maintained even through a network function that rewrites packet headers
- Support for adding network function instances to a running chain at runtime
- A traffic-driven test methodology used to validate forwarding, load distribution, and affinity behavior

## System Overview

At a high level, the system separates two concerns:

- A **control plane** that tracks which network function chains exist, where their instances are deployed, and how traffic should be routed between them.
- A **data plane** made of virtual switches and containerized network functions, programmed by the control plane so that traffic flows through the network functions without the endpoints needing any awareness of them.

An administrator registers a chain — for example, "Firewall then NAT" between one internal and one external endpoint, the only orderings supported being Firewall→NAT, Firewall-only, or NAT-only — and then requests that instances of each network function be launched. The control plane deploys the requested containers, wires them into the data plane, and records their addressing. When real traffic between the registered endpoints reaches the data plane, each switch independently resolves address lookups and, the first time this connection's traffic reaches it, selects an instance of the network-function role it hosts (via round robin) and installs forwarding rules for both directions of that connection through that instance. The switch hosting the Firewall role and the switch hosting the NAT role make this decision independently of each other; together, their two independent selections form the connection's path through the chain, with every packet of that connection — in both directions, at each switch — pinned to the instance it was first assigned to there.

See `docs/architecture.md` for the full component breakdown and deployment lifecycle.

## Key Engineering Challenge: Connection Affinity

The most technically interesting problem in this system is keeping a single logical connection pinned to the same network function instances in both directions, even when one of those instances rewrites packet addressing, as a NAT function does. A naive per-packet lookup breaks down once headers change mid-path, because the reverse-direction traffic no longer matches the addressing that was used to make the original forwarding decision. The system addresses this per switch, independently: each switch that hosts a network-function role selects an instance for that role — and proactively installs bidirectional forwarding rules for its own segment of the connection — the first time this connection's traffic reaches that specific switch. The switch hosting the Firewall role and the switch hosting the NAT role never make a joint decision and are never programmed in one combined action; each is triggered purely by when the connection's traffic happens to reach it, and later packets at that switch are then handled directly by the data plane without requiring another control-plane decision.

See `docs/design-notes.md` for the full explanation, including its precise scope and its limitations.

## Technology Stack

- **Python** for the control-plane and controller application logic
- **OS Ken**, an OpenFlow controller framework derived from the Ryu project, for SDN control and OpenFlow 1.3 programming
- **Open vSwitch** as the programmable data-plane switch
- **Docker** for on-demand deployment of network function instances
- **Linux networking** (`iptables`, routing) inside the network function containers
- A **REST interface** for administrator-driven chain registration and instance deployment
- **iperf3** and **hping3**, used historically to generate test traffic

`implementation/requirements.txt` lists the Python package dependencies, curated from static inspection of the source's import statements. Operating-system-level tools (Docker, Open vSwitch, `iptables`, `iperf3`, `hping3`) are not Python packages and are not part of that file — they are listed above as the environment the implementation assumes.

## Repository Structure

```
nfv-service-chain-orchestrator/
├── README.md
├── .gitignore
├── implementation/     # curated, flat copy of the final project's source
└── docs/               # architecture, design, evolution, testing, and attribution
    └── assets/         # diagrams referenced from the documents above
```

`implementation/` is intentionally **flat** — it mirrors the directory layout the code was actually written and run in, rather than being reorganized into topic folders. The Python files (`controller.py`, `osken_app.py`, `workshop_parent.py`, `helpers.py`, `wsgi.py`, `rest_router_start_server.py`) use plain, non-package imports such as `from osken_app import ServiceChainForwarder`, and the shell scripts and JSON fixtures reference each other by bare relative filenames, such as `curl --data @sfc_1.json` and `docker build -f Dockerfile.endpoint -t endpoint:latest .`. All of this only resolves correctly because every file lives in the same directory. Splitting `implementation/` into subfolders (for example, separating "control plane" from "infrastructure" files) would require introducing Python package structure and rewriting every import and relative path — a cosmetic reorganization with real behavioral risk, and one this repository deliberately avoids. Commands described in `docs/testing.md` and in `implementation/commands.md` assume the current directory is `implementation/`.

## Code Guide

For a reader who wants to go straight to the code, these are the entry points worth starting from:

- **REST / NFVI orchestration entry point:** `implementation/controller.py` — the OS Ken application entry point; registers the REST routes and drives on-demand container deployment.
- **OS Ken forwarding and connection-affinity logic:** `implementation/osken_app.py` — ARP handling, round-robin instance selection, and the per-switch bidirectional flow installation described in `docs/design-notes.md`.
- **Shared controller scaffolding:** `implementation/workshop_parent.py` — the base OpenFlow event-dispatch class the rest of the controller builds on.
- **Helper / state-allocation utilities:** `implementation/helpers.py` — MAC address, switch port, and instance-name allocation for newly deployed network functions.
- **WSGI integration:** `implementation/wsgi.py` and `implementation/rest_router_start_server.py` — the REST/WSGI plumbing that exposes `controller.py`'s routes over HTTP.
- **NF initialization scripts:** `implementation/init_fw.sh` and `implementation/init_nat.sh` — the in-container configuration run when a new Firewall or NAT instance is deployed.
- **Topology and demo automation:** `implementation/setup_topology.sh` and the numbered `implementation/step0_initialize_infra.sh` through `implementation/step3_scaleup.sh` scripts.
- **Traffic-generation / test harness:** `implementation/test_service.py`.
- **Representative service-chain and scale-up fixtures:** `implementation/sfc_1.json` (chain registration) and `implementation/scale_sfc_1.json` (adding an instance to a running chain).

This list intentionally does not enumerate every file in `implementation/` — see the directory itself for the complete set, and Section "Repository Structure" above for why it's laid out the way it is.

## Documentation Map

- `docs/architecture.md` — component breakdown, control-plane/data-plane split, and the conceptual deployment lifecycle
- `docs/design-notes.md` — the connection-affinity mechanism in depth, plus known limitations and tradeoffs
- `docs/project-evolution.md` — how the system was built up in stages, from basic containerized network functions to full dynamic orchestration
- `docs/testing.md` — the testing methodology used and a summary of the evidence it produced
- `docs/attribution.md` — provenance of framework/scaffolding components versus project-specific work

## Environment and Execution Status

Three distinct things are true here, and this repository is deliberate about keeping them separate:

- **Historical coursework execution.** The system was built, deployed, and exercised on a real Linux/Docker/Open vSwitch/OS Ken environment during its original development. The testing evidence summarized in `docs/testing.md` — packet traces, OpenFlow flow-table captures — comes from that historical execution, not from anything performed for this repository.
- **Curation performed on the copy in this repository.** The implementation under `implementation/` is a curated copy of the original, not a byte-identical one. Changes made during portfolio preparation were limited to: privacy redaction (one institutional email address removed from one file's header comment, name retained); removal of an unused import that referred to an excluded module; project-scoped Docker container cleanup (a `nfv-orchestrator=true` label, replacing a blanket "remove every container" step); explicit secure fail-mode and OpenFlow 1.3 configuration for both Open vSwitch bridges; a small set of statically-confirmed-unused imports and stale commented-out code removed from the project-specific Python files; four internal class renames for readability (`Workshop4` → `ServiceChainForwarder`, `FLOW_INFO` → `FlowInfo`, `rest_controller` → `RestEnabledController`, `RESTLinkage` → `NfvRestApi`); and a small set of test-harness improvements (an explicit `-t` duration derived from the traffic profile, a standard `if __name__ == "__main__":` guard, deriving the iperf3 server list from the traffic profile instead of a separately hard-coded one). All of this was done through static reading of the source, not by running it — see the next point. Every one of these changes is described in more detail in `docs/attribution.md`.
- **No fresh dynamic re-validation.** Neither this curation pass nor the documentation work that preceded it has built a Docker image, created an Open vSwitch bridge, run OS Ken, or generated traffic. Nothing here should be read as a claim that the curated copy currently builds, deploys, or passes tests, or that its behavior is guaranteed identical to the original — that would require a separate, explicitly-scoped verification effort, which has not been performed.

## Known Limitations

- The network topology is fixed (two switches, a predetermined addressing scheme) rather than general-purpose.
- Control-plane state is held in memory in a single process, with no persistence across restarts.
- Scaling down — removing a network function instance from a running chain — is not supported; only scaling up is.
- Newly deployed network function instances are not health-checked or monitored after deployment.
- A multi-step deployment sequence has no rollback if a later step fails after an earlier one has already succeeded.
- The REST interface has no authentication.
- The design assumes a single control-plane instance and does not address distributed or highly-available operation.
- A chain supports at most a fixed two-role model — an optional Firewall stage (switch 1) followed by an optional NAT stage (switch 2) — not arbitrary NF ordering, repeated roles, or chains of other lengths.
- The controller predicts each network function instance's OpenFlow port number from its own internal counters rather than querying Open vSwitch for the actual assigned value; this assumes a clean topology and ports created in the expected order. See `docs/design-notes.md` for the full explanation.

## What Remains Private

The curated implementation in `implementation/` **is** part of the project's private source repository. What is not included there is the complete, uncurated original course submission: the instructor-provided assignment specifications, the three earlier workshop submissions that preceded this project, and the original unfiltered environment/dependency freeze. Those remain private and are not republished here — the workshops are represented only through the prose narrative in `docs/project-evolution.md`, not as files a reader can open directly, and the instructor specifications are not reproduced anywhere in this repository.

## Attribution

See `docs/attribution.md` for a summary of which parts of the system were third-party framework code, which were provided as course scaffolding, and which were project-specific implementation work.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
