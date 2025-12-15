# Ultrasonic Presence Detection Model

Goal
------
Detect whether someone is inside a room using ultrasonic distance sensors (e.g., HC-SR04 or MaxBotix). This approach is passive and local — sensors measure distance to the nearest object and infer presence from sustained close-range readings.

Approaches
---------
- Doorway-mounted single-sensor: place a sensor pointing across a doorway to detect crossing events or sustained presence near the doorway.
- Dual-sensor doorway counter: two sensors spaced along the doorway to detect direction and count people entering/leaving.
- Ceiling-mounted coverage: one or more sensors mounted on ceiling pointing down; infer occupancy when measured distance is below a threshold for a sustained window.
- Hybrid: combine ultrasonic with PIR or pressure mats to reduce false positives.

Hardware
--------
- HC-SR04: Trigger, Echo, Vcc (5V), GND. Echo output is 5V — use a voltage divider before feeding into 3.3V MCU pins.
- MaxBotix MB1000/MB1010: analog or serial output, often easier for 3.3V MCUs.
- Microcontroller: Arduino, ESP32, or other MCU with digital I/O and timers.

Wiring (HC-SR04 typical)
------------------------
- HC-SR04 -> Arduino/MCU
  - Vcc -> 5V (or sensor's required supply)
  - GND -> GND
  - Trig -> digital out (e.g., D7)
  - Echo -> digital in (e.g., D6) via voltage divider if MCU is 3.3V

Design notes and limitations
---------------------------
- Ultrasonic sensors measure distance to the nearest reflective surface; soft materials (curtains, clothing) absorb sound and reduce reliability.
- Temperature and humidity change speed of sound — long-term installations should compensate or recalibrate.
- Minimum/maximum ranges: HC-SR04 is typically ~2cm–400cm; performance degrades at edges.
- Blind spots exist — larger rooms need multiple sensors or ceiling placement.
- Good for privacy-preserving presence detection (no camera), but requires careful calibration to avoid false triggers.

Algorithm (practical)
---------------------
1. Sample distance at a fixed interval (e.g., 5–10 Hz).
2. Maintain a short-time window (e.g., last 3–5 seconds) and compute fraction of samples below a distance threshold.
3. Use hysteresis: require consecutive windows above threshold to declare "present" and consecutive windows below to clear.
4. For doorway counting, use two sensors and detect the temporal order of triggers to infer direction.

Example Arduino sketch — simple presence (HC-SR04)
--------------------------------------------------
``` cpp
#define TRIG_PIN 7
#define ECHO_PIN 6

unsigned long sampleInterval = 200; // ms (5 Hz)
const int WINDOW_SAMPLES = 15; // ~3s window at 200ms
int windowIdx = 0;
int belowThreshCount = 0;
int samples[WINDOW_SAMPLES];
const int DIST_THRESH_CM = 120; // distance indicating presence

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  for(int i=0;i<WINDOW_SAMPLES;i++) samples[i]=0;
}

long measureDistanceCm(){
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long dur = pulseIn(ECHO_PIN, HIGH, 30000UL); // timeout 30ms
  if(dur==0) return -1; // no echo
  long dist = (dur / 2) / 29.1; // cm
  return dist;
}

void loop(){
  long d = measureDistanceCm();
  int below = (d>0 && d < DIST_THRESH_CM) ? 1 : 0;

  // sliding buffer update
  belowThreshCount -= samples[windowIdx];
  samples[windowIdx] = below;
  belowThreshCount += samples[windowIdx];
  windowIdx = (windowIdx + 1) % WINDOW_SAMPLES;

  float fraction = (float)belowThreshCount / WINDOW_SAMPLES;
  static int state = 0; // 0=absent,1=present
  if(state==0 && fraction > 0.4) state = 1; // require >40% samples in window
  else if(state==1 && fraction < 0.2) state = 0; // hysteresis

  Serial.print("d="); Serial.print(d);
  Serial.print(" cm frac="); Serial.print(fraction,2);
  Serial.print(" present="); Serial.println(state==1);

  delay(sampleInterval);
}
```

ESP32 example (HC-SR04)
-----------------------
This is a drop-in conversion for ESP32 using the Arduino core. Pulse timing works with `pulseIn()` on most ESP32 boards; for higher reliability consider using the RMT peripheral.

``` cpp
#include <Arduino.h>

#define TRIG_PIN 18
#define ECHO_PIN 19

unsigned long sampleInterval = 200; // ms (5 Hz)
const int WINDOW_SAMPLES = 15; // ~3s window at 200ms
int windowIdx = 0;
int belowThreshCount = 0;
int samples[WINDOW_SAMPLES];
const int DIST_THRESH_CM = 120; // distance indicating presence

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  for(int i=0;i<WINDOW_SAMPLES;i++) samples[i]=0;
}

long measureDistanceCm(){
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long dur = pulseIn(ECHO_PIN, HIGH, 30000UL); // timeout 30ms
  if(dur==0) return -1; // no echo
  long dist = (dur / 2) / 29.1; // cm
  return dist;
}

void loop(){
  long d = measureDistanceCm();
  int below = (d>0 && d < DIST_THRESH_CM) ? 1 : 0;

  // sliding buffer update
  belowThreshCount -= samples[windowIdx];
  samples[windowIdx] = below;
  belowThreshCount += samples[windowIdx];
  windowIdx = (windowIdx + 1) % WINDOW_SAMPLES;

  float fraction = (float)belowThreshCount / WINDOW_SAMPLES;
  static int state = 0; // 0=absent,1=present
  if(state==0 && fraction > 0.4) state = 1; // require >40% samples in window
  else if(state==1 && fraction < 0.2) state = 0; // hysteresis

  Serial.print("d="); Serial.print(d);
  Serial.print(" cm frac="); Serial.print(fraction,2);
  Serial.print(" present="); Serial.println(state==1);

  delay(sampleInterval);
}
```

Example — doorway counter (concept)
-----------------------------------
- Place Sensor A and Sensor B separated across doorway (A outside, B inside).
- Monitor each sensor for a threshold-crossing event; when both trigger within a short time window, use the order (A->B = entry, B->A = exit) to increment/decrement a counter.
- Debounce and require clear gaps to avoid double-counting.

Calibration tips
----------------
- Record baseline empty-room distances at sensor locations.
- Test with volunteers walking at typical speeds through the doorway; tune distance thresholds, sample rate, and window sizes.

Privacy and safety
------------------
- Ultrasonic sensing is non-imaging and privacy-friendly, but store only presence/occupancy events (not raw distances), and inform occupants.

Next steps
----------
- Add dual-sensor doorway counting example code.
- Add MQTT/HTTP integration and sample payloads for HALO.
