# 🔧 Hardware Setup Guide

This document details the hardware components, assembly process, and wiring for the Autonomous Search & Rescue Drone System.

---

## Primary Drone Assembly

### Bill of Materials

| # | Component | Qty | Notes |
|---|-----------|-----|-------|
| 1 | Hexacopter Frame (F550/Custom) | 1 | 550mm class frame |
| 2 | Pixhawk 4 / Cube Orange FC | 1 | Running PX4 firmware |
| 3 | u-blox M8N GPS + Compass | 1 | Mounted on mast away from interference |
| 4 | Brushless Motors 2212 920KV | 6 | Matched CW/CCW pairs |
| 5 | ESC 30A BLHeli | 6 | One per motor |
| 6 | Propellers 10x4.5 | 6 | 3 CW + 3 CCW |
| 7 | 4S 5000mAh LiPo Battery | 1-2 | ~15-20 min flight time |
| 8 | Power Distribution Board | 1 | Distributes battery power to ESCs |
| 9 | Raspberry Pi 4 (4GB RAM) | 1 | Companion computer |
| 10 | Pi Camera Module v2 | 1 | 8MP, bottom-mounted |
| 11 | SiK Telemetry Radio 915MHz | 1 pair | Air + Ground unit |
| 12 | Servo Motor (SG90) | 1 | Mini-drone release mechanism |
| 13 | 3D-Printed Docking Mount | 1 | Custom designed |
| 14 | Vibration Dampening Mount | 1 | For Pixhawk isolation |

### Wiring Diagram

```
Battery (4S LiPo)
    │
    ▼
┌──────────────────┐
│  Power Module     │──── Voltage/Current sense ──► Pixhawk POWER port
│  (XT60 → PDB)    │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  PDB (Power      │
│  Distribution)    │
└┬──┬──┬──┬──┬──┬─┘
 │  │  │  │  │  │
 ▼  ▼  ▼  ▼  ▼  ▼
ESC ESC ESC ESC ESC ESC  ──► Each ESC signal wire → Pixhawk MAIN OUT 1-6
 │   │   │   │   │   │
 ▼   ▼   ▼   ▼   ▼   ▼
 M1  M2  M3  M4  M5  M6    (Motors)


Pixhawk TELEM1 ──[UART]──► SiK Radio (Air)
Pixhawk TELEM2 ──[UART]──► Raspberry Pi (Serial MAVLink)
Pixhawk AUX OUT 1 ──────► Servo Motor (Mini-drone release)
Pixhawk GPS port ────────► u-blox M8N GPS Module


Raspberry Pi
├── CSI port ──────► Pi Camera Module v2
├── USB ────────────► (optional) USB telemetry
├── Wi-Fi ──────────► Video streaming to ground station
└── GPIO ───────────► (optional) Status LEDs, buzzer
```

---

## Raspberry Pi Camera Setup

### Physical Mounting
1. Mount the Pi Camera Module to the **bottom of the drone frame** using a 3D-printed bracket
2. Ensure the camera faces **straight down** for optimal aerial coverage
3. Use a **short CSI ribbon cable** (15cm) to minimize interference
4. Secure all cables with zip ties to prevent vibration-induced disconnection

### Camera Configuration
```bash
# Enable camera interface
sudo raspi-config
# Navigate to Interface Options → Camera → Enable

# Test camera
raspistill -o test.jpg

# Start video stream for ground station
raspivid -t 0 -w 1280 -h 720 -fps 30 -b 2000000 -o - | \
  gst-launch-1.0 fdsrc ! h264parse ! rtph264pay ! \
  udpsink host=<GCS_IP> port=5600
```

---

## Mini-Drone Build

### Components

| # | Component | Specification |
|---|-----------|--------------|
| 1 | Micro Frame | 75-100mm wheelbase |
| 2 | Flight Controller | F4 / F7 micro FC (BetaFlight) |
| 3 | Motors | 0802 / 1103 brushless × 4 |
| 4 | ESC | 4-in-1 micro ESC (12A) |
| 5 | Camera | Micro FPV camera (e.g., Caddx Ant) |
| 6 | VTX | 25-200mW micro video transmitter |
| 7 | Receiver | ELRS / FrSky micro RX |
| 8 | Battery | 1S 450mAh LiPo (with PH2.0 connector) |
| 9 | Propellers | 40mm tri-blade × 4 |
| 10 | Magnets | Neodymium disc magnets × 4 (for docking) |

### Docking Mechanism
1. **Primary drone side**: 3D-printed dock with alignment pins and neodymium magnets (attracting polarity)
2. **Mini-drone side**: Matching magnet positions on top plate + alignment recesses
3. **Release**: Servo motor on the primary drone actuates a physical latch
4. **Re-attach**: Operator manually guides the mini-drone to hover beneath the primary drone; magnets pull it into alignment

---

## Ground Station Setup

### Hardware
- **Laptop**: Any laptop with a dedicated GPU (for ML inference)
  - Recommended: NVIDIA GPU with CUDA support
  - Minimum: GTX 1050 Ti / RTX 3050 or equivalent
- **SiK Radio (Ground Unit)**: Connected via USB to the laptop
- **RC Controller**: FrSky Taranis X9D+ or RadioMaster TX16S
  - Bound to the mini-drone's receiver
  - Used only during manual mini-drone operation
- **FPV Goggles/Monitor**: For mini-drone FPV feed (optional, can use laptop)

### Software Installation
```bash
# Install QGroundControl
# Download from https://docs.qgroundcontrol.com/master/en/getting_started/download_and_install.html

# Install Python dependencies for ML pipeline
pip install ultralytics opencv-python numpy

# Install GStreamer for video reception
# macOS: brew install gstreamer
# Ubuntu: sudo apt install gstreamer1.0-tools gstreamer1.0-plugins-*
```

---

## Safety Checklist (Pre-Flight)

- [ ] All propellers secured with prop nuts / lock nuts
- [ ] Battery voltage at full charge (4.2V per cell for LiPo)
- [ ] Pixhawk calibration complete (accelerometer, compass, radio)
- [ ] GPS lock acquired (minimum 8 satellites recommended)
- [ ] Failsafe parameters configured (RTL on low battery, geofence, link loss)
- [ ] Mini-drone docked and latch secured
- [ ] Raspberry Pi camera streaming confirmed
- [ ] Ground station receiving telemetry and video
- [ ] QGroundControl mission uploaded and verified
- [ ] All observers at safe distance (minimum 30m)
