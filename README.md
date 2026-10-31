# ESPHome – Dometic CI-BUS fridge (10-series compressor)

Read and control a **Dometic 10-series compressor fridge** (RC / RCL / RCS / RCD 10)
over its **CI-BUS** connector with an ESP32, ESPHome and a cheap LIN transceiver –
fully local, integrated into Home Assistant.

🇩🇪 [Deutsche Anleitung](docs/README.de.md) · 🇮🇹 [Guida in italiano](docs/README.it.md)

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

| | |
|---|---|
| ✅ Power on / off | ✅ Mode Performance / Silent / Turbo |
| ✅ Cooling level 1–5 | ✅ Compressor running |
| ✅ Condenser fan running | ✅ Error code + text (W04, W10, E33 …) |
| ✅ Power source (AC / DC12 / DC24) | ✅ Changes made on the fridge panel are picked up |

Tested on a **Dometic RCL 10.4EP** with an **FST T151** LIN module and an ESP32 (ESP-IDF),
ESPHome 2026.10.

> The internal fridge temperature is **not** part of the CI-BUS status frame
> (the same was found by other projects). Use a separate sensor if you need it.

---

## Hardware

- ESP32 (classic, e.g. ESP32-DevKit)
- LIN transceiver module that can act as **LIN master** (e.g. FST T151, TJA1021-based boards)
- 12 V supply for the transceiver, common GND with the ESP32 and the fridge

### Wiring

| ESP32 | Transceiver | Fridge |
|---|---|---|
| GPIO27 (UART TX) | TXD input | |
| GPIO26 (UART RX) | RXD output | |
| GND | GND | GND |
| | 12 V / VBAT | |
| | LIN | CI-BUS |
| | SLP / EN → 3.3 V (if the board has it) | |

⚠️ **Check TX/RX on your module.** On our FST T151 the ESP32 TX had to go to the
terminal we expected to be RX. Symptom of swapped lines: not even the echo of our own
request comes back. Fix: swap `lin_tx_gpio` and `lin_rx_gpio` in the YAML.

## Installation

1. Copy `esphome/dometic-fridge.yaml` and `esphome/secrets.yaml.example` into your ESPHome folder.
2. Rename `secrets.yaml.example` → `secrets.yaml` and fill in your values.
3. Adjust `lin_tx_gpio` / `lin_rx_gpio` if needed.
4. Flash: `esphome run dometic-fridge.yaml`
5. Add the device in Home Assistant; optional dashboard card: `homeassistant/card.yaml`.

According to Pekaway's documentation the **CI option has to be enabled in the fridge menu**.

**Safety:** the ESP32 never sends a command on its own. Until you use one of the
controls it only reads the status. After the first command it sends a control frame every
second that mirrors the current state (heartbeat), so manual changes on the fridge are kept.

---

## Protocol

CI-BUS is LIN 2.x: 19200 baud, 8N1, **enhanced checksum** (sum of PID + data bytes with
carry, inverted). The ESP32 is the **master**; the fridge is a slave.

### Status – LIN ID `0x0C` (PID `0x4C`), sent by the fridge

| Byte | Bits | Meaning |
|---|---|---|
| D0 | 0–1 | power source: 0 = AC, 1 = DC24, 2 = DC12 |
| D0 | 3–4 | mode: 0 = Performance, 2 = Silent, 3 = Turbo |
| D0 | 5 | always 1 in our captures |
| D0 | 6 | compressor running |
| D1 | – | cooling level 1–5 |
| D2 | 1 | condenser fan running |
| D3 | 2 / 3 | fan 1 / fan 2 available |
| D6 | 0–6 | error code (0 = none, 4 = W04, 10 = W10, 33 = E33 …) |
| D7 | 0–1 | page |
| D7 | 7 | fridge switched on |

Examples from our fridge (on, DC12, Performance, level 5):

```
22 05 00 01 00 00 80 80   compressor off
62 05 02 01 00 00 80 80   compressor + condenser fan on
```

### Control – LIN ID `0x0B` (PID `0x8B`), sent by the master

| Byte | Value |
|---|---|
| D0 | `0x20` \| mode << 3 \| power-source bits (copied from status D0 bits 0–1) |
| D1 | level 1–5 |
| D2 | `0x80` |
| D3–D6 | `0x00` |
| D7 | power << 7 \| **SYNC** << 3 (bit 4 = c_mode, bit 2 = lock – not used) |

A change is applied with a **SYNC pulse**: send the frame with D7 bit 3 set, wait ~40 ms,
send the same frame without SYNC, then read `0x0C` to confirm. Repeat a few times if the
status does not match yet. Between commands the master sends the frame with the values
read from the fridge (heartbeat).

Example – level 4, Performance, on, DC12: `22 04 80 00 00 00 00 88` (SYNC), then `… 80`.

### Error codes

`1` W01 defrost NTC1 · `2` W02 fridge air NTC2 · `3` E03 display ↔ power module ·
`4` W04 no CI master · `10` W10 door open > 2 min · `11` W11 battery voltage ·
`14` W14 evaporator fan · `17/18` defrost heater · `19` W19 battery low ·
`26` W26 ambient sensor · `32` E32 condenser fan · `33` E33 compressor does not start ·
`34` E34 compressor overload · `35` E35 controller over-temperature

### Sending a BREAK with the ESP32 UART

The YAML sends one `0x00` byte at 9600 baud (≈ 18 bit times low at 19200 baud), then
switches back to 19200 baud and sends `0x55` + PID. No extra hardware or LIN library needed.

## Known quirk

In our setup the fridge did not answer right after a cold boot of the ESP32, although
our own request was echoed correctly. Applying the UART pins again at runtime fixed it
every time. The YAML therefore re-applies the pins automatically after 5 unanswered
requests (at most every 30 s). Log line: `No answer for … requests: re-applying UART pins`.

---

## Credits

- **Pekaway – VAN_PI** ([GitHub](https://github.com/Pekaway/VAN_PI), Apache-2.0):
  the status/control layout and the SYNC sequence were derived from their
  `DometicRC10` CI2MQTT bridge and its documentation (`VanPi-Core-OS/data/ci2mqtt/DometicRC10.md`).
- Community **CI-BUS LIN decoder** spreadsheet – CI-BUS basics and absorber fridge frames.
- **YpsilonTM/esphome-caravan-fridge-cibus** – status frame of the absorber RML 10.4T.

## Trademark notice

Dometic is a trademark of its respective owner. This is an independent community
project, not affiliated with, endorsed by or supported by Dometic.
The name is used only to describe compatibility.

## Disclaimer

Use at your own risk. This project is provided "as is", without warranty of any kind.
The authors are not liable for any damage to devices, vehicles, food or anything else.
Controlling the fridge over the bus may affect the manufacturer's warranty.

## Support

If this project saved you some time, you can buy me a coffee ☕

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

## License

[MIT](LICENSE)
