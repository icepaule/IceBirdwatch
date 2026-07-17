# IceBirdwatch

Lokale, KI-gestützte Vogelerkennung für ein Solar-Vogelbad mit Kamera —
**ohne Cloud-Abo**, komplett auf eigener Infrastruktur.

![WAVEEME Vogelhaus mit Kamera, Solar und Vogeltränke](docs/images/produkt-hauptbild.jpg)

## Ausgangslage

Das Gerät ([WAVEEME Vogelhaus mit Kamera, 3K, Solar, KI-Vogelerkennung & Vogeltränke](https://www.amazon.de/dp/B0GJS3ZBTQ),
ca. 67€) bringt eine herstellereigene App samt Cloud-KI mit — die
Vogelerkennung ist dort nur 1 Monat kostenlos, danach vermutlich
abopflichtig. Ziel dieses Projekts: die Artbestimmung stattdessen über eine
**selbst gehostete Ollama-Vision-KI** laufen zu lassen und die Ergebnisse in
**Home Assistant** anzuzeigen — ganz ohne Abo und ohne dass Bilder das
Heimnetz verlassen.

## Architektur

```mermaid
flowchart LR
    subgraph Garten
        CAM["WAVEEME Kamera<br/>(Xiongmai/DVRIP bestaetigt)<br/>DVRIP :34567"]
    end

    subgraph "prox2 — CT203 frigate01"
        CROP["go2rtc exec-Stream<br/>Privacy-Crop<br/>(Nachbarhaus raus)"]
        FRIGATE["Frigate NVR<br/>Bewegungs-/Objekt-<br/>erkennung (bird)"]
    end

    subgraph "ki02 (10.10.0.210)"
        N8N["n8n Workflow<br/>MQTT-Trigger"]
        OLLAMA["Ollama<br/>qwen2.5vl:7b<br/>Artbestimmung"]
        JSONFILE[("vogelbad_historie.json<br/>(im n8n-Datenverzeichnis)")]
    end

    subgraph "NUC-HA (10.10.0.100)"
        MQTT{{"Mosquitto<br/>MQTT-Broker"}}
        RELAY["TCP-Relay<br/>(socat, VLAN12-Ausnahme)"]
        HA["Home Assistant"]
        DASH["Dashboard 'Vogelbad'<br/>+ Kiosk-Banner"]
    end

    CAM -- DVRIP-Stream --> RELAY
    RELAY -- weitergeleitet --> CROP
    CROP -- "zugeschnittener Stream" --> FRIGATE
    FRIGATE -- "MQTT: frigate/reviews<br/>(objects=[bird], Ende)" --> MQTT
    MQTT -- Event --> N8N
    N8N -- "Snapshot holen<br/>(Frigate API)" --> FRIGATE
    N8N -- "Bild + Prompt" --> OLLAMA
    OLLAMA -- "JSON: Art, lateinischer<br/>Name, Konfidenz" --> N8N
    N8N -- anhaengen --> JSONFILE
    N8N -- "MQTT: home/vogelbad/erkennung" --> MQTT
    MQTT --> HA
    HA --> DASH
    N8N -. "Webhook /vogelbad-historie<br/>(REST-Sensor, Polling)" .-> HA
```

**Warum n8n als Zwischenschicht statt direkt Frigate → Ollama?** Damit sich
der Ollama-Prompt (Sprache, gewünschtes Antwortformat, zusätzliche Hinweise
wie Region/Jahreszeit) jederzeit **ohne Code-Änderung** in der n8n-UI
anpassen lässt.

## Komponenten

| Komponente | Host | Zweck |
|---|---|---|
| TCP-Relay (socat) | NUC-HA (systemd-Service) | Leitet die IoT-VLAN-isolierte Kamera an CT203 weiter, das selbst keinen VLAN-Zugriff hat |
| Frigate | prox2 / CT203 (`frigate01`) | Nimmt DVRIP-Stream (über Relay), erkennt Bewegung + Objektklasse "bird", erzeugt Snapshots |
| n8n | ki02 (10.10.0.210:5678) | Orchestriert Erkennung → Ollama → Speicherung → MQTT |
| Ollama (qwen2.5vl:7b) | ki02 (10.10.0.210:11434) | Vision-KI für Artbestimmung (Umgangssprachlich + lateinisch) |
| Mosquitto | NUC-HA (10.10.0.100:1883) | MQTT-Broker (Home-Assistant-Addon) |
| Home Assistant | NUC-HA | Dashboard, Kiosk-Benachrichtigung, Sensor-Historie |

## Status

- [x] Frigate-Container läuft, Kamera per DVRIP über TCP-Relay eingebunden
- [x] n8n-Workflows importiert + aktiviert, End-to-End getestet (Ollama antwortet korrekt)
- [x] Home Assistant neu geladen — Dashboard + Kiosk-Banner aktiv
- [x] Kamera draußen am Vogelbad montiert
- [x] Privacy-Crop aktiv (Nachbarhaus rechts ausgeblendet, siehe docs/setup.md)
- [ ] Erste echte Vogel-Erkennung abgewartet

## Dokumentation

| Dokument | Inhalt |
|---|---|
| [docs/hardware.md](docs/hardware.md) | Kamera-Hintergrund, Xiongmai-Diagnose, Produktbilder |
| [docs/setup.md](docs/setup.md) | Komplette Schritt-für-Schritt-Anleitung (Kamera → Frigate → n8n → HA), Privacy-Crop |
| [docs/ki-erkennung.md](docs/ki-erkennung.md) | Die lokale KI-Erkennung im Detail: n8n-Workflow node-für-node, Ollama-Prompt |
| [docs/homeassistant.md](docs/homeassistant.md) | Home-Assistant-Dashboard, Kiosk-Banner, Entities — mit Screenshots |

## Home Assistant Dashboard

![Home Assistant Dashboard - Übersicht](docs/images/ha-dashboard-uebersicht.png)

Live-Bild (bereits privacy-gecroppt), aktuelle Art, Konfidenz, Historie —
Details und weitere Screenshots in [docs/homeassistant.md](docs/homeassistant.md).

## Keine sensiblen Daten in diesem Repo

Alle Beispiel-Configs verwenden Platzhalter (`TODO-KAMERA-IP`,
`REPLACE_MIT_..._CREDENTIAL_ID`) bzw. referenzieren Home-Assistant-`!secret`-
Einträge. Echte Zugangsdaten, IPs und Passwörter liegen ausschließlich in den
privaten `secrets.yaml`/Credential-Stores der jeweiligen Systeme, nicht hier.
