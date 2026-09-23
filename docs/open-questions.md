# Open Questions

Resolve these before production work.

## Must answer before the PoC

| Question | Owner |
|---|---|
| Which non-destructive GUI/hardware test is the PoC? | Test Engineering |
| Which Windows version, application version, drivers, and firmware are in scope? | Rig owner |
| Which runner applies to the rig: Go/UI Automation or PAD? | Platform + Test Engineering |
| Can the test run in a managed interactive desktop (Go)? | Windows Workplace + Security |
| Can the real test run in PAD's unattended RDP session rather than the console? | Test Engineering + rig owner |
| What happens on RDP disconnect, lock, desktop switch, and human login? | Windows Workplace + Test Engineering |
| Which GCP region, projects, VPCs, and shared-VPC/landing-zone controls apply? | Valeo Cloud Platform |
| Is a Palo Alto VM-Series already the approved GCP inspection appliance, and who owns its HA/policy/logging? | Valeo Network Security |
| Which Valeo-approved SD-WAN/MPLS path and HA VPN peer devices terminate the factory-to-GCP connection? | Valeo Network + Cloud Platform |
| Which on-premises prefixes may be routed to GCP, and which must never be advertised? | Valeo Network Security |
| Which exact LAN hosts and ports may the cloud reach? | Network Security |
| Which Wormhole version, topology, and egress rules apply to our tenant? | Platform team |
| What recovery action is safe after `UNKNOWN`? | Test Engineering + hardware owner |

## Must answer before production

- CI identity, gateway/runner mTLS rotation, and emergency credential revocation.
- API contract: callbacks or polling, errors, cancellation, idempotency, and artifacts.
- Queue, lease, preflight, execution, upload, and recovery timeouts.
- Retry rules per test type, especially after `UNKNOWN`.
- Artifact retention, privacy, encryption, and access audit.
- Go binding review: API coverage, licence, maintenance, and security.
- PAD governance: Process licences, Dataverse environment, OAuth permissions, machine connections, callback allow-list, and flow versioning.
- Rig labels, shared devices, maintenance windows, quotas, and priorities.
- Agent/runner updates, signing, SBOM, rollback, monitoring, and support ownership.
- IPsec/BGP, Cloud Router, VM-Series, and route-filter ownership; alerting and incident handoff across Valeo Network, Security, and Platform teams.
- Whether any WireGuard use is approved, and how it is terminated, monitored, and governed without bypassing the landing zone.

Record every answer with owner, date, evidence, and accepted risk.
