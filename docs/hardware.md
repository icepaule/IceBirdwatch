# Hardware

## Produkt

[WAVEEME Vogelhaus mit Kamera](https://www.amazon.de/dp/B0GJS3ZBTQ) — ca. 67€,
3K-Auflösung, 160°-Weitwinkel, 3W-Solarpanel, 2,4GHz-WLAN mit externer
5dBi-Antenne, microSD-Slot (bis 128GB), integrierte Vogeltränke.

![Live-View in der Hersteller-App](docs/images/produkt-app.jpg)

Die App-Vorschau zeigt das Kernproblem: Artbestimmung läuft über die
Hersteller-Cloud. Eine echte Kundenrezension bestätigt, dass die KI-Erkennung
nur **1 Monat kostenlos** ist, danach vermutlich abopflichtig — Fotos landen
aber unabhängig davon auch lokal auf der microSD-Karte.

![Vogeltränke und Futterbereich](docs/images/produkt-lifestyle-1.jpg)

![Solarpanel und Montage](docs/images/produkt-lifestyle-2.jpg)

## Der Xiongmai-Verdacht

Die Kategorie "günstige Solar-Vogelkamera für 50-100€" wird von wenigen
chinesischen OEM-Plattformen beliefert. Dokumentiert ist mindestens ein
nahezu identischer Fall: Ein anderes No-Name-"AI Birdfeeder"-Produkt entpuppte
sich als 1:1 rebrandete **Xiongmai-IP-Kamera** (Goke-SoC), bei der die
gesamte "smarte" App nur ein Software-Layer über einer Standard-DVR-/IPC-
Firmware ist — inklusive funktionierendem lokalem RTSP-Zugriff auf Port 554
und dem proprietären XM-Protokoll auf Port 34567
([Quelle: ruse.tech Blogpost](https://www.ruse.tech/blogs/hacking-trisvision-xiongmai-iot-birdfeeder)).

**Erwartetes Standard-RTSP-URL-Muster** (Xiongmai/XMEye-Kameras):

```
rtsp://<user>:<passwort>@<kamera-ip>:554/user=<user>&password=<passwort>&channel=1&stream=0.sdp
```

Falls die Kamera stattdessen auf der anderen verbreiteten Chip-Familie
(GM8135/A9, "iLnkP2P"-Protokoll) basiert, gibt es dafür ebenfalls ein
funktionierendes Reverse-Engineering-Tool: [cam-reverse](https://github.com/DavidVentura/cam-reverse).

## Diagnose-Vorgehen (nach Inbetriebnahme)

1. Kamera per Hersteller-App ans WLAN koppeln.
2. IP-Adresse ermitteln (Router-/DHCP-Übersicht oder `nmap -sn <Subnetz>`).
3. Ports scannen:
   ```
   nmap -p 554,34567,8000,37777,80 <kamera-ip>
   ```
4. Falls Port 554 offen: RTSP-URL-Muster oben testen (z.B. mit VLC oder
   `ffprobe`).
5. Falls nur Port 34567 offen: `xiongmai-cam-api`-Python-Bibliothek
   probieren, ggf. mit Standard-Zugangsdaten aus dem Referenzfall
   (`admin` / auf dem Gerätelabel geprüfte oder auf dem XM-Protokoll übliche
   Defaults).
6. Falls beides fehlschlägt (rein Cloud-P2P, kein lokaler Zugriff): Fallback
   auf manuelles microSD-Karten-Auslesen — die Kamera speichert Fotos/Videos
   nachweislich unabhängig vom Cloud-Abo lokal auf die Karte.

Ergebnis in `frigate/config.yml` (`cameras.vogelbad.ffmpeg.inputs[0].path`)
eintragen.

## Warum keine WiFi-SD-Karte als Workaround

Kurz geprüft und verworfen: Die Kamera hat einen **microSD**-Slot. WiFi-SD-
Karten (z.B. EZShare) existieren nur im vollen SD-Kartenformat oder als
SD-große Adapterhüllen für microSD-Karten — beide passen nicht in einen
microSD-Slot. Echte microSD-native WiFi-Karten gibt es als Produktkategorie
praktisch nicht mehr am Markt (Stand 2026).
