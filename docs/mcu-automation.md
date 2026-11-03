# Automating scanning with a microcontroller (experimental)

> **Experimental.** This feature has not been tested on real hardware yet. Connections may drop, timing may need tuning, and the protocol may change between versions.

ManaCore's Scan Card screen can be driven by an ESP or Arduino board (card feeder, robot arm, a
physical button). Open **Scan Card > Settings > Connect a microcontroller**.

| Board | Link | Pick in the app |
|---|---|---|
| ESP8266 / ESP32 | WiFi, TCP server on the board | ESP WiFi (IP + port) |
| Arduino + HC-05 / HC-06 | Bluetooth Classic serial (pair in Android settings first, PIN 1234 / 0000) | HC-05/06 |
| Arduino + HM-10 (or clone) | Bluetooth LE UART (service FFE0, char FFE1) | HM-10 |

## Protocol

Plain text, one message per line (`\n` terminated). Baud rate for HC-05/06/HM-10 is whatever the
module is set to (9600 default); the app does not care.

Board to app:

| Line | Meaning |
|---|---|
| `PING` | app answers `PONG` |
| `SCAN` or `SCAN:5000` | scan one card, give up after N ms (default 8000). Adds it even if "Auto-add" is off |
| `RETRY` or `RETRY:5000` | rescan the same card more loosely (reads more text lines, adds on the first read). Use after `TIMEOUT` / `REVIEW` |
| `PONG` | your answer to the app's heartbeat `PING` |
| `PAUSE` / `RESUME` | stop / allow the camera to add cards on its own |
| `UNDO` | remove the last scanned card (`UNDONE` or `NOTHING`) |
| `FLASH:1` / `FLASH:0` | torch on / off |
| `STATUS` | app answers `STATUS\|<cards this session>\|<paused 0/1>` |

App to board:

| Line | Meaning |
|---|---|
| `READY` | sent right after connecting |
| `ADDED\|name\|SET\|number\|foil` | a card was added (sent for every add) |
| `REVIEW\|name\|confidence` | `SCAN` timed out with only an uncertain match (it is in the review list) |
| `TIMEOUT` | `SCAN` timed out, nothing recognised |
| `REJECT` | sent right after `REVIEW` / `TIMEOUT`: route this card to the manual pile |
| `ERR\|ADD_FAILED` | the card was recognised but saving it failed |
| `PING` | heartbeat every 5 s. Reply `PONG`; once you have, 16 s of silence makes the app drop and reconnect |

A feeder loop is: `PAUSE` once, then for each card: feed it, send `SCAN`, wait for `ADDED` /
`REVIEW` / `TIMEOUT`, eject it, repeat. The Scan Card screen must be open on the phone.

## ESP32 (WiFi) sketch

```cpp
#include <WiFi.h>
WiFiServer server(8080);
WiFiClient client;

void setup() {
  Serial.begin(115200);
  WiFi.begin("YOUR_SSID", "YOUR_PASSWORD");
  while (WiFi.status() != WL_CONNECTED) delay(300);
  Serial.println(WiFi.localIP());           // type this IP into the app
  server.begin();
  client.setNoDelay(true);
}

void loop() {
  if (!client || !client.connected()) { client = server.available(); return; }
  if (client.available()) {
    String line = client.readStringUntil('\n');
    line.trim();
    Serial.println("app: " + line);          // READY, ADDED|..., TIMEOUT ...
    if (line.startsWith("ADDED")) { /* eject card, feed next */ }
  }
  if (digitalRead(0) == LOW) {               // e.g. BOOT button = "scan this card"
    client.print("SCAN\n");
    delay(500);
  }
}
```

## Arduino + HC-05 / HC-06 / HM-10 sketch

Wire the module's TX to pin 10 and RX to pin 11 (use a voltage divider on the Arduino-TX to
module-RX line: the module is 3.3 V).

```cpp
#include <SoftwareSerial.h>
SoftwareSerial bt(10, 11);   // RX, TX

void setup() { bt.begin(9600); Serial.begin(9600); }

void loop() {
  if (bt.available()) {
    String line = bt.readStringUntil('\n');
    line.trim();
    Serial.println(line);
    if (line.startsWith("ADDED") || line == "TIMEOUT") { /* advance the feeder */ }
  }
  if (digitalRead(2) == LOW) { bt.print("SCAN\n"); delay(500); }
}
```

## Notes

- Raw TCP is used for WiFi, so the board needs no HTTPS or certificate.
- BLE writes go out in 20-byte chunks, so long lines are fine.
- If the link drops the app retries 6 times with growing delays (1.5 s up to 15 s), then shows "reconnect failed"; reopen the dialog to retry by hand.
- While linked the screen stays on, orientation is locked, and a "Scanner board linked" notification keeps the link alive in the background.
