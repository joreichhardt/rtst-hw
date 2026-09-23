# Architecture

## Scope

The cloud hosts the test API, scheduler, job store, and artifact storage. Windows rigs keep the GUI application and physical hardware. A local gateway is the only LAN endpoint exposed to the cloud through Control Plane Wormhole.

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
| Wormhole Agent | Network transport only. |
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

- Cloud workload → gateway only; no broad LAN route, no direct Windows or RDP access.
- CI authenticates with OIDC workload identity or short-lived tokens.
- Gateway and Windows runner use separate device identities and rotating mTLS credentials.
- Allow-listed test definitions only; never expose arbitrary shell execution.
- Encrypt artifacts, use short-lived download URLs, and redact secrets from logs/screenshots.
- Confirm actual Wormhole egress, DNS, proxy, and TLS-inspection requirements with the deployed Control Plane tenant. Do not infer them from this document.

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
