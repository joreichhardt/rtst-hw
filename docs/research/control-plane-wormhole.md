# Quellenprüfung: Control Plane Wormhole

**Abgerufen:** 23.09.2026  
**Zweck:** Nur produktspezifische Aussagen, die in dieser Architekturplanung verwendet werden.

## Verifizierte Fakten

| Aussage | Beleg |
|---|---|
| Der Control-Plane-Wormhole-Agent verbindet Workloads mit TCP-/UDP-Endpunkten in privaten Netzen, einschließlich On-Premises-Netzen. Der Agent wird im privaten Netz betrieben und baut eine sichere persistente Verbindung zu Control-Plane-Servern auf; Anfragen und Antworten werden darüber getunnelt. | [Agent reference, Overview](https://docs.controlplane.com/reference/agent) |
| Agents sind organisationsbezogen und werden mit Identities verwendet. | [Agent reference, Overview](https://docs.controlplane.com/reference/agent) |
| Version-2-Agents laufen aktiv/aktiv; bei fehlenden Heartbeats übernehmen verbleibende Agents Arbeit. Control Plane empfiehlt für HA eine feste Gruppe mit mindestens zwei Instanzen. | [Agent reference, Version 2](https://docs.controlplane.com/reference/agent) |
| Die v2-Bidirektionalitätsfunktion bietet einen Proxy auf Port 3128 für Aufrufe aus dem privaten Netz zu Control-Plane-Workloads. | [Agent reference, Bi-directional Functionality](https://docs.controlplane.com/reference/agent) |

## Nicht von der Quelle behauptet

Die geprüfte Wormhole-Referenz belegt **nicht**, dass Control Plane:

- Windows UI Automation, RDP, Windows-Logon oder eine interaktive Sitzung verwaltet,
- GUI-Tests oder physische Hardwaretests ausführt,
- einen Scheduler, Reservierungssemantik oder genau-einmalige Hardwareausführung bereitstellt,
- konkrete Egress-Firewall-Ziele/Ports für den individuellen Tenant ersetzt.

Diese Themen sind in den Planungsunterlagen entweder eigene Komponenten oder explizite PoC-/Team-Entscheidungen.

## Konsequenz für den Entwurf

Wormhole ist als eingeschränkter Netzwerkpfad zum **lokalen Test-Gateway** vorgesehen. Der Windows-Testagent (oder alternativ Power Automate Desktop) bleibt der GUI-Runner. Dadurch hängen die Richtigkeits- und Windows-Sitzungsanforderungen nicht an einer nicht belegten Produktaussage.

## Weitere Einstiegsquellen

- [Control Plane Introduction](https://docs.controlplane.com/introduction)
- [Control Plane Documentation Index](https://docs.controlplane.com/llms.txt)
