# Five-Minute Team Talk

## 0:00–0:40 — Problem

Today, people use RDP to run GUI tests on physical Windows rigs. That is manual, hard to schedule, and does not give CI a reliable result package.

Goal: CI books a rig, runs one test, and gets status, logs, and screenshots.

## 0:40–1:30 — Design

Show the [architecture diagram](../README.md#architecture).

The cloud owns the API, scheduler, jobs, and artifacts. A local gateway is the only LAN service reached through Control Plane Wormhole. The Windows rig runs the GUI test. RDP stays for people.

**Key point:** Wormhole provides network transport. It does not run Windows GUI tests.

## 1:30–2:20 — Reliability

One rig runs one job. The scheduler uses leases and fencing tokens, so a late message cannot run an old job. Delivery may happen more than once; hardware execution must not.

If a job starts and its outcome is lost, it becomes `UNKNOWN`. Do not automatically rerun it; recover the rig first.

## 2:20–3:10 — Security

The cloud can reach the gateway only, not RDP or the whole LAN. CI, gateway, and runner use separate identities. Test definitions are allow-listed; this is not a remote shell service.

Control Plane documents Wormhole for private TCP/UDP connectivity. It does not document Windows desktop automation.

## 3:10–4:20 — Two GUI runner options

**Option A: Go + Windows UI Automation.** Direct control, but we own the runner, updates, and desktop-session behaviour.

**Option B: Power Automate Desktop.** The Test API calls Dataverse `RunDesktopFlow`; it can poll status or receive a callback. Unattended PAD creates its own RDP session, not the console session. On Windows 10/11 it cannot run while another user session is active. Test this on the real hardware before choosing it.

## 4:20–5:00 — Ask

Provide one rig and one non-destructive test for the PoC. We will measure GUI stability, session behaviour, network limits, retries, and recovery. Choose the runner from those results, not assumptions.
