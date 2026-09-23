# Proof of Concept: buchbarer Windows-Hardwaretest

## Ziel und Nicht-Ziel

**Ziel:** Mit einem Windows-Teststand zeigen, dass eine CI-Pipeline einen exklusiv reservierten GUI-/Hardwaretest starten und ein nachvollziehbares Ergebnis mit Logs und Screenshot erhält. Der PoC vergleicht zwei austauschbare GUI-Runner: Go/UI Automation über den minimal freigegebenen Wormhole-Pfad und Power Automate Desktop (PAD) über dessen Dataverse-API.

**Nicht-Ziel:** Produktionsreife Hochverfügbarkeit, flächendeckendes Device Management, vollwertiges Testauthoring oder Migration aller bestehenden RDP-Abläufe.

## Testaufbau

- 1 Cloud-Workload mit einfacher Test-API/Scheduler-Attrappe
- 1 lokales Test-Gateway hinter einem Control-Plane-v2-Wormhole-Agent
- 1 Windows-Test-PC mit Zielprogramm und physischer Hardware
- 1 dediziertes Windows-Testkonto, vorbereitete interaktive Sitzung
- **Track A:** 1 Go-Testagent mit Windows-UI-Automation-Prototyp (Kandidat: `uandersonricardo/uiautomation`)
- **Track B:** 1 API-startbarer PAD-Desktop-Flow mit Maschinen-/Maschinengruppen-Verbindung und passender Process-Lizenz
- 1 CI-Job, der eine Testanforderung stellt, den Status abfragt und Artefakte herunterlädt

## Phasen und Abnahme

| Phase | Arbeitsergebnis | Prüfschritt | Abnahmekriterium |
|---|---|---|---|
| 0. Voraussetzungen | Asset-Inventar, Windows-/RDP-Policy, Netzwerkfreigabe, Testfall | Team bestätigt einen nicht-destruktiven Testfall und Recovery-Owner. | Kein produktiver Test startet ohne dokumentierten Safe-State/Recovery. |
| 1. Netzwerkpfad | Gateway ist über Wormhole gezielt erreichbar | Cloud-Workload ruft genau einen TLS-Health-Endpunkt am Gateway auf. | Kein Zugriff auf Windows-PC, RDP oder beliebige LAN-Ziele möglich; Regel durch Netztest belegt. |
| 2A. Agent-Liveness | Go-Service + interaktiver Worker melden Zustand | Gateway zeigt Version, Teststand-ID, Session-ID und Bereitschaft. | Ein gesperrter/fehlender Desktop ist explizit `NOT_READY`, nie fälschlich bereit. |
| 2B. PAD-Vertrag | Dataverse `RunDesktopFlow`, `flowsessionId`, Callback/Polling | Test-API startet den Flow und korreliert Status/Ausgaben idempotent mit `jobId`. | Callback ist erreichbar und idempotent; Polling liefert denselben terminalen Status; Berechtigung und Lizenz sind dokumentiert. |
| 3. GUI-Automation | Ein kleiner stabiler Zielablauf in jedem gewählten Track | Runner startet Zielprogramm, liest eindeutiges UI-Element, führt nicht-destruktive Aktion aus. | 10 aufeinanderfolgende Durchläufe ohne manuelle UI-Intervention; bei Fehler Screenshot + strukturierter Grund. |
| 3B. Konsolenabhängigkeit | PAD-Unattended gegen reale Hardware | Test während PAD-erzeugter RDP-Sitzung ausführen. | Die Hardware und Zielanwendung funktionieren darin nachweisbar – oder PAD wird für diesen Teststand ausgeschlossen. |
| 4. Buchung | Atomare Teststand-Reservierung | Zwei parallele CI-Anfragen fordern denselben Teststand. | Genau eine Anfrage läuft; die andere bleibt `QUEUED` oder erhält definierte Kapazitätsantwort. |
| 5. E2E-Ergebnis | CI-Vertrag | CI fordert Test an und lädt Artefakte. | `jobId`, terminaler Status, Zeitstempel, Logs und mindestens ein Screenshot sind abrufbar. |
| 6. Fehlerfälle | Runbook und Zustandsmodell | Netzpfad, Gateway und Agent nacheinander unterbrechen. | Keine Doppel-Ausführung; nach gestartetem, unaufklärbarem Job Status `UNKNOWN` + Quarantäne/Recovery-Hinweis. |
| 7. RDP/Sperre | Windows-Verhalten gemessen | Für Go: RDP trennen/sperren/wiederverbinden. Für PAD: eigene unbeaufsichtigte RDP-Sitzung, gesperrten Bildschirm und aktive Benutzer-Sitzung testen. | Verhalten ist je Versuch dokumentiert; der Go-Agent blockiert/abbricht sicher. PAD ist auf Windows 10/11 ausgeschlossen, falls irgendeine aktive Sitzung besteht, wie Microsoft dokumentiert. |
| 8. Sicherheitsreview | Bedrohungs- und Berechtigungsmatrix | Credential-Rotation, unzulässige Testdefinition und Artefaktzugriff testen. | Keine dauerhaften CI-Geheimnisse in Windows; nicht erlaubter Test wird vor Ausführung abgewiesen. |

## Messprotokoll

Für jeden Lauf speichern: `jobId`, `attempt`, `fencingToken`, Teststand, Agent-/Gateway-Version, Session-ID (pseudonymisiert), Start/Ende, Scheduler-Entscheidung, Ergebnis, Artefakt-Hashes und Recovery-Aktion.

Zusätzliche Kennzahlen:

- Queue-Wartezeit und Reservierungsdauer
- Zeit von Auftrag bis Agent-ACK und bis Ergebnis
- Erfolgsquote je Testfall und Fehlerklasse
- Tunnel-/Gateway-/Agent-Disconnects
- Anzahl `UNKNOWN` und benötigte manuelle Recoveries

## Explizite Go/UI-Automation-Prüfungen (Track A)

1. Kann das Binding die für die Zielanwendung nötigen UIA-Patterns lesen und auslösen?
2. Funktioniert der Worker im Testkonto und in dessen interaktiver Sitzung – nicht nur in einer Entwickler-RDP-Sitzung?
3. Wie verhalten sich Element-Suche und Screenshots bei Lock Screen, RDP-Disconnect, RDP-Reconnect, Modal-Dialog und UAC?
4. Gibt es reproduzierbare Selektoren (AutomationId/ControlType) statt bildbasierter Koordinatenklicks?
5. Können jeder Schritt, UIA-Fehler und Screenshot ohne sensible Werte protokolliert werden?

## Entscheidungsmatrix und Go/No-Go

| Kriterium | Go-Agent / UI Automation | PAD / Dataverse |
|---|---|---|
| Netzpfad | Wormhole zum lokalen Gateway bleibt Kern des Tests. | Start/Status über Microsoft Cloud; Wormhole für den GUI-Start nicht nötig. |
| Windows-Desktop | Kontrollierte interaktive Sitzung erforderlich. | Unattended erstellt RDP-Sitzung statt Konsole und sperrt Bildschirm. |
| Eigenentwicklung | Höher: Worker, Update, UIA-Adapter, Artefakte. | Niedriger: Flow-Entwicklung und Power-Platform-Integration. |
| Harter Ausschluss | Ziel-UI ist nicht stabil automatisierbar. | Hardware/Anwendung benötigt Konsolensitzung oder aktive Windows-Sitzung verhindert den Lauf. |

**Go** nur wenn Phase 1, Phase 4–8 sowie mindestens einer der Tracks 2A/3 oder 2B/3/3B erfüllt sind und das Team die in [Offene Fragen](open-questions.md) priorisierten Entscheidungen trifft.

**No-Go oder Architekturwechsel**, wenn der Zieltest in keinem Runner zuverlässig ausgeführt werden kann, die RDP-/Lock-Policy den gewählten Desktopmodus unbrauchbar macht oder die gewünschte Netzwerkbegrenzung nicht nachweisbar ist. Mögliche Alternativen sind ein physischer/virtueller Konsolenzugang oder Schnittstellen-/Protokollautomation statt GUI.
