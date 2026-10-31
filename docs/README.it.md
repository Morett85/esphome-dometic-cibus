# ESPHome – Frigorifero Dometic su CI-BUS (serie 10 a compressore)

Leggere e comandare un **frigorifero Dometic a compressore della serie 10** (RC / RCL / RCS / RCD 10)
tramite il connettore **CI-BUS**, con un ESP32, ESPHome e un economico transceiver LIN –
tutto in locale, integrato in Home Assistant.

🇬🇧 [English](../README.md) · 🇩🇪 [Deutsch](README.de.md)

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

| | |
|---|---|
| ✅ Acceso / spento | ✅ Modalità Performance / Silent / Turbo |
| ✅ Livello 1–5 | ✅ Compressore in funzione |
| ✅ Ventola condensatore | ✅ Codice errore + testo (W04, W10, E33 …) |
| ✅ Alimentazione (AC / DC12 / DC24) | ✅ Le modifiche fatte sul pannello del frigo vengono rilevate |

Provato con un **Dometic RCL 10.4EP**, un modulo LIN **FST T151** e un ESP32 (ESP-IDF),
ESPHome 2026.10.

> La temperatura interna del frigo **non** fa parte del frame di stato CI-BUS
> (lo hanno constatato anche altri progetti). Se serve, usare un sensore separato.

---

## Hardware

- ESP32 (classico, per esempio ESP32-DevKit)
- Modulo transceiver LIN che possa funzionare da **master LIN** (per esempio FST T151, schede con TJA1021)
- 12 V per il transceiver, massa in comune con ESP32 e frigo

### Collegamenti

| ESP32 | Transceiver | Frigo |
|---|---|---|
| GPIO27 (UART TX) | ingresso TXD | |
| GPIO26 (UART RX) | uscita RXD | |
| GND | GND | GND |
| | 12 V / VBAT | |
| | LIN | CI-BUS |
| | SLP / EN → 3,3 V (se presente) | |

⚠️ **Verifica TX/RX sul tuo modulo.** Sul nostro FST T151 il TX dell'ESP32 andava sul morsetto
che pensavamo fosse RX. Sintomo di linee invertite: non torna nemmeno l'eco della nostra richiesta.
Soluzione: scambiare `lin_tx_gpio` e `lin_rx_gpio` nel YAML.

## Installazione

1. Copia `esphome/dometic-fridge.yaml` e `esphome/secrets.yaml.example` nella cartella di ESPHome.
2. Rinomina `secrets.yaml.example` in `secrets.yaml` e compila i dati.
3. Se serve, modifica `lin_tx_gpio` / `lin_rx_gpio`.
4. Carica il firmware: `esphome run dometic-fridge.yaml`
5. Aggiungi il dispositivo in Home Assistant; card opzionale: `homeassistant/card.yaml`.

Secondo la documentazione Pekaway, nel menu del frigo l'**opzione CI deve essere attiva**.

**Sicurezza:** l'ESP32 non invia mai un comando di sua iniziativa. Finché non usi un controllo
si limita a leggere lo stato. Dal primo comando in poi invia ogni secondo un frame di controllo
con lo stato attuale (heartbeat), quindi le modifiche fatte sul frigo vengono mantenute.

---

## Protocollo

Il CI-BUS è LIN 2.x: 19200 baud, 8N1, **checksum enhanced** (somma di PID + byte dati con riporto,
invertita). L'ESP32 è il **master**, il frigo è slave.

### Stato – LIN ID `0x0C` (PID `0x4C`), inviato dal frigo

| Byte | Bit | Significato |
|---|---|---|
| D0 | 0–1 | alimentazione: 0 = AC, 1 = DC24, 2 = DC12 |
| D0 | 3–4 | modalità: 0 = Performance, 2 = Silent, 3 = Turbo |
| D0 | 5 | sempre 1 nelle nostre letture |
| D0 | 6 | compressore in funzione |
| D1 | – | livello 1–5 |
| D2 | 1 | ventola condensatore in funzione |
| D3 | 2 / 3 | ventola 1 / ventola 2 presente |
| D6 | 0–6 | codice errore (0 = nessuno, 4 = W04, 10 = W10, 33 = E33 …) |
| D7 | 0–1 | pagina |
| D7 | 7 | frigo acceso |

Esempi dal nostro frigo (acceso, DC12, Performance, livello 5):

```
22 05 00 01 00 00 80 80   compressore spento
62 05 02 01 00 00 80 80   compressore + ventola condensatore accesi
```

### Comando – LIN ID `0x0B` (PID `0x8B`), inviato dal master

| Byte | Valore |
|---|---|
| D0 | `0x20` \| modalità << 3 \| bit alimentazione (copiati dallo stato D0 bit 0–1) |
| D1 | livello 1–5 |
| D2 | `0x80` |
| D3–D6 | `0x00` |
| D7 | acceso << 7 \| **SYNC** << 3 (bit 4 = c_mode, bit 2 = lock – non usati) |

Una modifica si applica con un **impulso SYNC**: si invia il frame con D7 bit 3 a 1, si aspettano
~40 ms, si invia lo stesso frame senza SYNC, poi si legge `0x0C` per conferma. Se serve si ripete.
Tra un comando e l'altro il master invia il frame con i valori letti dal frigo (heartbeat).

Esempio – livello 4, Performance, acceso, DC12: `22 04 80 00 00 00 00 88` (SYNC), poi `… 80`.

### Codici di errore

`1` W01 NTC1 sbrinamento · `2` W02 NTC2 aria · `3` E03 display ↔ modulo di potenza ·
`4` W04 nessun master CI · `10` W10 porta aperta > 2 min · `11` W11 tensione batteria ·
`14` W14 ventola evaporatore · `17/18` resistenza sbrinamento · `19` W19 batteria bassa ·
`26` W26 sensore ambiente · `32` E32 ventola condensatore · `33` E33 il compressore non parte ·
`34` E34 compressore in sovraccarico · `35` E35 sovratemperatura del controller

### BREAK con la UART dell'ESP32

Il YAML invia un byte `0x00` a 9600 baud (≈ 18 tempi di bit a livello basso a 19200 baud), torna
a 19200 baud e invia `0x55` + PID. Non servono hardware aggiuntivo né librerie LIN.

## Particolarità nota

Nel nostro impianto, subito dopo un avvio a freddo dell'ESP32 il frigo non rispondeva, anche se
l'eco della nostra richiesta tornava corretta. Riapplicare i pin della UART a ESP già avviato lo
risolveva ogni volta. Per questo il YAML riapplica i pin da solo dopo 5 richieste senza risposta
(al massimo ogni 30 s). Riga di log: `No answer for … requests: re-applying UART pins`.

---

## Ringraziamenti

- **Pekaway – VAN_PI** ([GitHub](https://github.com/Pekaway/VAN_PI), Apache-2.0): la struttura di
  stato/comando e la sequenza SYNC sono state ricavate dal loro bridge CI2MQTT `DometicRC10` e dalla
  relativa documentazione.
- Foglio della community **CI-BUS LIN decoder** – basi del CI-BUS e frame dei frigo ad assorbimento.
- **YpsilonTM/esphome-caravan-fridge-cibus** – frame di stato del RML 10.4T ad assorbimento.

## Nota sui marchi

Dometic è un marchio del rispettivo titolare. Questo è un progetto indipendente della
community, non affiliato, sostenuto né approvato da Dometic. Il nome è usato solo per
descrivere la compatibilità.

## Esclusione di responsabilità

Uso a proprio rischio. Il progetto è fornito "così com'è", senza alcuna garanzia. Gli autori non
rispondono di danni ad apparecchi, veicoli, alimenti o altro. Comandare il frigo tramite il bus
potrebbe avere effetti sulla garanzia del produttore.

## Supporto

Se questo progetto ti ha fatto risparmiare tempo, puoi offrirmi un caffè ☕

[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-PayPal-0070ba?logo=paypal&logoColor=white)](https://paypal.me/moretto85)

## Licenza

[MIT](../LICENSE)
