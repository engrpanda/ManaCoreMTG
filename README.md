<p align="center">
  <img src="docs/app_logo.png" alt="ManaCore logo" width="160" />
</p>

<h1 align="center">ManaCore</h1>

<p align="center">
  <strong>MTG Tester &amp; Simulator — see how your deck flows.</strong><br />
  A Magic: The Gathering companion for Commander / EDH players, built natively for Android.
</p>

<p align="center">
  <img alt="Platform" src="https://img.shields.io/badge/platform-Android%208.0%2B-3DDC84" />
  <img alt="Version" src="https://img.shields.io/badge/version-0.16.1-blue" />
</p>

---

## Overview

ManaCore helps Magic: The Gathering players see how consistently a deck actually performs *before* it hits the table. Build or import a decklist, then run it through:

**Mulligan → Opening Hand → Draw Flow → Turn-by-Turn Analysis → Repeated Simulation**

Hundreds of simulated games run in seconds and are calculated locally on your device, giving you real odds of drawing lands, ramp, tutors, interaction and combo pieces on curve. No account, no server and no AI are required.

## Features

### Deck building and collection
- **Decks, Collection and Wishlist** with folders, tags, notes and custom deck cover art.
- **Commander support**, including partner commanders and grouping by type, color, subtype or mana value.
- **Import** from a deck link or a pasted decklist (a tolerant parser handles many export formats and detects the commander).
- **Card scanning:** camera + on-device OCR to add cards by scanning them, with duplicate protection, two-frame confirmation, exact-printing detection from the set code and collector number, an offline name list, a review list for uncertain scans and session history. Microcontroller control is experimental. See [Automated scanning](#automated-scanning-with-a-microcontroller-experimental) to drive it from an ESP or Arduino board.
- **Export** a deck as plain text, CSV, or a shareable **visual deck image** (with owner name, deck price and a preview before sharing).
- **Live prices** from Scryfall, shown in your choice of 30+ currencies, with per-deck totals and value by card type.
- **Backups:** save and restore your data (Automatic Backup with PRO).

### Testing your deck
- **Simulator:** repeated game simulations with land, ramp and color-source statistics.
- **Mulligan and Opening Hand** testing.
- **Playtest table:** goldfish a game with a full battlefield, library and graveyard tools.
- **Deck stats:** mana curve, color pie and type breakdown.

### Discovery
- **Search** powered by Scryfall: full-text search, card details, rulings, printings and legality.
- **⚡ Synergy:** deterministic, on-device suggestions of cards that work well together, plus **🧩 Known Combos** from Commander Spellbook.
- **Home feed:** card art, new spoilers and Magic news from public RSS feeds.

### Life Counter
- Multiplayer table (up to 10 players) with landscape tiles, player profiles with decks and commander scanning.
- Commander damage, custom counters, monarch and initiative, dice and coin, "who goes first", undo/redo and match history.
- Optional turn tracking; long-press the turn counter to jump to any turn and player.

### Appearance
- **System, Light and Dark** themes.
- **✦ Mana** *(PRO)*: an animated theme with a drifting five-color mana aurora, glass surfaces, gradient buttons and rainbow-edged cards.

## Automated scanning with a microcontroller *(experimental)*

> **Experimental.** Microcontroller control is a new feature that has not been tested on real hardware yet. Expect rough edges: connections may drop, timing may need tuning, and commands or the protocol may change between versions. The example sketches are starting points, not finished designs. Please report problems by email (see Support).

Scan Card can be driven by a microcontroller, for example a card feeder that places one card at a
time under the phone's camera. The phone does the hard work (camera, text recognition, card lookup);
the board handles the motors, sensors and buttons. They talk by sending short text lines.

| Board | Link | Choose in the app |
|---|---|---|
| ESP8266 / ESP32 | WiFi (the board runs a TCP server) | **ESP WiFi** |
| Arduino + HC-05 / HC-06 | Bluetooth Classic serial | **HC-05/06** |
| Arduino + HM-10 (or clone) | Bluetooth Low Energy | **HM-10** |

### Quick start

1. Upload one of the example sketches below to your board.
2. Open **Scan Card → Settings (gear) → Connect a microcontroller**.
3. Pick the matching tab and connect:
   - **ESP WiFi:** phone and board on the same network. Open the board's serial monitor, read the IP address it prints and enter it with port `8080`.
   - **HC-05/06:** pair it first in Android Bluetooth settings (PIN `1234` or `0000`), then choose it in the list.
   - **HM-10:** power the module and tap **Scan for devices**, then choose it.
4. The scan screen shows a **Board linked** badge. The screen stays on and rotation is locked while linked, and a "Scanner board linked" notification keeps the link alive in the background.
5. Keep the **Scan Card** screen open on the phone. Commands are only handled while it is showing.

If the link drops, the app retries up to 6 times, then asks you to reconnect from the same dialog.

### The protocol

Plain text, one message per line, ending with `\n`. Baud rate is whatever your Bluetooth module is set to (9600 is the usual default).

Board to app:

| Line | Meaning |
|---|---|
| `SCAN` or `SCAN:5000` | Scan one card and give up after the given milliseconds (default 8000). Adds the card even if Auto-add is off |
| `RETRY` or `RETRY:5000` | Rescan the same card more loosely (reads more text lines, adds on the first read). Use after `TIMEOUT` or `REVIEW` |
| `PAUSE` / `RESUME` | Stop / allow the camera to add cards on its own |
| `UNDO` | Remove the last scanned card (app replies `UNDONE` or `NOTHING`) |
| `FLASH:1` / `FLASH:0` | Torch on / off |
| `STATUS` | App replies `STATUS\|<cards this session>\|<paused 0/1>` |
| `PING` / `PONG` | Either side can ping. The app pings every 5 s; once your board has answered `PONG`, 16 s of silence makes the app drop and reconnect |

App to board:

| Line | Meaning |
|---|---|
| `READY` | Sent right after connecting |
| `ADDED\|name\|SET\|number\|foil` | A card was added (foil is `1` or `0`) |
| `REVIEW\|name\|confidence` | `SCAN` timed out with only an uncertain match; it is in the app's review list |
| `TIMEOUT` | `SCAN` timed out and nothing was recognised |
| `REJECT` | Sent right after `REVIEW` or `TIMEOUT`: send this card to your manual pile |
| `ERR\|ADD_FAILED` | The card was recognised but saving it failed |

A feeder loop is: send `PAUSE` once, then for each card feed it, send `SCAN`, wait for `ADDED` (eject to the done pile) or `REJECT` (eject to the manual pile), and repeat.

### Example: Arduino Uno + HC-05/06 + servo feeder

Wiring: HC-05 `TX` to pin 10, HC-05 `RX` to pin 11 through a voltage divider (1 kΩ from pin 11, 2 kΩ to GND, the module is 3.3 V), servo signal to pin 9, push button between pin 2 and GND. Tune the servo angles to your mechanism.

```cpp
#include <SoftwareSerial.h>
#include <Servo.h>

SoftwareSerial bt(10, 11);          // RX, TX
Servo feeder;

const int BUTTON_PIN = 2;
const int SERVO_PIN = 9;
const int ANGLE_REST = 60;          // waiting for the next card
const int ANGLE_FEED = 120;         // pushes one card under the camera
const int ANGLE_REJECT = 30;        // pushes the card to the manual pile
const int STACK_SIZE = 50;          // stop after this many cards

bool running = false;
int fed = 0;

void feedCard() {
  feeder.write(ANGLE_FEED);   delay(500);
  feeder.write(ANGLE_REST);   delay(700);   // let the card settle and the camera focus
  fed++;
}

void scanNext() {
  if (fed >= STACK_SIZE) { running = false; return; }
  feedCard();
  bt.print("SCAN:8000\n");
}

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  feeder.attach(SERVO_PIN);
  feeder.write(ANGLE_REST);
  bt.begin(9600);
  Serial.begin(9600);
}

void handleLine(String line) {
  line.trim();
  Serial.println(line);
  if (line == "PING")              bt.print("PONG\n");
  else if (line.startsWith("ADDED")) { if (running) scanNext(); }
  else if (line == "REJECT") {
    feeder.write(ANGLE_REJECT); delay(500); feeder.write(ANGLE_REST); delay(500);
    if (running) scanNext();
  }
}

void loop() {
  if (bt.available()) handleLine(bt.readStringUntil('\n'));

  if (digitalRead(BUTTON_PIN) == LOW) {        // button starts / stops a run
    delay(250);
    running = !running;
    if (running) { fed = 0; bt.print("PAUSE\n"); scanNext(); }
    else bt.print("RESUME\n");
    while (digitalRead(BUTTON_PIN) == LOW) {}
  }
}
```

### Example: ESP32 over WiFi

```cpp
#include <WiFi.h>

WiFiServer server(8080);
WiFiClient client;

void setup() {
  Serial.begin(115200);
  pinMode(0, INPUT_PULLUP);                   // BOOT button = "scan this card"
  WiFi.begin("YOUR_SSID", "YOUR_PASSWORD");
  while (WiFi.status() != WL_CONNECTED) delay(300);
  Serial.println(WiFi.localIP());             // type this IP into the app, port 8080
  server.begin();
}

void loop() {
  if (!client || !client.connected()) { client = server.available(); return; }
  client.setNoDelay(true);

  if (client.available()) {
    String line = client.readStringUntil('\n');
    line.trim();
    Serial.println("app: " + line);
    if (line == "PING") client.print("PONG\n");
    if (line.startsWith("ADDED")) { /* eject the card, feed the next one */ }
    if (line == "REJECT")        { /* send the card to the manual pile */ }
  }

  if (digitalRead(0) == LOW) { client.print("SCAN\n"); delay(600); }
}
```

For an HM-10 use the Arduino sketch above unchanged: the module forwards serial data over BLE, and the app splits long lines into 20-byte pieces for you.

### Troubleshooting

- **Cannot connect over WiFi:** make sure the phone and board are on the same network (not a guest network), the IP is the one printed by the board, and the port matches the sketch.
- **HC-05/06 not in the list:** pair it in Android Bluetooth settings first. Only paired devices are listed.
- **HM-10 not found:** it must be powered and not already connected to another phone. Allow the Bluetooth permission when asked.
- **Garbled text:** the baud rate in your sketch must match the module's (`9600` by default).
- **A card keeps timing out:** improve lighting, turn on **Auto torch** in Scan Settings, or send `RETRY` before rejecting it.
- **Same card twice in a row is skipped:** `SCAN` already resets duplicate protection, so this only happens in the app's own continuous mode.

More detail is in [`docs/mcu-automation.md`](docs/mcu-automation.md).

## Download

**[⬇️ Download the latest APK](https://github.com/engrpanda/ManaCore/releases/latest)**

Requires Android 8.0 or newer. Allow "Install unknown apps" for your browser or file manager when prompted.

## Data sources and credits

ManaCore is a fan project and relies on the following services. It is not produced by or endorsed by any of them.

- **[Scryfall](https://scryfall.com)** — card data, images and prices
- **[Commander Spellbook](https://commanderspellbook.com)** — known combo data
- **[Frankfurter](https://frankfurter.dev)** — currency exchange rates
- Public RSS feeds of Draftsim, MTG Arena Zone, EDHREC, MTGGoldfish and Hipsters of the Coast for the news feed

## Privacy

ManaCore has no accounts, ads or analytics. Decks, collections, photos and settings stay on your device; the app only contacts the services above to fetch card data, prices, rates and news. The full policy is available in-app under **Settings → About → Privacy Policy**.

## Legal

ManaCore is unofficial Fan Content permitted under the Fan Content Policy. Not approved/endorsed by Wizards. Portions of the materials used are property of Wizards of the Coast. ©Wizards of the Coast LLC.

Magic: The Gathering and all related names, symbols and artwork are property of Wizards of the Coast. Simulation results are statistical estimates and prices are for reference only. See **Settings → About → Disclaimer** for the full text.

## Support

- Email: [support.engrpanda@gmail.com](mailto:support.engrpanda@gmail.com)
- Google Play: [EngrPanda on Google Play](https://play.google.com/store/apps/developer?id=engrpandaece)

## License

© EngrPanda. All rights reserved.
