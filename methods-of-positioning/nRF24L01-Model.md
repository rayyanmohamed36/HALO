# nRF24L01+ Presence Detection Model

Goal
------
Use small nRF24L01+ radio beacons and fixed anchor receivers to detect whether a person is inside a room. This assumes the person carries (or wears) a battery-powered nRF24L01+ transmitter that periodically broadcasts a short beacon.

Approach (high level)
----------------------
- Person carries a low-power transmitter that sends periodic beacons (ID + sequence).
- One or more fixed anchors (receivers) in the room listen for those beacons.
- Presence is inferred using simple heuristics:
  - Received Power Detector (RPD) boolean (available on many nRF24L01+ chips) — indicates carrier > ~-64 dBm.
  - Packet reception rate (PRR): fraction of beacons received over a short window.
  - Packet success and CRC counts; retransmit counts when using ACKs.
- Combine RPD and PRR with smoothing (moving average, hysteresis) to reduce false positives.

Hardware
--------
- nRF24L01+ modules (one per transmitter and each anchor receiver)
- Microcontrollers: small 3.3V boards (Arduino Pro Mini 3.3V, ESP32, or similar)
- 3.3V regulator and decoupling capacitor (10uF) across VCC/GND on module
- Level shifting or direct 3.3V MCU (do NOT power the nRF24L01+ from 5V)
- Optional: LiPo battery + boost/reg for portable beacon

Wiring (common)
-----------------
- nRF24L01+ ↔ Arduino (SPI)
  - GND -> GND
  - VCC -> 3.3V (use a regulator; add a 10uF cap)ßßßß
  - CE -> D9
  - CSN -> D10
  - SCK -> D13
  - MOSI -> D11
  - MISO -> D12
  - IRQ -> (optional)

Software / Libraries
---------------------
- Use the RF24 library by TMRh20 (Arduino/RPi): https://github.com/nRF24/RF24

Design notes and limitations
----------------------------
- nRF24L01+ does not provide RSSI as a numeric value. It has a Received Power Detector (RPD) register which is a boolean: true if received power > ~-64 dBm. Use this as a strong-adjacent indicator.
- For finer-grained inference, use PRR (packet reception rate) over windows (e.g., last 5–10 seconds).
- Multipath, reflections, and body attenuation affect signal — calibrate per-room and use multiple anchors for robustness.
- This method requires the person to carry an active radio beacon.

Algorithm (practical)
---------------------
1. Transmitter sends a beacon every 100–500 ms with a unique ID and sequence number.
2. Each anchor maintains a sliding window (e.g., last N beacons / last T seconds): counts received beacons and RPD hits.
3. Compute PRR = received / expected in window. If PRR > threshold or RPD is frequently true, declare presence.
4. Add hysteresis: require M consecutive windows above threshold to set present, and K consecutive windows below to clear.

Example Arduino sketches (minimal)
----------------------------------

Transmitter (beacon)
``` cpp
#include <SPI.h>
#include <RF24.h>

RF24 radio(9,10); // CE, CSN
const byte addr[6] = "BCAST";

struct Beacon { uint16_t id; uint32_t seq; } beacon;

void setup(){
  Serial.begin(115200);
  radio.begin();
  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_1MBPS);
  radio.openWritingPipe(addr);
  radio.stopListening();
  beacon.id = 1; beacon.seq = 0;
}

void loop(){
  beacon.seq++;
  radio.write(&beacon, sizeof(beacon));
  delay(200); // send 5x/sec
}
```

Receiver (anchor) — measure PRR and RPD
``` cpp
#include <SPI.h>
#include <RF24.h>

RF24 radio(9,10);
const byte addr[6] = "BCAST";

struct Beacon { uint16_t id; uint32_t seq; } beacon;

// sliding window state
const unsigned long WINDOW_MS = 5000;
unsigned long windowStart = 0;
int expectedInWindow = 25; // ~200ms beacons => ~25 in 5s
int receivedInWindow = 0;
int rpdHits = 0;

void setup(){
  Serial.begin(115200);
  radio.begin();
  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_1MBPS);
  radio.openReadingPipe(1, addr);
  radio.startListening();
  windowStart = millis();
}

void loop(){
  if(radio.available()){
    radio.read(&beacon, sizeof(beacon));
    receivedInWindow++;
    if(radio.testRPD()) rpdHits++;
  }

  unsigned long now = millis();
  if(now - windowStart >= WINDOW_MS){
    float prr = (float)receivedInWindow / expectedInWindow;
    float rpdRate = (float)rpdHits / max(1, receivedInWindow);

    bool present = (prr > 0.2) || (rpdRate > 0.3); // example thresholds — calibrate

    Serial.print("PRR="); Serial.print(prr,3);
    Serial.print(" RPD_rate="); Serial.print(rpdRate,3);
    Serial.print(" -> present="); Serial.println(present);

    // reset window
    receivedInWindow = 0; rpdHits = 0; windowStart = now;
  }
}
```

Calibration tips
----------------
- Measure with the transmitter at known distances (0.5m, 1m, 3m, doorway) and record PRR and RPD counts.
- Tune thresholds conservatively to avoid transient false positives (use hysteresis windows).
- If you need room-level accuracy without the person carrying a beacon, consider alternatives (BLE beacons, UWB, PIR sensors).

Privacy and safety
------------------
- This design requires a carried beacon. Respect privacy: inform occupants and secure beacon IDs.

Next steps
----------
- Integrate anchor nodes into the HALO system: publish presence events over MQTT/HTTP.
- Add multi-anchor voting to improve reliability and implement presence persistence/hysteresis parameters.

References
----------
- RF24 library: https://github.com/nRF24/RF24
- nRF24L01+ datasheet (RPD info)
