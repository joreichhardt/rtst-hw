# Remote Test Hardware

Architecture plan for running physical Windows test rigs as a CI service. This repository contains planning only.

## Goal

A CI pipeline requests a test, reserves a compatible test rig, and receives status, logs, and screenshots. GUI and hardware work stay on the assigned Windows machine.

RDP remains available for setup, diagnosis, and recovery. It is not the CI execution path.

## Architecture

```mermaid
flowchart LR
  CI[CI pipeline] -->|OIDC / API token| API[Test API]
  API --> SCH[Scheduler and reservations]
  API --> DB[(Job database)]
  API --> ART[(Artifact storage)]

  subgraph Cloud[Control Plane Cloud]
    API
    SCH
    DB
    ART
    CW[Cloud workload]
  end

  subgraph LAN[Local test network]
    WH[Control Plane Wormhole Agent\nnetwork tunnel only]
    GW[Test gateway]
    WIN[Windows test rig\nGUI runner + interactive desktop]
    HW[Physical hardware]
    RDP[RDP for people\nsetup and recovery]
  end

  CW -. TCP/UDP through Wormhole .-> WH
  WH --> GW
  GW --> WIN
  WIN --> HW
  RDP --> WIN
  WIN -->|logs and screenshots| GW
  GW -.-> API
  API -->|status and artifact URLs| CI
```

Editable diagram: [docs/architecture.drawio](docs/architecture.drawio)

## Keep these roles separate

| Component | Purpose |
|---|---|
| **Control Plane Wormhole Agent** | Private-network connectivity for selected TCP/UDP endpoints. It does not run tests or manage Windows desktops. |
| **Test gateway** | The only local endpoint exposed through Wormhole. It relays jobs and results. |
| **GUI runner** | Runs on the Windows rig: either a Go UI Automation agent or Power Automate Desktop. |
| **RDP** | Human-only setup, diagnosis, and recovery. |

## Operating rules

- One GUI/hardware job per test rig.
- Each job uses an idempotency key, lease, and fencing token.
- A job with an unknown outcome stays `UNKNOWN`; do not blindly rerun it.
- The cloud may reach the gateway, not RDP or arbitrary LAN hosts.
- Run GUI tests only when the selected runner confirms a usable Windows desktop.

## Documents

- [Architecture](docs/architecture.md)
- [Proof of concept](docs/proof-of-concept.md)
- [Five-minute team talk](docs/team-talk.md)
- [Open questions](docs/open-questions.md)
- [Control Plane Wormhole source check](docs/research/control-plane-wormhole.md)
