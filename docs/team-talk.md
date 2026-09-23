# Fünf-Minuten-Gespräch: Hardwaretests aus CI

## 0:00–0:40 – Problem

„Heute sitzen Mitarbeitende per RDP auf Windows-Testständen und klicken GUI-Tests an echter Hardware. Das funktioniert für Einzeltests, ist aber nicht buchbar, schlecht parallelisierbar und liefert CI keine belastbaren Artefakte.“

**Zielbild:** CI bucht einen passenden Teststand, führt exakt einen Test aus und erhält Status, Logs und Screenshots zurück.

## 0:40–1:30 – Architektur in einem Bild

Zeige [das Diagramm im README](../README.md#zielarchitektur).

„Cloud-seitig laufen Test-API und Scheduler. Lokal steht ein eng begrenztes Test-Gateway. Control Plane Wormhole verbindet den Cloud-Workload lediglich mit diesem Gateway. Für die GUI vergleichen wir zwei Runner: unseren Go-Testagenten mit Windows UI Automation in einer interaktiven Sitzung sowie Power Automate Desktop, das die API über Dataverse startet. RDP bleibt ein menschliches Diagnosewerkzeug.“

**Merksatz:** *Wormhole transportiert Netzwerkverkehr; Go-Agent oder PAD testen die GUI.*

## 1:30–2:25 – Zuverlässigkeit und Parallelität

„Der Scheduler reserviert einen Teststand atomar: pro Teststand läuft maximal ein Hardwaretest. Ein Lease plus Fencing-Token verhindert, dass eine verspätete erneute Zustellung einen alten Auftrag ausführt. Nachrichten dürfen erneut kommen, Hardwaretests nicht doppelt.“

„Wenn ein Job nach dem Start wegen eines Ausfalls nicht aufklärbar ist, heißt er `UNKNOWN`, nicht ‚fehlgeschlagen und sofort wiederholen‘. Der Stand wird quarantänisiert, bis ein definierter Recovery-Schritt erfolgt.“

## 2:25–3:20 – Sicherheit

„Die Cloud sieht nicht das gesamte Test-LAN. Über Wormhole ist nur der Gateway-Endpunkt erreichbar, nicht RDP und nicht jeder Windows-PC. CI, Gateway und Testagent authentisieren sich getrennt mit kurzlebigen bzw. rotierbaren Identitäten. Testdefinitionen sind allow-gelistet – kein Remote-Shell-Service.“

„Control Plane dokumentiert Wormhole als Tunnel zu privaten TCP/UDP-Endpunkten und v2-Agenten als aktiv/aktiv. Die Dokumentation behauptet jedoch keine Windows-GUI-Automation; genau deshalb ist das eine klar getrennte eigene Komponente.“

## 3:20–4:20 – Windows-Risiko ehrlich ansprechen

„Beim Go-Agenten benötigt UI-Automation die interaktive Sitzung. RDP-Trennung, Sperrbildschirm, Support-Anmeldung, UAC und Updates können sie verändern. Der Agent prüft den Desktop vor dem Start und bricht sicher ab."

„PAD ist eine ernsthafte Alternative: Microsoft dokumentiert `RunDesktopFlow` per Dataverse mit Status-Polling oder Callback. Unattended PAD erzeugt jedoch eine eigene RDP-Sitzung, nicht die Konsole; auf Windows 10/11 darf dabei keine andere aktive Sitzung bestehen. Der PoC führt deshalb einen echten Hardwaretest in beiden Modi aus. Das vorgeschlagene `uandersonricardo/uiautomation` bleibt ein PoC-Kandidat, keine unvalidierte Produktionsentscheidung.“

## 4:20–5:00 – Bitte ans Team

„Lasst uns einen Teststand und einen nicht-destruktiven Testfall für den PoC freigeben. Nach acht klaren Abnahmephasen entscheiden wir anhand von Messdaten: UI-Stabilität, RDP-/Sperrverhalten, Netzwerkgrenzen und Recovery. Erst dann skalieren wir auf weitere Hardware.“

## Erwartete Rückfragen – Kurzantworten

| Frage | Antwort |
|---|---|
| „Kann Control Plane die Windows-GUI bedienen?“ | Nein, das ist nicht als Wormhole-Funktion belegt. Wormhole ist der Netzwerkpfad; der Go-Agent bedient die GUI. |
| „Warum nicht einfach RDP aus CI?“ | RDP koppelt CI an einen fragilen menschlichen Desktopzugriff, erschwert Sicherheit/Parallelität und liefert keinen idempotenten Auftragsvertrag. |
| „Was passiert bei Verbindungsverlust?“ | Vor Start erneut zustellbar; nach tatsächlichem Start konservativ `UNKNOWN` plus Quarantäne, bis Recovery entschieden ist. |
| „Können mehrere Teams gleichzeitig testen?“ | Ja, auf verschiedenen reservierten Testständen; niemals gleichzeitig auf demselben Stand. |
