# 🧪 Simulation & Testing Guide

This document covers the simulation setup, HITL testing process, and validation procedures used to verify the SAR drone system before real-world deployment.

---

## Why Simulate First?

Testing autonomous drones in real life is:
- **Expensive** — Crashes destroy hardware ($$$)
- **Dangerous** — Uncontrolled drones are a safety hazard
- **Time-consuming** — Field testing requires travel, weather, permissions
- **Limited** — Can't easily reproduce edge cases

PX4's simulation framework lets us validate **everything** in software before risking a single flight.

---

## Simulation Modes

### 1. SITL (Software-In-The-Loop)
```
┌──────────────────┐        ┌───────────────────┐
│    PX4 Firmware   │◄──────►│   Physics Engine  │
│  (runs on laptop) │        │   (Gazebo/jMAVSim)│
└────────┬─────────┘        └───────────────────┘
         │
         │ MAVLink (localhost)
         ▼
┌──────────────────┐
│  QGroundControl   │
│  (Mission + GCS)  │
└──────────────────┘
```

- **Everything runs on the laptop** — no hardware needed
- PX4 firmware compiled for the host platform
- Gazebo provides realistic 3D physics and sensor simulation
- Perfect for rapid prototyping and algorithm testing

### 2. HITL (Hardware-In-The-Loop)
```
┌──────────────────┐        ┌───────────────────┐
│  Pixhawk (real FC)│◄─USB──►│   Physics Engine  │
│  PX4 firmware     │        │   (Gazebo/jMAVSim)│
│  running on HW    │        │   (runs on laptop)│
└────────┬─────────┘        └───────────────────┘
         │
         │ MAVLink (USB/Serial)
         ▼
┌──────────────────┐
│  QGroundControl   │
│  (Mission + GCS)  │
└──────────────────┘
```

- **Real flight controller** in the loop — firmware runs on the actual Pixhawk
- Physics and sensors are still simulated on the laptop
- Tests the actual hardware (FC, sensors) against simulated environments
- Much closer to real flight behavior

---

## HITL Setup Procedure

### Prerequisites
- Pixhawk flight controller (flashed with PX4 firmware)
- USB cable (Pixhawk → Laptop)
- QGroundControl installed
- Gazebo simulator installed
- PX4 source code / toolchain

### Step 1: Flash PX4 with HITL Configuration
```bash
# Clone PX4 source
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
cd PX4-Autopilot

# Build for HITL
make px4_fmu-v5_default  # For Pixhawk 4

# Flash the firmware
# (Or use QGroundControl: Vehicle Setup → Firmware → PX4)
```

### Step 2: Configure HITL in QGroundControl
1. Connect Pixhawk via USB
2. Open QGroundControl → Vehicle Setup → Safety
3. Enable **HITL mode**: Set `SYS_HITL = 1`
4. Select airframe: **HIL Quadcopter X** (or appropriate multirotor)
5. Reboot the flight controller

### Step 3: Launch the Simulator
```bash
# Start Gazebo with HITL configuration
cd PX4-Autopilot
make px4_sitl_default gazebo  # For SITL

# For HITL, Gazebo connects to the real FC via serial
# Ensure the serial port matches your Pixhawk connection
```

### Step 4: Connect QGroundControl
- QGC auto-detects the Pixhawk on USB
- You should see simulated telemetry data (altitude, GPS, battery)
- The simulated drone appears in the Gazebo 3D view

---

## Test Cases

### Test 1: Autonomous Waypoint Mission
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Define 4 GPS waypoints in QGC | Waypoints appear on map |
| 2 | Generate serpentine path | Coverage path fills the search area |
| 3 | Upload mission | "Mission uploaded" confirmation |
| 4 | Arm and start mission | Drone takes off autonomously |
| 5 | Monitor flight path | Drone follows serpentine pattern |
| 6 | Mission complete | Drone enters HOLD or RTL |

### Test 2: Position Hold (for Mini-Drone Deployment)
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Mid-mission, switch to HOLD | Drone holds GPS position |
| 2 | Wait 60 seconds | Position drift < 1m |
| 3 | Resume mission | Drone continues from last waypoint |

### Test 3: Failsafe — Low Battery
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Simulate battery drain | Battery level drops in telemetry |
| 2 | Battery reaches threshold | Warning triggered in QGC |
| 3 | Battery reaches critical | Drone initiates RTL automatically |
| 4 | Drone returns to launch | Lands at home position |

### Test 4: Failsafe — GPS Loss
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Simulate GPS signal loss | GPS status → No Fix |
| 2 | Wait for timeout | Drone enters failsafe mode |
| 3 | Failsafe action | Drone lands or holds position (configurable) |

### Test 5: Failsafe — Communication Link Loss
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Disconnect telemetry radio | Link lost in QGC |
| 2 | Wait for `COM_DL_LOSS_T` timeout | Drone triggers data link loss failsafe |
| 3 | Failsafe action | Drone initiates RTL |

### Test 6: Geofence Breach
| Step | Action | Expected Result |
|------|--------|----------------|
| 1 | Define geofence in QGC | Fence appears on map |
| 2 | Command drone outside fence | Drone approaches boundary |
| 3 | Geofence triggered | Drone stops and initiates RTL |

---

## Video Stream Testing

### Local Testing (Without Drone)
```bash
# Simulate a video stream using a webcam or test video
# On the "drone" side (or local machine):
gst-launch-1.0 videotestsrc ! \
  x264enc tune=zerolatency ! \
  rtph264pay ! \
  udpsink host=127.0.0.1 port=5600

# On the "ground station" side:
python detection_pipeline.py  # Uses GStreamer to receive and process
```

### With Raspberry Pi (Pre-Flight)
```bash
# Connect Pi to the same network as the ground station
# On the Pi:
./stream_video.sh  # Starts the camera stream

# On the ground station:
python detection_pipeline.py  # Should show live annotated feed
```

---

## Test Results Summary

| Test | Status | Notes |
|------|--------|-------|
| Waypoint Mission | ✅ Pass | Serpentine pattern covers full area |
| Position Hold | ✅ Pass | Drift < 0.5m over 60s |
| Low Battery RTL | ✅ Pass | RTL triggered at configured threshold |
| GPS Loss Failsafe | ✅ Pass | Land mode activated |
| Link Loss Failsafe | ✅ Pass | RTL after 10s timeout |
| Geofence | ✅ Pass | Drone stopped at boundary |
| Video Stream (local) | ✅ Pass | 720p @ 30fps, ~200ms latency |
| ML Detection (local) | ✅ Pass | ~25 FPS inference on GTX 1650 |
| Camera + Detection | ✅ Pass | End-to-end pipeline functional |

---

## Known Limitations of Simulation

| Limitation | Real-World Impact |
|------------|------------------|
| No wind simulation (basic) | Real flights affected by wind gusts |
| Idealized GPS accuracy | Real GPS has ±2-5m accuracy |
| No RF interference | Real radio links may have interference |
| No physical vibration | Vibration affects camera image quality |
| Simplified battery model | Real discharge curves are non-linear |

These were accounted for by adding safety margins to all parameters during real-world tuning.
