# Setup

## Ablauf einer Erkennung

```mermaid
sequenceDiagram
    participant Cam as WAVEEME Kamera
    participant Frigate
    participant MQTT as Mosquitto
    participant n8n
    participant Ollama
    participant DB as SQLite
    participant HA as Home Assistant

    Cam->>Frigate: RTSP-Stream
    Frigate->>Frigate: Bewegung + Objekt "bird" erkannt
    Frigate->>MQTT: publish frigate/reviews (type=end, objects=[bird])
    MQTT->>n8n: Trigger
    n8n->>Frigate: GET Snapshot (JPEG)
    n8n->>Ollama: POST /api/generate (Bild + Prompt, qwen2.5vl)
    Ollama-->>n8n: JSON (Art, lateinisch, Konfidenz)
    n8n->>DB: INSERT Historie-Eintrag
    n8n->>MQTT: publish home/vogelbad/erkennung
    MQTT->>HA: MQTT-Sensor aktualisiert
    HA->>HA: Automation -> Notification + Kiosk-Banner
```

## 1. Frigate (`examples/frigate-config.yml`)

Läuft als Docker-Container (`examples/frigate-compose.yml`). Wichtigster
Schritt nach Kamera-Diagnose (siehe [hardware.md](hardware.md)):

```yaml
cameras:
  vogelbad:
    ffmpeg:
      inputs:
        - path: rtsp://<user>:<passwort>@<kamera-ip>:554/user=<user>&password=<passwort>&channel=1&stream=0.sdp
```

Danach `docker compose restart`. Web-UI unter `http://<frigate-host>:5000`.

## 2. n8n-Workflows

Zwei Workflows, in der n8n-UI über **Import from File** einzulesen:

- `examples/n8n_workflow_erkennung.json` — MQTT-Trigger, holt Snapshot,
  fragt Ollama, speichert Historie, publisht an MQTT.
- `examples/n8n_workflow_historie.json` — Webhook-Endpoint
  `/webhook/vogelbad-historie`, liefert die letzten 50 Erkennungen als JSON
  für Home Assistant.

Nach Import:

1. **MQTT-Credential** anlegen (Broker-Host/Port/User/Passwort des eigenen
   Mosquitto) und in beiden MQTT-Nodes zuweisen.
2. **Keine Datenbank nötig**: Die Historie wird als einfache JSON-Datei
   (`~/.n8n/vogelbad_historie.json`, Startinhalt `[]`) direkt per `fs` in den
   Code-Nodes gelesen/geschrieben — das n8n-nodes-base-Paket bringt in
   aktuellen Versionen keinen SQLite-Node mehr mit, und ein DB-Credential
   entfällt damit komplett. Datei einmalig anlegen und für den n8n-User
   beschreibbar machen.
3. Beide Workflows **aktivieren**.
4. **Ollama-Prompt anpassen**: Node "Ollama: qwen2.5vl Artbestimmung" im
   ersten Workflow → `jsonBody` → Feld `prompt`. Genau hier ist der Punkt, an
   dem sich Sprache, Detailgrad oder Sonderregeln (z.B. "auch Eichhörnchen
   erkennen") ändern lassen, ohne Code anzufassen.

### Zwei Stolpersteine bei aktuellen Frigate-/n8n-Versionen

- **Frigate 0.17+ hat `frigate/events` durch `frigate/reviews` ersetzt.**
  Statt eines flachen `{type, before, after}` mit `after.label`/`after.camera`
  gibt es jetzt ein verschachteltes Review-Objekt mit
  `after.data.objects` (Array von Labels wie `"bird"`) und
  `after.data.detections` (Array der eigentlichen Event-IDs, die für den
  Snapshot-Abruf gebraucht werden). Der Filter-Node im Workflow ist bereits
  auf dieses Schema angepasst.
- **Große Snapshots landen bei n8n im `filesystem-v2`-Binärmodus.** Der
  klassische Trick `$input.item.binary.data.data` liefert dann nur den
  String `"filesystem-v2"` statt echter Bilddaten (Ollama meldet dann
  `illegal base64 data at input byte 10`). Der zuverlässige Weg ist der
  eingebaute **"Convert to/from binary data"-Node** (`moveBinaryData`,
  Modus "Binary to JSON", Option "Data Is Base64"/"Keep As Base64") statt
  eines eigenen Code-Node-Hacks — der läuft im Hauptprozess und nicht im
  separaten JS-Task-Runner, der auf Binärdaten-Zugriff nicht zuverlässig
  reagiert.

## Privacy-Crop (Nachbargrundstück ausblenden)

Bei Weitwinkel-/Fisheye-Kameras wie dieser landet fast immer ungewollt
Nachbargrundstück im Bild. Getestet und wieder verworfen wurde die
**native Privacy-Mask der Kamera-Firmware** (Xiongmai-Feld
`AVEnc.VideoWidget.Covers`, per DVRIP setzbar, API bestätigt Erfolg) — im
echten Geräte-Config-Export steht aber `"AVEnc.Cover": null`, d.h. das
Feature existiert nur im geerbten Xiongmai-SDK-Schema, ist am Encoder
dieses Kameramodells aber nicht verdrahtet und hat keinerlei Wirkung.

Der funktionierende Weg ist ein **Crop in einem eigenen go2rtc-`exec`-Stream**,
der Frigate vorgeschaltet wird (nicht als Frigate-`output_args`, siehe
unten warum):

```yaml
go2rtc:
  streams:
    <kamera>_dvrip:
      - dvrip://user:pass@host:34567
    <kamera>_cropped:
      - exec:/usr/lib/ffmpeg/7.0/bin/ffmpeg -i rtsp://127.0.0.1:8554/<kamera>_dvrip -vf "crop=W:H:X:Y,format=yuv420p" -c:v libx264 -preset ultrafast -an -f rtsp {{output}}

cameras:
  <kamera>:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/<kamera>_cropped
          input_args: preset-rtsp-restream
          roles: [detect, record]
```

Zwei nicht offensichtliche Stolpersteine dabei:

1. **Reale Stream-Auflösung prüfen, nicht raten.** Kamera-Config-Felder wie
   `Camera.FishLensParam.ImageWidth/Height` beschreiben NICHT zwangsläufig
   die tatsächliche Hauptstream-Auflösung. Bei uns stand dort 1280x720,
   der echte Stream lief aber mit **2880x1616 ("5M")** — bestätigt per
   `ffmpeg -i rtsp://.../<kamera>_dvrip` (Zeile `Stream #0:0: Video: hevc
   ... 2880x1616`) und per Geräte-Config-Export (`AVEnc.Encode[0]
   .MainFormat[0].Video.Resolution: "5M"`). Ein Crop mit falscher
   Referenzauflösung schneidet nur einen kleinen, falschen Bildausschnitt
   heraus (bei uns: linke obere Ecke statt Bildmitte).
2. **`format=yuv420p` in die Filterkette hängen.** Manche Kamera-Streams
   liefern `yuvj420p` (Full-Range/JPEG-Farbraum). Ohne explizite
   Formatangabe im selben `-vf`-Ausdruck kommt es bei Crop+Neukodierung zu
   sichtbaren Farbfehlern (Lila-/Magenta-Stich, v.a. auf Grün/Vegetation).

**Warum nicht einfach `-vf crop=...` in Frigates eigenem `output_args`?**
Frigate hängt für die `detect`-Rolle IMMER zusätzlich einen eigenen
`-vf fps=..,scale=..` vor benutzerdefinierte `output_args` — zwei `-vf` im
selben ffmpeg-Aufruf kollidieren (nur der letzte gewinnt bzw. das Ergebnis
ist unvorhersehbar). Der Crop muss deshalb VOR Frigate passieren, nicht in
Frigates eigener ffmpeg-Pipeline.

**Exakten Crop-Bereich ermitteln**, wenn kein Lineal am Bildschirm liegt:
Referenzbild vom gewünschten Ausschnitt (z.B. Screenshot aus der
Hersteller-App) per Feature-Matching (OpenCV ORB + Homographie) gegen ein
Vollbild der Kamera abgleichen, statt Pixel zu schätzen — deutlich
präziser als Augenmaß, funktioniert auch wenn Referenzbild und Vollbild
unterschiedliche Auflösung/Seitenverhältnis/Kompression haben.

## 3. Home Assistant

- `examples/ha-package-vogelbad_kamera.yaml` → nach `packages/` kopieren.
  Enthält MQTT-Sensor, REST/Template-Sensor für die Historie, und die
  Notification-Automation.
- `examples/ha-dashboard-vogelbad.yaml` → nach `dashboards/` kopieren, dazu
  in `configuration.yaml` unter `lovelace: dashboards:` registrieren.
- `examples/ha-kiosk-banner-snippet.yaml` → als zusätzliche Karte in ein
  bestehendes Kiosk-Dashboard einfügen (zeigt neue Erkennung 90 Sekunden
  lang an, blendet danach automatisch aus).

Nach dem Einfügen: **Einstellungen → System → Neu laden → YAML-
Konfiguration neu laden** (Packages + Dashboards), kein voller Neustart
nötig.

## Test ohne Kamera

Die Pipeline lässt sich vor der Kamera-Inbetriebnahme durchtesten, indem man
per `mosquitto_pub` manuell eine Testnachricht auf `home/vogelbad/erkennung`
schickt:

```bash
mosquitto_pub -h <mqtt-broker-ip> -u <user> -P <passwort> \
  -t home/vogelbad/erkennung \
  -m '{"umgangssprachlich":"Blaumeise","lateinisch":"Cyanistes caeruleus","konfidenz":0.93,"anmerkung":"Test","zeitstempel":"2026-07-17T08:00:00","bild_url":"https://example.invalid/test.jpg"}'
```

Damit lässt sich der Home-Assistant-Teil (Sensor, Dashboard, Kiosk-Banner,
Notification) unabhängig von Frigate/n8n/Kamera verifizieren.
