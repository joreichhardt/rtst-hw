# Proof of Concept

## Goal

Prove that CI can reserve one Windows rig, run a safe GUI/hardware test, and collect a terminal status, logs, and screenshot.

Run both candidates on the same real rig:

- **Track A:** Go agent using Windows UI Automation through the local gateway and Wormhole.
- **Track B:** Power Automate Desktop started through Dataverse.

## Setup

- One Control Plane cloud workload with a minimal Test API and scheduler.
- One local gateway behind a Wormhole v2 agent.
- One Windows rig with the target application and physical hardware.
- One non-destructive test case with a documented safe state and recovery owner.
- A dedicated Windows test account.
- One CI job.

## Acceptance criteria

| Step | Check | Pass condition |
|---|---|---|
| Network boundary | Cloud calls the gateway health endpoint. | No access to RDP, Windows hosts, or arbitrary LAN targets. |
| Rig readiness | Runner reports version, rig ID, and desktop readiness. | Locked or missing desktop is `NOT_READY`. |
| Track A: Go runner | Run a simple UI Automation flow. | Ten consecutive runs; failures include a reason and screenshot. |
| Track B: PAD | Start a desktop flow through `RunDesktopFlow`. | `flowsessionId` maps to `jobId`; callback and polling produce the same final status. |
| PAD session | Run against real hardware in PAD's unattended RDP session. | Application and hardware work there; otherwise exclude PAD for this rig. |
| Reservation | Submit two jobs for one rig. | Exactly one runs; the other queues or gets a defined capacity response. |
| CI contract | Create a job and fetch results. | Status, timestamps, logs, and screenshot are available. |
| Failure handling | Disconnect tunnel, gateway, and runner separately. | No duplicate run. A started job with lost outcome becomes `UNKNOWN`. |
| RDP and lock | Test disconnect, lock, reconnect, and human login. | Behaviour is recorded; runner fails safely rather than clicking blindly. |
| Security | Try expired credentials and an unapproved test definition. | Request is rejected before execution. |

## Required evidence

Record `jobId`, attempt, fencing token, rig ID, runner/gateway version, start/end time, state transitions, artifact hashes, and recovery action.

Measure queue time, job time, success rate, disconnects, `UNKNOWN` jobs, and manual recoveries.

## Decision

Choose a runner only if it passes the shared criteria and its own desktop/session test.

- Choose **Go** when direct UI Automation is stable and the managed interactive-session model is acceptable.
- Choose **PAD** when the real test works in PAD's unattended RDP session and licensing/governance are acceptable.
- Do not proceed if neither runner can execute the real test safely and repeatably.
