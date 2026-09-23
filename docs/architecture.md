# Architecture

## Scope

The test platform runs in the **GCP landing zone**: Test API, scheduler, job store, and artifact storage. Windows rigs keep the GUI application and physical hardware in the Valeo industrial network.

## Hybrid connection: GCP to on-premises hardware

The hardware stays on-premises. It is never internet-exposed or registered as a cloud workload.

### Two separate connection layers

| Layer | Purpose | Design |
|---|---|---|
| Corporate network | Valeo ↔ GCP transport, administration, routing, and security operations | Valeo-approved SD-WAN/MPLS carrying HA VPN (IPsec/BGP). The factory has no direct GCP Interconnect. |
| Test-service path | Restricted traffic from the Control Plane workload to the test service | Wormhole Agent on a local gateway; only the test gateway's approved TCP/UDP endpoint is reachable. |

### GCP landing-zone path

1. Deploy the platform in a dedicated GCP project/VPC in the Valeo landing zone.
2. Place a Palo Alto VM-Series appliance in the approved inspection/transit design. The Valeo firewall team owns policy, logging, upgrades, and HA design.
3. Connect the Valeo WAN edge to GCP through HA VPN (IPsec/BGP) over the approved SD-WAN/MPLS path. Cloud Router/BGP handles dynamic routing.
4. Route only approved on-premises test-network prefixes. Do not advertise broad industrial ranges; never expose the Windows rigs to the internet.
5. Install the Wormhole Agent on a dedicated local gateway VM, not on the Windows rigs. It maintains the Control Plane connection and exposes only the test gateway.
6. The gateway relays a reserved job to the local Windows runner. Results return via gateway → Wormhole → cloud API → CI.

**Decision:** WireGuard is not the default. It is only an option if Valeo Network Security explicitly approves its termination, key management, routing, monitoring, and support model inside the landing zone. It must not bypass the corporate WAN or Palo Alto policy point.

If the corporate connection or Wormhole disconnects, stop new dispatches. Treat a started job with no confirmed result as `UNKNOWN`.

## Verified Control Plane facts

Control Plane documents that its Wormhole agent:

- connects workloads to selected private TCP/UDP endpoints, including on-premises networks;
- runs in the private network and maintains a secure persistent connection to Control Plane;
- is organisation-scoped and used with identities;
- supports active-active v2 deployments; and
- has an optional v2 proxy on port 3128 for private-network calls to Control Plane workloads.

Wormhole documentation does **not** claim Windows GUI automation, RDP control, interactive-session management, test scheduling, or hardware test execution. Those are our components and must be proven in the PoC. [CP-1]

## Components

| Component | Responsibility |
|---|---|
| CI adapter | Creates a test job; polls status or receives a callback; fetches artifacts. |
| Test API | Authenticates callers and exposes jobs, status, and artifact links. |
| Scheduler | Selects a ready rig and creates an exclusive lease. |
| Job database | Source of truth for jobs, attempts, leases, and audit data. |
| Artifact storage | Immutable logs, screenshots, and optional diagnostic bundles. |
| GCP landing zone | VPC/project boundary for the test platform. |
| Palo Alto VM-Series | Valeo-managed inspection and policy enforcement point in the GCP design. |
| Cloud Router + HA VPN | Valeo-approved hybrid network underlay and dynamic routing. |
| Wormhole Agent | Restricted test-service network transport only. |
| Test gateway | Local job/result relay; the only Wormhole-exposed LAN service. |
| GUI runner | Executes the test on Windows: Go/UI Automation or Power Automate Desktop. |
| RDP | Human setup, diagnosis, and recovery only. |

## Job lifecycle

```mermaid
sequenceDiagram
  participant CI as CI
  participant API as Test API
  participant S as Scheduler
  participant G as Local gateway
  participant R as Windows runner

  CI->>API: Create job + idempotency key
  API->>S: Store job
  S->>S: Reserve one compatible rig
  S->>G: jobId, attempt, lease, fencing token
  G->>R: Offer job
  R->>R: Check desktop and hardware
  R-->>G: Ready or reject
  R->>R: Run test; collect logs/screenshots
  G->>API: Status and artifacts
  API-->>CI: Terminal status and artifact URLs
```

States: `QUEUED → RESERVED → DISPATCHED → RUNNING → SUCCEEDED | FAILED | UNKNOWN | CANCELLED`.

The scheduler issues a lease and monotonically increasing fencing token. Gateway and runner accept only the current token. Delivery is at-least-once; `jobId + attempt + fencingToken` must be idempotent. If the result is lost after the test starts, mark the job `UNKNOWN` and quarantine the rig until recovery decides whether a retry is safe.

## Reservations, timeouts, and failures

- Capacity is one GUI/hardware job per rig.
- Different rigs can run in parallel unless they share hardware or test data.
- Define queue, lease, preflight, execution, upload, and recovery timeouts per test type.
- A missed heartbeat does not immediately release physical hardware.
- A disconnected runner before `RUNNING` can receive the job again. After `RUNNING`, never retry automatically without a safe test-specific rule.

## Security and networking

- Corporate underlay: Valeo-approved SD-WAN/MPLS carrying HA VPN; no unmanaged direct tunnel or factory-to-GCP Interconnect.
- GCP landing zone: VM-Series policy controls and logs approved north-south/east-west paths as decided by Valeo Security.
- Route only test-network prefixes; no broad industrial LAN route, no direct Windows or RDP access from cloud workloads.
- Wormhole: cloud workload → gateway only; it complements, not replaces, the corporate hybrid network.
- CI authenticates with OIDC workload identity or short-lived tokens.
- Gateway and Windows runner use separate device identities and rotating mTLS credentials.
- Allow-listed test definitions only; never expose arbitrary shell execution.
- Encrypt artifacts, use short-lived download URLs, and redact secrets from logs/screenshots.
- Confirm actual Wormhole egress, DNS, proxy, and TLS-inspection requirements with the deployed Control Plane tenant. Confirm the VM-Series topology, HA model, route control, and logging requirements with Valeo Security.

## Windows GUI runner options

### Option A: Go + Windows UI Automation

A Go agent runs on each rig. A small Windows service can report health, but an interactive worker must perform UI automation. The worker starts a job only after checking the user session, desktop, application, and hardware.

Candidate binding: `github.com/uandersonricardo/uiautomation`. Validate API coverage, maintenance, licence, and security in the PoC.

### Option B: Power Automate Desktop (PAD)

The Test API can call Dataverse `RunDesktopFlow` with a flow ID, machine connection, and inputs. It receives a `flowsessionId`, then polls status/output or handles an idempotent HTTPS callback. [MS-1]

Unattended PAD creates its own RDP session, not the console session, keeps the screen locked, and signs out afterwards. On Windows 10/11, an unattended run cannot start while any Windows user session is active, even locked. A Power Automate Process licence is required. [MS-2]

PAD can replace the Go GUI runner, but not the API, scheduler, reservation model, or recovery policy. Its execution path is `Test API → Dataverse/Power Automate → PAD on Windows → Dataverse → Test API`; Wormhole is not needed to start the PAD flow. Prove that the real application and hardware work in PAD's RDP session. Reject this option for console-session-dependent tests.

## Windows risks

| Risk | Handling |
|---|---|
| RDP disconnect or desktop switch | Test the exact Windows/RDP policy. Stop the Go runner safely if its desktop changes. |
| Locked screen | Go runner reports `BLOCKED_SESSION`; test PAD under its locked unattended session. |
| Human RDP during a job | Block or audit it; move the job to `UNKNOWN` if the desktop changed. |
| UAC, updates, modal dialogs | Capture diagnostics, time out, and fail the environment. Do not add generic dialog clicking. |
| Crash or reboot | Quarantine the rig and use a recovery runbook. |

## Sources

- **[CP-1]** Control Plane, [Agent](https://docs.controlplane.com/reference/agent), accessed 2026-09-23.
- **[MS-1]** Microsoft Learn, [Work with desktop flows using code](https://learn.microsoft.com/en-us/power-automate/developer/desktop-flow-public-apis), accessed 2026-09-23.
- **[MS-2]** Microsoft Learn, [Run unattended desktop flows](https://learn.microsoft.com/en-us/power-automate/desktop-flows/run-unattended-desktop-flows), accessed 2026-09-23.
- **[GCP-1]** Google Cloud, [Cloud VPN overview](https://cloud.google.com/network-connectivity/docs/vpn/concepts/overview), accessed 2026-09-23.
- **[GCP-2]** Google Cloud, [Cloud Interconnect overview](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/overview), accessed 2026-09-23. It remains a reference for why it is not selected here.
