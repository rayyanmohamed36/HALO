
# ESP32 BLE Room-Detection Method (wearable proximity)

Overview
- Use one or more ESP32 devices as BLE scanners (central) placed in rooms. A person wears a BLE peripheral (small beacon or wearable) that advertises a known Service UUID or uses a fixed MAC/BLE address. Each ESP32 scans for that advertisement and estimates presence by RSSI + filtering.

Hardware
- Wearable: low-energy BLE peripheral advertising a unique Service UUID (or unique address). Configure advertise interval ~200–1000 ms and moderate TX power.
- Anchors: ESP32 (ESP32-WROOM or similar) with Wi‑Fi to forward detections (MQTT/HTTP) or act locally for alerts.

Algorithm (single-anchor / room-level)
1. ESP32 scans continuously (or duty-cycled) for BLE advertisements.
2. When an advertisement matches the wearable (by Service UUID or device address), record the RSSI and timestamp.
3. Apply smoothing (e.g., exponential moving average) to RSSI to reduce jitter.
4. Use calibrated RSSI threshold(s) and hysteresis:
	 - Enter threshold (e.g., RSSI > -65 dBm) to mark "in-room".
	 - Exit threshold lower (e.g., RSSI < -75 dBm) to avoid flapping.
5. Require a dwell time (e.g., 3–10 s) after threshold crossing before raising an in-room event.
6. Optionally send detection events to a server (MQTT/HTTP) including anchor id, smoothed RSSI, and timestamp.

Algorithm (multi-anchor / improved accuracy)
- Deploy 2+ ESP32 anchors per area; each reports smoothed RSSI. A server applies voting or simple proximity (highest RSSI) to decide which room the wearable is in. For better accuracy, use fingerprinting or calibrate path-loss and apply multilateration.

Triangulation example:
https://www.beaconzone.co.uk/blog/wp-content/uploads/2021/09/trilateration-1.webp

Signal processing & calibration
- Smooth RSSI: EMA with alpha ~0.2 or a small Kalman filter.
- Calibrate per-site: measure RSSI at representative positions to pick thresholds; building materials and interference change numbers.
- Compensate for TX power: if wearable reports advertised TX power, use it for distance estimation.

Battery and scanning tradeoffs
- Scan interval and window: continuous scan gives fastest updates but uses more power on the anchor (ESP32) — acceptable when mains powered. Duty-cycle scanning to save anchor energy if needed.
- Wearable advertise interval: shorter interval increases responsiveness but reduces wearable battery life. 200–500 ms is a common compromise.

Privacy & robustness
- Avoid storing raw MACs centrally if privacy is a concern; use hashed IDs or rotating identifiers.
- Combine BLE proximity with motion sensors (PIR, accelerometer on wearable) to reduce false positives.

Example Arduino/ESP32 (BLE scan) sketch — core idea
```cpp
#include <BLEDevice.h>

static const char* TARGET_UUID = "12345678-1234-5678-1234-56789abcdef0"; // wearable service
float emaRssi = NAN;
const float EMA_ALPHA = 0.2;
const int ENTER_RSSI = -65;
const int EXIT_RSSI = -75;
unsigned long enterTimestamp = 0;
bool inRoom = false;

class MyScanCallbacks: public BLEAdvertisedDeviceCallbacks {
	void onResult(BLEAdvertisedDevice advertisedDevice) {
		if (!advertisedDevice.haveServiceUUID()) return;
		if (advertisedDevice.isAdvertisingService(BLEUUID(TARGET_UUID))) {
			int rssi = advertisedDevice.getRSSI();
			if (isnan(emaRssi)) emaRssi = rssi;
			else emaRssi = EMA_ALPHA * rssi + (1 - EMA_ALPHA) * emaRssi;

			unsigned long now = millis();
			if (!inRoom && emaRssi > ENTER_RSSI) {
				if (enterTimestamp == 0) enterTimestamp = now;
				else if (now - enterTimestamp > 3000) { // 3s dwell
					inRoom = true;
					// publish/handle "entered" event
				}
			} else if (inRoom && emaRssi < EXIT_RSSI) {
				// immediate exit hysteresis or require dwell similarly
				inRoom = false;
				enterTimestamp = 0;
				// publish/handle "exited" event
			} else if (emaRssi <= ENTER_RSSI) {
				enterTimestamp = 0;
			}
		}
	}
};

void setup() {
	Serial.begin(115200);
	BLEDevice::init("");
	BLEScan* pBLEScan = BLEDevice::getScan();
	pBLEScan->setAdvertisedDeviceCallbacks(new MyScanCallbacks());
	pBLEScan->setActiveScan(true);
	pBLEScan->start(0, nullptr); // continuous
}

void loop() {
	// handle state, publish via WiFi/MQTT when state changes
	delay(1000);
}
```