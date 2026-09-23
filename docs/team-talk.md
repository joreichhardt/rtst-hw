# Five-Minute Team Talk

## 0:00–0:40 — Problem

Today, people use RDP to run GUI tests on physical Windows rigs. That is manual, hard to schedule, and does not give CI a reliable result package.

Goal: CI books a rig, runs one test, and gets status, logs, and screenshots.

## 0:40–1:30 — Design

Show the [architecture diagram](../README.md#architecture).

The platform runs in the GCP landing zone. The enterprise-approved SD-WAN/MPLS path carries HA VPN from the industrial site to GCP; the factory has no direct Interconnect. A Palo Alto VM-Series is the landing-zone inspection point. A local gateway is the only test service reached through Wormhole. The Windows rig runs the GUI test. RDP stays for people.

**Key point:** Corporate WAN provides hybrid connectivity. Wormhole is the restricted test-service path. Neither runs Windows GUI tests.

## 1:30–2:20 — Reliability

One rig runs one job. The scheduler uses leases and fencing tokens, so a late message cannot run an old job. Delivery may happen more than once; hardware execution must not.

If a job starts and its outcome is lost, it becomes `UNKNOWN`. Do not automatically rerun it; recover the rig first.

## 2:20–3:10 — Security

GCP gets only approved Enterprise test-network routes; the cloud workload reaches only the gateway through Wormhole, not RDP or the whole LAN. CI, gateway, and runner use separate identities. Test definitions are allow-listed; this is not a remote shell service.

Control Plane documents Wormhole for private TCP/UDP connectivity. It does not document Windows desktop automation.

## 3:10–4:20 — Two GUI runner options

**Option A: Go + Windows UI Automation.** Direct control, but we own the runner, updates, and desktop-session behaviour.

**Option B: Power Automate Desktop.** The Test API calls Dataverse `RunDesktopFlow`; it can poll status or receive a callback. Unattended PAD creates its own RDP session, not the console session. On Windows 10/11 it cannot run while another user session is active. Test this on the real hardware before choosing it.

## 4:20–5:00 — Ask

Provide one rig and one non-destructive test for the PoC. We will measure GUI stability, session behaviour, network limits, retries, and recovery. Choose the runner from those results, not assumptions.
