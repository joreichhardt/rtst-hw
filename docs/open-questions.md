# Offene Fragen vor der Umsetzung

Diese Fragen sind Umsetzungsvoraussetzungen, keine nachträglichen Optimierungen. Eigentümer und Entscheidungstermin sollten vor dem Produktionsdesign festgelegt werden.

## Priorität 0 – PoC nicht ohne Klärung starten

| Frage | Warum sie entscheidend ist | Vorschlag für Owner |
|---|---|---|
| Welcher konkrete, nicht-destruktive GUI-/Hardwaretest ist der PoC? | Bestimmt UIA-Patterns, Safe-State und Erfolgskriterium. | Test Engineering |
| Welche Windows-Version, Zielanwendung, Treiber und Hardware-Firmware sind verbindlich? | UI Automation und Recovery sind versionssensitiv. | Teststand-Owner |
| Welcher GUI-Runner gilt je Teststand: Go/UIA oder PAD? | Die Betriebs- und Netzanforderungen unterscheiden sich wesentlich. | Plattformteam + Test Engineering |
| Wie wird eine dauerhafte, interaktive Test-Sitzung sicher bereitgestellt (Go-Track)? | Service Session 0 ersetzt keinen interaktiven Desktop. | Windows Workplace / Security |
| Ist der echte Test mit PAD-Unattended-RDP statt Konsolensitzung funktionsfähig? | PAD nutzt laut Microsoft keine Konsolensitzung. | Test Engineering + Hardware-Owner |
| Was geschieht bei RDP-Disconnect, Lock Screen, Console-/RDP-Wechsel und Support-Anmeldung? | Ohne gemessene Policy ist GUI-Automation nicht verlässlich. | Windows Workplace + Test Engineering |
| Wer darf RDP nutzen und wie wird ein aktiver CI-Job geschützt? | Verhindert parallele menschliche und automatisierte Eingaben. | Security + Teststand-Owner |
| Welche LAN-Ziele/Ports darf der Cloud-Workload tatsächlich erreichen? | Legt die minimalen Wormhole- und Firewall-Regeln fest. | Network Security |
| Welche CP-Agent-Version/-Topologie und welche Egress-Anforderungen gelten in unserem Tenant? | CP-Dokumentation muss gegen den konkreten Tenant und das Release validiert werden. | Plattformteam |
| Welche Aktion ist nach einem `UNKNOWN` sicher? | Verhindert gefährliche Doppeltests oder falsche Freigaben. | Test Engineering + Hardware-Owner |

## Priorität 1 – Vor Produktionsfreigabe

| Frage | Zu entscheiden |
|---|---|
| Teststandmodell | Welche Labels/Capabilities, Abhängigkeiten und Wartungsfenster braucht der Scheduler? |
| Datenhaltung | Wie lange speichern wir Screenshots/Logs/Videos; enthalten sie Geheimnisse oder personenbezogene Daten? |
| Identitäten | OIDC-Anbieter für CI, mTLS-/Zertifikatsrotation für Gateway und Agent, Notfallwiderruf? |
| API-Vertrag | Polling oder Callback, Statusmodell, Fehlercodes, Idempotency-Key, Cancel-Semantik und Artefaktformate? |
| Timeouts | Queue-, Lease-, Preflight-, Test-, Upload- und Recovery-Timeout je Testklasse? |
| Wiederholung | Welche Testfälle sind nach `FAILED` retrybar, welche nach `UNKNOWN` nur manuell? |
| Agent-Lebenszyklus | Go-Version, Signierung, Update-Kanal, SBOM, Telemetrie, Rollback und Geräte-Onboarding? |
| UI-Automation | Besteht `uandersonricardo/uiautomation` den PoC hinsichtlich API-Abdeckung, Lizenz, Pflege und Security-Review – oder wird ein eigener dünner UIA-Adapter benötigt? |
| PAD-Governance | Sind Process-Lizenzen, Dataverse-Umgebung, OAuth-Berechtigungen, Machine Connections, Callback-Allowlist und Flow-ALM/Versionierung vorhanden? |
| Verfügbarkeit | Reicht ein lokales Gateway im PoC; wie sehen zwei Gateways und die Auswirkungen auf die Zuordnung zum physischen Teststand aus? |
| Observability | Welche Metriken/Alerts für Queue, Lease, Agent-Ready, Tunnel, UI-Schritte, Hardwarezustand und `UNKNOWN`? |

## Priorität 2 – Betriebsmodell und Skalierung

- Wie werden Windows-Images, Treiber und Testdaten reproduzierbar bereitgestellt und zurückgesetzt?
- Wie werden Hardwareexklusivität, Stromversorgung, USB-/COM-Port-Besitz und gemeinsame Messgeräte modelliert?
- Benötigen Teams Kostenstellen, Quoten, Prioritäten, manuelle Buchungsfenster oder Approval?
- Welchen Durchsatz, welche Laufzeiten und welche geografischen Standorte erwarten wir in 12 Monaten?
- Welcher Support-Level, welche Bereitschaft und welche Eskalationszeit gilt bei einem quarantänisierten Teststand?
- Welche Nachweise brauchen Audit, Informationssicherheit und Lieferantenschutz für Artefakte und den lokalen Tunnel?

## Entscheidungsausgang dokumentieren

Für jede beantwortete Frage erfassen: Entscheidung, Datum, Owner, betroffene Teststände, akzeptiertes Restrisiko und Link zum Nachweis (PoC-Test, Policy oder Runbook). So bleiben Architekturannahmen nicht stillschweigend produktiv.
