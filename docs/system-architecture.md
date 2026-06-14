# 🏗 System Architecture — Detailed Breakdown

This document provides an in-depth look at the architecture of the Autonomous Search & Rescue Drone System.

---

## High-Level Architecture

The system is composed of **three major subsystems** that communicate in real-time:

```
┌─────────────┐       ┌─────────────────┐       ┌──────────────┐
│   PRIMARY    │◄─────►│  COMMUNICATION  │◄─────►│   GROUND     │
│   DRONE      │       │     LAYER       │       │   STATION    │
│              │       │                 │       │              │
│ • Pixhawk FC │       │ • MAVLink       │       │ • QGC        │
│ • Raspberry  │       │ • Video Stream  │       │ • ML Engine  │
│   Pi + Cam   │       │ • RC Control    │       │ • Dashboard  │
│ • Mini-Drone │       │                 │       │              │
│   Dock       │       │                 │       │              │
└─────────────┘       └─────────────────┘       └──────────────┘
```

---

## 1. Primary Drone Subsystem

### Flight Controller (Pixhawk + PX4)
The brain of the drone handles:
- **Attitude control** — Stabilizing the drone using IMU (accelerometer + gyroscope)
- **Position hold** — Maintaining GPS position during hover (critical for mini-drone deployment)
- **Waypoint navigation** — Following the auto-generated serpentine path
- **Failsafe execution** — Autonomous RTL on low battery, GPS loss, or link failure

### Companion Computer (Raspberry Pi 4)
Handles tasks that the flight controller cannot:
- **Camera capture** — Interfacing with the Pi Camera Module via CSI ribbon cable
- **Video encoding** — H.264 encoding using the Pi's hardware encoder
- **Streaming** — Transmitting the encoded video over Wi-Fi to the ground station
- **MAVLink proxy** — Optionally forwarding MAVLink telemetry over the network

### GPS Module (u-blox M8N)
- Provides position data for autonomous navigation
- Used to **geo-tag detections** — when the ML model detects a human/animal, the drone's current GPS position is logged alongside the detection

### Mini-Drone Docking Bay
- Custom 3D-printed mount secured to the underside of the primary drone
- **Servo-actuated release mechanism** — A servo motor controls a latch that holds the mini-drone in place
- **Magnetic alignment** — Neodymium magnets assist in docking alignment when re-attaching
- **Electrical interface** — Optional charging connection while docked

---

## 2. Communication Layer

### Telemetry (MAVLink)
```
Pixhawk ──[UART]──► SiK Radio TX ~~[915MHz RF]~~► SiK Radio RX ──[USB]──► GCS Laptop
```
- **Protocol**: MAVLink v2
- **Data**: Attitude, position, battery, mission progress, system status
- **Range**: 1-2 km (with SiK telemetry radios)

### Video Stream
```
Pi Camera ──[CSI]──► Raspberry Pi ──[Wi-Fi]──► GCS Laptop
```
- **Pipeline**: `raspivid` → `gstreamer` → UDP/RTSP stream
- **Resolution**: 720p @ 30fps (configurable)
- **Latency**: ~200-400ms depending on network conditions

### Mini-Drone RC Control
```
RC Controller ──[2.4GHz]──► Mini-Drone RX
```
- Separate control channel from the primary drone
- FPV video from mini-drone transmitted on a different frequency to avoid interference

---

## 3. Ground Station Subsystem

### QGroundControl
- **Mission planning**: Input 4 GPS waypoints → auto-generate coverage path
- **Live monitoring**: Real-time map view, telemetry gauges, video feed
- **Parameter tuning**: Adjust flight speed, altitude, failsafe parameters
- **Log analysis**: Post-flight log review and analysis

### ML Detection Engine
- **Input**: Receives video stream from the Raspberry Pi
- **Processing**: Runs YOLOv8 inference on each frame
- **Output**: Annotated frames with bounding boxes + detection alerts
- **Logging**: Saves detections with timestamp, GPS coordinate, confidence score, and annotated image

### Custom Dashboard
- Displays detection history and GPS locations on a map
- Provides manual controls for mini-drone deployment
- Audio/visual alerts for survivor detection

---

## Data Flow Diagram

```
                    ┌─────────────────────────┐
                    │     MISSION START        │
                    │  4 GPS waypoints input   │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   PATH GENERATION       │
                    │  Serpentine coverage     │
                    │  computed from waypoints │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   MISSION UPLOAD        │
                    │  MAVLink → Pixhawk      │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   AUTONOMOUS FLIGHT     │
                    │  Drone follows path     │
                    │  Camera streams video   │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
    ┌─────────▼──────┐  ┌───────▼────────┐  ┌──────▼──────────┐
    │  TELEMETRY     │  │  VIDEO STREAM  │  │  ML DETECTION   │
    │  → QGC         │  │  → Display     │  │  → YOLOv8       │
    │  Monitoring     │  │  Live Feed     │  │  → Alerts       │
    └────────────────┘  └────────────────┘  └──────┬──────────┘
                                                    │
                                          ┌─────────▼─────────┐
                                          │  DETECTION LOG    │
                                          │  GPS + Timestamp  │
                                          │  + Confidence     │
                                          │  + Screenshot     │
                                          └───────────────────┘
```

---

## Scalability Considerations

- **Multiple mini-drones**: The docking system can be extended to carry 2+ mini-drones for larger disaster sites
- **Swarm search**: Multiple primary drones can divide the search area and coordinate via a central GCS
- **Relay network**: In larger areas, intermediate relay drones can extend communication range
- **Edge inference**: Future iterations could run lightweight ML models directly on the Raspberry Pi to reduce latency
