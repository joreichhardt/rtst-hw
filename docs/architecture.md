# Technische Architektur und Ablauf

## 1. Entscheidungen, Annahmen und geprüfte Fakten

### Vorgeschlagene Architekturentscheidungen

- **Cloud:** Test-API, Scheduler, Auftrags-/Ergebnisdatenbank und Artefaktspeicher laufen als Control-Plane-Workload.
- **Lokales Netz:** Ein dediziertes Test-Gateway ist der einzige Dienst, den der Cloud-Workload über Wormhole ansprechen darf.
- **GUI-Runner, Option A – Go-Agent:** Ein eigener, in **Go** implementierter Agent läuft auf jedem Test-PC. Er führt den Test in einer angemeldeten interaktiven Sitzung über die Windows UI Automation API aus. Als PoC-Kandidat gilt das vom Team vorgeschlagene Go-Binding `github.com/uandersonricardo/uiautomation`; Kompatibilität, Wartungsstand, Lizenz und Rechtebedarf sind im PoC zu bestätigen.
- **GUI-Runner, Option B – Power Automate Desktop (PAD):** Die Test-API startet einen Desktop Flow via Dataverse `RunDesktopFlow`, übergibt Flow-ID, Maschinen-/Maschinengruppen-Verbindung und Eingabewerte und verarbeitet `flowsessionId` über Callback oder Polling. Damit kann PAD den Go-GUI-Runner ersetzen, nicht aber Test-API, Scheduler oder Reservierungslogik. [MS-1]
- **Keine Fern-GUI-Steuerung:** Der Cloud-Workload greift nie direkt per RDP oder UI Automation auf Windows zu.
- **RDP:** Ausschließlich Menschen verwenden RDP für Provisionierung, Diagnose und manuelle Wiederherstellung.

### Verifizierte Aussagen zu Control Plane Wormhole

Stand **23.09.2026** dokumentiert Control Plane:

1. Ein Wormhole-**Agent** verbindet Control-Plane-Workloads mit TCP- oder UDP-Endpunkten in VPCs und anderen privaten Netzen, einschließlich On-Premises-Netzen. Er läuft im privaten Netz, baut eine sichere persistente Verbindung zu öffentlich gehosteten Control-Plane-Servern auf und tunnelt Anfrage und Antwort. [CP-1]
2. Agents sind organisationsbezogen und werden zusammen mit Identities verwendet. [CP-1]
3. Version-2-Agents arbeiten aktiv/aktiv. Bei ausgebliebenen Heartbeats gelten sie als offline; verbleibende aktive Agents übernehmen Arbeit. Für hochverfügbare Deployments empfiehlt die Dokumentation eine feste Instanzgruppe mit mindestens zwei Instanzen. [CP-1]
4. Die v2-Bidirektionalitätsfunktion enthält einen Proxy auf Port 3128 für Verbindungen **aus** dem privaten Netz zu Control-Plane-Workloads. [CP-1]

**Wichtige Grenze:** Die Quelle beschreibt Netzwerktransport zu TCP/UDP-Endpunkten. Sie beschreibt weder Windows-UI-Automation noch Windows-Login, RDP-Sitzungen, Bildschirmsperren oder Hardwaretests. Für diese Funktionen gibt es in diesem Entwurf keine Zuschreibung an Control Plane.

### Zu prüfende Annahmen

| Annahme | Warum nötig | Nachweis im PoC |
|---|---|---|
| Der v2-Wormhole-Agent kann auf einer vom Windows-PC getrennten Gateway-VM im Test-LAN betrieben werden. | Der Tunnel soll nicht auf jedem Test-PC laufen. | Test-Gateway hinter Wormhole mit nur einem erlaubten Endpunkt erreichbar. |
| Das Gateway ist als lokaler TCP-Dienst hinter Wormhole erreichbar. | Die Test-API muss Aufträge/Ergebnisse vermitteln. | End-to-end API-Request aus dem Cloud-Workload. |
| Der Go-Agent kann die Zielanwendung stabil per Windows UI Automation bedienen. | Option A ist eine mögliche Ausführungsstrategie. | Wiederholbarer Happy Path und definierte Fehlerdiagnostik. |
| PAD kann denselben Teststand via API/Dataverse zuverlässig ausführen. | Option B kann Eigenentwicklung des GUI-Runners reduzieren. | Ein API-gestarteter Desktop Flow auf echter Hardware. |
| Eine für den gewählten Runner passende Windows-Sitzung bleibt während CI-Jobs nutzbar. | GUI-Automation hängt vom Desktopmodus ab. | RDP-Trennung/Sperre kontrolliert reproduzieren. |

## 2. Komponenten

| Komponente | Ort | Kernaufgaben | Persistenz / Vertrauensgrenze |
|---|---|---|---|
| CI-Adapter | CI-System | Fordert Test an, pollt oder empfängt Callback, lädt Artefakte | besitzt nur kurzlebige API-Berechtigung |
| Test-API | Cloud | Authentisiert, validiert Testanfragen; liefert Status und Artefakt-URLs | authoritative API-Grenze |
| Scheduler | Cloud | Matcht Anforderungen auf freie Teststände; vergibt Lease und Fencing-Token | atomare Reservierungsdaten |
| Auftrags-/Ergebnisdatenbank | Cloud | Zustände, Versuche, Reservierungen, Auditdaten | Quelle der Wahrheit |
| Artefaktspeicher | Cloud | Immutable Logs, Screenshots, optional Video/Diagnosebundle | zeitlich begrenzte Download-URLs |
| Cloud-Workload | Cloud | Hält die Verbindung zur Gateway-API über Wormhole | keine GUI-/RDP-Rechte |
| **Control Plane Wormhole Agent** | Gateway-VM im lokalen Netz | Netzwerktransport zwischen Cloud-Workload und privatem Gateway | nicht unser Testausführer |
| Test-Gateway | Lokales Netz | Authentisiert Testagenten; vermittelt Befehle/Events; puffert bei kurzer Cloud-Störung | einzige über Wormhole exponierte LAN-API |
| **Windows-Testagent (Go), Option A** | jeder Windows-Teststand | Zustands-Heartbeat, UI-Automation, Hardwarezugriff, Screenshot/Log-Sammlung | arbeitet nur in lokaler interaktiver Sitzung |
| **Power Automate Desktop, Option B** | jeder Windows-Teststand | Führt einen von Dataverse gestarteten Desktop Flow aus; gibt Flow-Status/-Ausgaben zurück | separate Power-Platform-Identität und Lizenz nötig [MS-1][MS-2] |
| Windows-Testkonto | jeder Windows-Teststand | Dediziertes Konto für Testdesktop bzw. PAD-Verbindung | keine tägliche Admin-Nutzung |
| RDP-Zugang | lokales Netz | Manuelle Einrichtung, Beobachtung, Recovery | niemals regulärer CI-Ausführungsweg |

## 3. Auftragsablauf

```mermaid
sequenceDiagram
  participant C as CI
  participant A as Test-API
  participant S as Scheduler/DB
  participant G as Lokales Test-Gateway
  participant W as Windows-Testagent
  participant H as Hardware / GUI

  C->>A: POST /test-jobs (Anforderung, Idempotency-Key)
  A->>S: Auftrag anlegen oder bestehenden zurückgeben
  S-->>A: QUEUED, jobId
  A-->>C: 202 Accepted, Status-URL
  S->>S: passenden freien Teststand atomar leasen
  S->>G: Job(jobId, attempt, lease, fencingToken)
  G->>W: Auftragsangebot
  W->>W: interaktive Sitzung + Hardware-Check
  alt bereit und Lease gültig
    W-->>G: ACK mit Agent-/Sitzungsstatus
    G-->>S: RUNNING
    W->>H: GUI- und Hardwaretest
    W->>G: Events, Logs, Screenshots
    G->>A: Ergebnisse und Artefakt-Referenzen
    A-->>C: SUCCEEDED / FAILED + signierte Artefakt-URLs
  else nicht bereit, Lease abgelaufen oder Fencing-Token falsch
    W-->>G: REJECTED / UNKNOWN
    G-->>S: nicht ausführen; Diagnose speichern
  end
```

### Zustandsmodell und erneute Zustellung

`QUEUED → RESERVED → DISPATCHED → RUNNING → {SUCCEEDED | FAILED | UNKNOWN | CANCELLED}`

- Der Scheduler reserviert den Teststand transaktional und vergibt `leaseId`, Ablaufzeit und monotonen `fencingToken`.
- Test-Gateway und Windows-Testagent akzeptieren nur den aktuellsten Token für den Teststand.
- Eine Nachricht wird **mindestens einmal** zugestellt; deswegen muss `jobId + attempt + fencingToken` idempotent verarbeitet werden.
- Bestätigung einer *Zustellung* ist keine Bestätigung einer *Hardwareausführung*. Erst `RUNNING` nach lokalem Preflight zählt als Start.
- Bei verlorenem Ergebnis nach `RUNNING` wird der Auftrag `UNKNOWN`. Der Scheduler darf ihn erst nach menschlicher oder testfallspezifischer, sicherer Entscheidung wiederholen; blindes Wiederholen kann Hardwarezustand verändern.

### Parallelität und Reservierung

- Kapazität eines Teststands: **1** paralleler GUI-/Hardwarejob.
- Verschiedene Teststände können parallel arbeiten, sofern sie keine exklusive gemeinsame Hardware, Stromversorgung oder Testdaten teilen.
- Der Scheduler prüft Labels/Capabilities (Hardwaretyp, Firmware, Softwareversion, Standort), Quarantäne, Wartungsfenster und maximale Laufzeit.
- Die Reservierung endet erst nach terminalem Ergebnis oder kontrollierter Recovery. Ein Heartbeat-Ausfall allein entsperrt Hardware nicht sofort.

## 4. Sicherheit und Netzwerk

1. **Eingrenzung:** Der Cloud-Workload darf nur den Gateway-Host und dessen festgelegten TCP-Port über Wormhole erreichen; keine breiten LAN-Subnets und keine direkten Windows-PCs.
2. **Richtung:** Der Agent baut laut CP-Dokumentation die persistente Verbindung nach außen auf. Die konkret erforderlichen Egress-Ziele, Ports, DNS-/Proxy-/TLS-Inspection-Ausnahmen werden aus der beim PoC geltenden CP-Konfiguration übernommen und von Netzwerkbetrieb freigegeben – nicht aus dieser Planung geraten.
3. **Doppelte Authentisierung:** CI→Test-API via OIDC-Workload-Identity oder kurzlebigem CI-Token; API/Gateway und Gateway/Windows-Agent jeweils mTLS mit rotierbaren, gerätespezifischen Credentials.
4. **Autorisierung:** CI darf nur freigegebene Testprofile/Umgebungen anfordern. Der Agent akzeptiert nur signierte, allow-gelistete Testdefinitionen, keine beliebigen Shell-Befehle.
5. **Artefakte:** Keine Zugangsdaten oder personenbezogene Testdaten in Screenshots/Logs. Verschlüsselte Speicherung, kurze signierte URLs, Retention und Zugriffsaudit.
6. **RDP:** Firewall- und Rollenrichtlinie nur für Support-Gruppe; separate Konten, MFA, Protokollierung. Während eines aktiven Jobs sperrt die Plattform den manuellen RDP-Zugriff organisatorisch/technisch oder setzt den Job auf `UNKNOWN`.

## 5. Zwei GUI-Runner-Optionen und Windows-Sitzung

### Option A: eigener Go-Agent direkt gegen Windows UI Automation

Windows UI Automation adressiert die UI im interaktiven Desktop. Ein Dienst in Session 0 ist daher nicht als Ersatz für eine sichtbare Benutzersitzung zu planen.

**Entwurf:** Der Go-Testagent besteht aus einem kleinen Windows-Service für Liveness/Update/Bootstrap und einem pro Testkonto gestarteten interaktiven Worker für UI Automation. Nur der Worker darf einen Job als `RUNNING` markieren, nachdem er Benutzer, Session-ID, Desktop-Zustand, Zielanwendung und Hardwareverbindung geprüft hat.

### Option B: Power Automate Desktop über Dataverse

Microsoft dokumentiert `RunDesktopFlow` als Dataverse-Aktion zum Starten eines Desktop Flows aus einer Anwendung. Sie benötigt Flow-ID und Desktop-Flow-Verbindung (zur Maschine bzw. Maschinengruppe); Status/Ausgaben sind über `flowsessionId` abfragbar. Ein HTTPS-Callback kann den Abschluss melden, wobei der Empfänger idempotent sein muss; Polling bleibt Fallback. [MS-1]

Für unbeaufsichtigte Ausführung erstellt PAD laut Microsoft pro Lauf eine RDP-Sitzung, führt den Flow darin aus und meldet den Benutzer danach ab. Es verbindet sich nicht mit der Konsolensitzung; die Sitzung ist gesperrt. Windows 10/11 können keine unbeaufsichtigten Flows ausführen, wenn irgendeine aktive Windows-Benutzersitzung besteht (auch gesperrt). Eine Process-Lizenz ist erforderlich. [MS-2]

**Architekturfolgen:** PAD kann den Go-GUI-Runner ersetzen. Dann liegt der Ausführungsweg bei `Test-API → Dataverse/Power-Automate-API → PAD auf Windows → Dataverse → Test-API`; der Wormhole-Pfad ist dafür nicht erforderlich. Wormhole bleibt als separat zu bewertender, eng begrenzter Zugang zu lokalen Gateway-Diensten sinnvoll, falls diese weiterhin benötigt werden. Entscheidend ist: Der POC muss mit der *echten* Hardware nachweisen, dass die von PAD erzeugte RDP-Sitzung für Anwendung und Hardware geeignet ist. Konsolensitzungsabhängige Tests sind ein No-Go für diesen PAD-Modus.

| Risiko | Erwartete Auswirkung | Vorgesehene Behandlung im PoC |
|---|---|---|
| RDP-Trennung | Bei Go hängt das Verhalten von Richtlinie/Client ab; PAD-Unattended erstellt eine eigene RDP-Sitzung statt der Konsole. | Beide Modi auf Ziel-Windows-Version messen; nur bestätigtes Verhalten produktiv zulassen. |
| Bildschirmsperre | Go-UIA kann unzugänglich werden; PAD-Unattended hält den Zielbildschirm gesperrt. | Go-Worker beendet sicher mit `BLOCKED_SESSION`; PAD-Flow unter der realen Sperr-/RDP-Bedingung testen. |
| RDP-Anmeldung durch Support während Job | Desktop-Konflikt und unzuverlässige Interaktion. | Reservierungsanzeige, RDP-Sperrprozess und Audit; Job wird nicht fortgesetzt, wenn der Desktop wechselte. |
| Dialog/UAC/Update | Modal-Dialog blockiert Ablauf. | Definierte Allowlist, Screenshot, Timeout, `FAILED_ENVIRONMENT`; keine generische Dialogklick-Automation. |
| Neustart/Crash | Hardware kann in unbekanntem Zustand verbleiben. | Watchdog, Quarantäne und manueller Recovery-Runbook. |

## Quellen

- **[CP-1]** Control Plane, *Agent* (Wormhole-Referenz): https://docs.controlplane.com/reference/agent — abgerufen am 23.09.2026.
- **[CP-2]** Control Plane, *Introduction* und Dokumentationsindex: https://docs.controlplane.com/introduction und https://docs.controlplane.com/llms.txt — abgerufen am 23.09.2026.
- **[MS-1]** Microsoft Learn, *Work with desktop flows using code*: https://learn.microsoft.com/en-us/power-automate/developer/desktop-flow-public-apis — abgerufen am 23.09.2026.
- **[MS-2]** Microsoft Learn, *Run unattended desktop flows*: https://learn.microsoft.com/en-us/power-automate/desktop-flows/run-unattended-desktop-flows — abgerufen am 23.09.2026.

Die Quellen belegen die genannten Wormhole- bzw. PAD-Netzwerk-/API-/Sitzungsfunktionen. Alle Aussagen zu Test-Gateway, Go-Agent, UI Automation, API, Scheduler und Windows-Betrieb sind bewusst **unser Architekturvorschlag** und müssen im PoC validiert werden.
