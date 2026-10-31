# ESPHome – Dometic CI-BUS Kühlschrank (10er-Serie Kompressor)

Einen **Dometic Kompressorkühlschrank der 10er-Serie** (RC / RCL / RCS / RCD 10) über den
**CI-BUS**-Anschluss mit einem ESP32, ESPHome und einem günstigen LIN-Transceiver auslesen
und steuern – komplett lokal, eingebunden in Home Assistant.

🇬🇧 [English](../README.md) · 🇮🇹 [Italiano](README.it.md)

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

| | |
|---|---|
| ✅ Ein / Aus | ✅ Modus Performance / Silent / Turbo |
| ✅ Kühlstufe 1–5 | ✅ Kompressor läuft |
| ✅ Kondensatorlüfter läuft | ✅ Fehlercode + Text (W04, W10, E33 …) |
| ✅ Stromquelle (AC / DC12 / DC24) | ✅ Änderungen am Bedienfeld werden übernommen |

Getestet mit einem **Dometic RCL 10.4EP**, einem **FST T151** LIN-Modul und einem ESP32
(ESP-IDF), ESPHome 2026.10.

> Die Innentemperatur des Kühlschranks ist **nicht** im CI-BUS-Statusframe enthalten
> (das haben auch andere Projekte festgestellt). Dafür einen separaten Sensor verwenden.

---

## Hardware

- ESP32 (klassisch, z. B. ESP32-DevKit)
- LIN-Transceiver-Modul, das als **LIN-Master** arbeiten kann (z. B. FST T151, Boards mit TJA1021)
- 12 V für den Transceiver, gemeinsame Masse mit ESP32 und Kühlschrank

### Verkabelung

| ESP32 | Transceiver | Kühlschrank |
|---|---|---|
| GPIO27 (UART TX) | TXD-Eingang | |
| GPIO26 (UART RX) | RXD-Ausgang | |
| GND | GND | GND |
| | 12 V / VBAT | |
| | LIN | CI-BUS |
| | SLP / EN → 3,3 V (falls vorhanden) | |

⚠️ **TX/RX am Modul prüfen.** Bei unserem FST T151 musste TX des ESP32 an die Klemme,
die wir für RX gehalten hatten. Merkmal vertauschter Leitungen: Es kommt nicht einmal das
Echo der eigenen Anfrage zurück. Lösung: `lin_tx_gpio` und `lin_rx_gpio` im YAML tauschen.

## Installation

1. `esphome/dometic-fridge.yaml` und `esphome/secrets.yaml.example` in den ESPHome-Ordner kopieren.
2. `secrets.yaml.example` in `secrets.yaml` umbenennen und ausfüllen.
3. Bei Bedarf `lin_tx_gpio` / `lin_rx_gpio` anpassen.
4. Flashen: `esphome run dometic-fridge.yaml`
5. Gerät in Home Assistant hinzufügen; optionale Dashboard-Karte: `homeassistant/card.yaml`.

Laut Pekaway-Dokumentation muss im Menü des Kühlschranks die **CI-Option aktiviert** sein.

**Sicherheit:** Der ESP32 sendet nie von sich aus einen Befehl. Bis du ein Bedienelement
benutzt, liest er nur den Status. Nach dem ersten Befehl sendet er jede Sekunde einen
Steuerframe mit dem aktuellen Zustand (Heartbeat) – Änderungen am Kühlschrank bleiben erhalten.

---

## Protokoll

CI-BUS ist LIN 2.x: 19200 Baud, 8N1, **Enhanced Checksum** (Summe aus PID + Datenbytes mit
Übertrag, invertiert). Der ESP32 ist **Master**, der Kühlschrank ist Slave.

### Status – LIN-ID `0x0C` (PID `0x4C`), sendet der Kühlschrank

| Byte | Bits | Bedeutung |
|---|---|---|
| D0 | 0–1 | Stromquelle: 0 = AC, 1 = DC24, 2 = DC12 |
| D0 | 3–4 | Modus: 0 = Performance, 2 = Silent, 3 = Turbo |
| D0 | 5 | in unseren Aufzeichnungen immer 1 |
| D0 | 6 | Kompressor läuft |
| D1 | – | Kühlstufe 1–5 |
| D2 | 1 | Kondensatorlüfter läuft |
| D3 | 2 / 3 | Lüfter 1 / Lüfter 2 vorhanden |
| D6 | 0–6 | Fehlercode (0 = keiner, 4 = W04, 10 = W10, 33 = E33 …) |
| D7 | 0–1 | Seite (page) |
| D7 | 7 | Kühlschrank eingeschaltet |

Beispiele (ein, DC12, Performance, Stufe 5):

```
22 05 00 01 00 00 80 80   Kompressor aus
62 05 02 01 00 00 80 80   Kompressor + Kondensatorlüfter an
```

### Steuerung – LIN-ID `0x0B` (PID `0x8B`), sendet der Master

| Byte | Wert |
|---|---|
| D0 | `0x20` \| Modus << 3 \| Stromquellen-Bits (aus Status D0 Bit 0–1 übernommen) |
| D1 | Stufe 1–5 |
| D2 | `0x80` |
| D3–D6 | `0x00` |
| D7 | Ein << 7 \| **SYNC** << 3 (Bit 4 = c_mode, Bit 2 = lock – nicht verwendet) |

Eine Änderung wird mit einem **SYNC-Impuls** übernommen: Frame mit gesetztem D7 Bit 3 senden,
~40 ms warten, denselben Frame ohne SYNC senden, danach `0x0C` lesen und prüfen. Bei Bedarf
wiederholen. Zwischen den Befehlen sendet der Master den Frame mit den vom Kühlschrank
gelesenen Werten (Heartbeat).

Beispiel – Stufe 4, Performance, ein, DC12: `22 04 80 00 00 00 00 88` (SYNC), danach `… 80`.

### Fehlercodes

`1` W01 Abtau-NTC1 · `2` W02 Luft-NTC2 · `3` E03 Display ↔ Leistungsmodul ·
`4` W04 kein CI-Master · `10` W10 Tür > 2 min offen · `11` W11 Batteriespannung ·
`14` W14 Verdampferlüfter · `17/18` Abtauheizung · `19` W19 Batterie niedrig ·
`26` W26 Umgebungssensor · `32` E32 Kondensatorlüfter · `33` E33 Kompressor startet nicht ·
`34` E34 Kompressor überlastet · `35` E35 Übertemperatur Steuerung

### BREAK mit dem ESP32-UART

Das YAML sendet ein `0x00`-Byte mit 9600 Baud (≈ 18 Bitzeiten Low bei 19200 Baud), schaltet
zurück auf 19200 Baud und sendet `0x55` + PID. Keine zusätzliche Hardware oder LIN-Bibliothek nötig.

## Bekannte Eigenheit

Bei uns antwortete der Kühlschrank direkt nach einem Kaltstart des ESP32 nicht, obwohl das
Echo der eigenen Anfrage korrekt zurückkam. Ein erneutes Setzen der UART-Pins zur Laufzeit hat
es jedes Mal behoben. Das YAML setzt die Pins daher nach 5 unbeantworteten Anfragen
automatisch neu (höchstens alle 30 s). Logzeile: `No answer for … requests: re-applying UART pins`.

---

## Danksagung

- **Pekaway – VAN_PI** ([GitHub](https://github.com/Pekaway/VAN_PI), Apache-2.0): Status-/Steuerlayout
  und SYNC-Ablauf wurden aus ihrer `DometicRC10` CI2MQTT-Bridge und deren Dokumentation abgeleitet.
- Community-Tabelle **CI-BUS LIN Decoder** – CI-BUS-Grundlagen und Absorber-Frames.
- **YpsilonTM/esphome-caravan-fridge-cibus** – Statusframe des Absorbers RML 10.4T.

## Markenhinweis

Dometic ist eine Marke des jeweiligen Inhabers. Dies ist ein unabhängiges
Community-Projekt, nicht verbunden mit, unterstützt oder genehmigt von Dometic.
Der Name dient nur zur Beschreibung der Kompatibilität.

## Haftungsausschluss

Nutzung auf eigene Gefahr. Bereitgestellt „wie besehen“, ohne jegliche Gewährleistung.
Die Autoren haften nicht für Schäden an Geräten, Fahrzeugen, Lebensmitteln oder Sonstigem.
Die Steuerung über den Bus kann die Herstellergarantie beeinflussen.

## Unterstützung

Wenn dir das Projekt Zeit gespart hat, kannst du mir einen Kaffee spendieren ☕

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

## Lizenz

[MIT](../LICENSE)
