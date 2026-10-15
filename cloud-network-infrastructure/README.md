# Cloud & Network Infrastructure

Control planes and cloud pipelines that program infrastructure: software-defined networking, containerized network functions, and an event-driven serverless scanning service.

**Skills demonstrated:** OpenFlow and Open vSwitch, SDN controllers (OS-Ken, Mininet), Docker, REST interfaces, Azure Service Bus and Function Apps, PostgreSQL, and Python.

| Project | Summary | Source |
|---|---|---|
| [nfv-service-chain-orchestrator](nfv-service-chain-orchestrator.md) | Single-node NFV control plane that deploys containerized Firewall and NAT chains, programs Open vSwitch over OpenFlow, and balances connections across instances with connection affinity. | Available on request |
| [sdn-traffic-engineering-controller](sdn-traffic-engineering-controller.md) | OpenFlow 1.3 controller with shortest-path and widest-path routing, plus adaptive per-flow rerouting and redistribution driven by live bandwidth monitoring. | Available on request |
| [vulnscan](vulnscan.md) | Event-driven vulnerability-scanning pipeline for GitHub repositories on Azure Service Bus and Function Apps, running Semgrep, Trivy, and Gitleaks and storing results in PostgreSQL. | Available on request |

[Back to portfolio overview](../README.md)
