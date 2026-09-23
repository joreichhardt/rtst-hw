# Control Plane Wormhole Source Check

**Accessed:** 2026-09-23

## Verified facts

| Claim | Source |
| --- | --- |
| A Wormhole agent connects workloads to selected TCP/UDP endpoints in private networks, including on-premises networks. It runs in the private network and maintains a persistent secure connection to Control Plane. | [Control Plane Agent reference](https://docs.controlplane.com/reference/agent) |
| Agents are organisation-scoped and used with identities. | [Control Plane Agent reference](https://docs.controlplane.com/reference/agent) |
| v2 agents can run active-active. Control Plane recommends a fixed group of at least two instances for high availability. | [Control Plane Agent reference](https://docs.controlplane.com/reference/agent) |
| v2 provides an optional proxy on port 3128 for private-network calls to Control Plane workloads. | [Control Plane Agent reference](https://docs.controlplane.com/reference/agent) |

## Not claimed by the source

The reference does not claim Windows UI Automation, RDP control, Windows sign-in/session management, GUI testing, physical hardware testing, job scheduling, reservations, or exactly-once execution.

## Design consequence

Use Wormhole as a narrow network path to the local test gateway. Keep GUI execution in a separate Windows runner: Go/UI Automation or Power Automate Desktop.

## Related sources

- [Control Plane introduction](https://docs.controlplane.com/introduction)
- [Control Plane documentation index](https://docs.controlplane.com/llms.txt)
