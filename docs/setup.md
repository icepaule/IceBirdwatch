# Setup — Schritt für Schritt

Diese Anleitung deckt die komplette Kette ab: Kamera → Frigate → n8n/Ollama
→ Home Assistant. Für die KI-Erkennung im Detail siehe
[ki-erkennung.md](ki-erkennung.md), für die Home-Assistant-Seite
[homeassistant.md](homeassistant.md) (mit Screenshots).

## Ablauf einer Erkennung

```mermaid
sequenceDiagram
    participant Cam as WAVEEME Kamera
    participant Frigate
    participant MQTT as Mosquitto
    participant n8n
    participant Ollama
    participant JSON as vogelbad_historie.json
    participant HA as Home Assistant

    Cam->>Frigate: Video-Stream (DVRIP/RTSP)
    Frigate->>Frigate: Bewegung + Objekt "bird" erkannt
    Frigate->>MQTT: publish frigate/reviews (type=end, objects=[bird])
    MQTT->>n8n: Trigger
    n8n->>Frigate: GET Snapshot (JPEG)
    n8n->>Ollama: POST /api/generate (Bild + Prompt, qwen2.5vl)
    Ollama-->>n8n: JSON (Art, lateinisch, Konfidenz)
    n8n->>JSON: neuer Eintrag anhängen
    n8n->>MQTT: publish home/vogelbad/erkennung
    MQTT->>HA: MQTT-Sensor aktualisiert
    HA->>HA: Automation -> Notification + Kiosk-Banner
```

## Schritt 1: Kamera-Diagnose

Siehe [hardware.md](hardware.md) für das vollständige Vorgehen (nmap-Scan,
Protokoll-Erkennung). Ergebnis ist entweder eine fertige RTSP-URL oder —
bei Xiongmai/DVRIP-Kameras wie dieser — IP, Port (meist `34567`), Nutzer
und Passwort für das DVRIP-Protokoll.

## Schritt 2: Netzwerk-Erreichbarkeit sicherstellen

Läuft die Kamera in einem eigenen IoT-VLAN mit Isolation, hat der
Frigate-Host darauf womöglich **gar keinen** Zugriff (nicht nur einzelne
Ports) — selbst wenn andere Dienste auf demselben Host andere Netze
problemlos erreichen. Kurzer Test von beiden Seiten:

```bash
# Vom Frigate-Host aus:
timeout 3 bash -c 'echo > /dev/tcp/<kamera-ip>/34567' && echo OK || echo FAIL
```

Schlägt das fehl, obwohl die Kamera nachweislich läuft: einen Host finden,
der Zugriff auf beide Netze hat (z.B. weil er selbst eine IP im
IoT-Segment hat), und dort einen simplen TCP-Relay aufsetzen:

```bash
# systemd-Service, z.B. /etc/systemd/system/kamera-relay.service
[Unit]
Description=TCP-Relay zur IoT-VLAN-isolierten Kamera
After=network-online.target

[Service]
ExecStart=/usr/bin/socat -d TCP-LISTEN:34567,fork,reuseaddr TCP:<kamera-ip>:34567
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Frigate zeigt danach auf `<relay-host>:34567` statt direkt auf die
Kamera-IP.

## Schritt 3: Frigate aufsetzen

`examples/frigate-compose.yml` und `examples/frigate-config.yml` als
Ausgangspunkt nehmen, Platzhalter ersetzen. Läuft als eigenständiger
Docker-Container:

```bash
docker compose up -d
```

Web-UI unter `http://<frigate-host>:5000`, ohne Login (sofern nicht extra
abgesichert):

![Frigate Live-Ansicht](images/frigate-live.png)

Bei DVRIP-Kameras übernimmt **go2rtc** (in Frigate eingebaut) das
Protokoll nativ — keine RTSP-URL-Bastelei nötig:

```yaml
go2rtc:
  streams:
    vogelbad_dvrip:
      - dvrip://<user>:<passwort>@<kamera-oder-relay-ip>:34567

cameras:
  vogelbad:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/vogelbad_dvrip
          input_args: preset-rtsp-restream
          roles: [detect, record]
```

Nach Config-Änderungen: `docker compose restart`.

## Schritt 4 (optional): Privacy-Crop

Falls die Kamera Nachbargrundstück mit im Bild hat: siehe eigener
Abschnitt weiter unten.

## Schritt 5: n8n-Workflows

Ausführlich in [ki-erkennung.md](ki-erkennung.md) beschrieben — hier nur
die Einrichtungsschritte:

1. Beide Workflow-Dateien in der n8n-UI über **Import from File** einlesen:
   `examples/n8n_workflow_erkennung.json` und
   `examples/n8n_workflow_historie.json`.
2. **MQTT-Credential** anlegen (Broker-Host/Port/User/Passwort) und in
   beiden MQTT-Nodes zuweisen.
3. **Historie-Datei anlegen**: `~/.n8n/vogelbad_historie.json` mit Inhalt
   `[]`, beschreibbar für den n8n-Prozess. Kein Datenbank-Credential nötig
   — aktuelle n8n-Versionen bringen keinen SQLite-Node mehr mit, die
   Historie wird direkt als JSON-Datei geführt.
4. Beide Workflows **aktivieren**.
5. **Ollama-Prompt anpassen** (optional): Node "Ollama: qwen2.5vl
   Artbestimmung" → `jsonBody` → Feld `prompt`.

### Zwei Stolpersteine bei aktuellen Frigate-/n8n-Versionen

- **Frigate 0.17+ hat `frigate/events` durch `frigate/reviews` ersetzt.**
  Statt eines flachen `{type, before, after}` mit `after.label`/`after.camera`
  gibt es jetzt ein verschachteltes Review-Objekt mit
  `after.data.objects` (Array von Labels wie `"bird"`) und
  `after.data.detections` (Array der eigentlichen Event-IDs, die für den
  Snapshot-Abruf gebraucht werden). Der Filter-Node im Workflow ist bereits
  auf dieses Schema angepasst. Wer eine andere Frigate-Version nutzt: erst
  per `mosquitto_sub -t 'frigate/#' -v` das tatsächliche Schema prüfen,
  nicht blind übernehmen.
- **Große Snapshots landen bei n8n im `filesystem-v2`-Binärmodus.** Der
  klassische Trick `$input.item.binary.data.data` liefert dann nur den
  String `"filesystem-v2"` statt echter Bilddaten (Ollama meldet dann
  `illegal base64 data at input byte 10`). Der zuverlässige Weg ist der
  eingebaute **"Convert to/from binary data"-Node** (`moveBinaryData`,
  Modus "Binary to JSON", Option "Data Is Base64"/"Keep As Base64") statt
  eines eigenen Code-Node-Hacks — der läuft im Hauptprozess und nicht im
  separaten JS-Task-Runner, der auf Binärdaten-Zugriff nicht zuverlässig
  reagiert.

## Schritt 6: Home Assistant

Siehe [homeassistant.md](homeassistant.md) — Dashboard, Kiosk-Banner,
Entities, mit Screenshots.

## Privacy-Crop (Nachbargrundstück ausblenden)

Bei Weitwinkel-/Fisheye-Kameras wie dieser landet fast immer ungewollt
Nachbargrundstück im Bild.

![Ergebnis nach dem Privacy-Crop](images/vogelbad-crop-ergebnis.jpg)

*Live-Bild nach dem Crop (Nachtsicht-Aufnahme) — das Nachbarhaus rechts ist
komplett aus dem Bild, nur noch die eigene Terrasse und der Garten sind zu
sehen.*
 Getestet und wieder verworfen wurde die
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

`exec:`-Quellen sind in go2rtc aus Sicherheitsgründen standardmäßig
deaktiviert — im Compose-File `GO2RTC_ALLOW_ARBITRARY_EXEC=true` als
Environment-Variable setzen (siehe `examples/frigate-compose.yml`).

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

## Test ohne Kamera

Der komplette Home-Assistant-Teil lässt sich unabhängig von
Frigate/n8n/Kamera durchtesten, indem man per `mosquitto_pub` manuell eine
Testnachricht auf `home/vogelbad/erkennung` schickt — Details dazu in
[homeassistant.md](homeassistant.md#test-ohne-kamera).
