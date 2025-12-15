
# Indoor Positioning Methods — Quick Notes

- **Bluetooth Low Energy (BLE) beacons:** inexpensive, easy to deploy; room-level to sub-meter accuracy with fingerprinting or multiple receivers. Good battery life for wearables; privacy-friendly when passive.

- **Ultra-Wideband (UWB):** centimeter-level accuracy, excellent for precise tracking and locating within rooms. Higher cost and requires UWB-capable devices or anchors.

- **Wi‑Fi RTT / Fingerprinting:** uses existing Wi‑Fi infrastructure; RTT offers better accuracy where supported. Fingerprinting requires an initial site survey and periodic updates.

- **RFID (active/passive):** passive tags are cheap and low-maintenance for zone entry/exit; active RFID offers longer range and better accuracy but higher cost.

- **Inertial/Pedestrian Dead Reckoning (IMU on wearables):** tracks movement between anchors; useful for short periods but drifts over time—best when fused with anchors (BLE/UWB).

- **Magnetic field mapping:** leverages indoor magnetic anomalies for localization on smartphones; low power and non-intrusive but site-specific and sensitive to environment changes.

- **Computer vision / depth cameras:** high accuracy and rich context (posture, falls), but raises privacy concerns and requires camera placement and processing infrastructure.

- **Passive environmental sensors (PIR, pressure mats):** good for detecting presence, motion and room transitions; low cost but coarse-grained positioning.

- **Hybrid/fusion approaches:** combine two or more methods (e.g., BLE + IMU, UWB + vision) to improve robustness, reduce drift, and balance cost vs. accuracy.

**Practical considerations for dementia care:**

- **Privacy:** prefer solutions minimizing camera use or that anonymize/aggregate data.
- **Battery & maintenance:** choose low-maintenance tags/sensors to reduce caregiver burden.
- **Accuracy vs. cost:** room-level may be sufficient for navigation and safety; reserve high-precision options for critical tasks.
- **Ease of setup / calibration:** fingerprinting and vision systems need more upfront work; BLE/UWB anchors should be easy to deploy.
- **Integration:** combine positioning with fall detection, geofencing, and caregiver alerts for useful interventions.
- **Comfort & acceptance:** wearable devices should be unobtrusive and comfortable for long-term use.