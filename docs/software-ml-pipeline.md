# 💻 Software & ML Pipeline

This document covers the complete software stack and the machine learning detection pipeline for real-time aerial human and animal detection.

---

## Software Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       GROUND STATION                           │
│                                                                │
│  ┌────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ QGroundControl  │  │  Detection App   │  │  Alert System │  │
│  │                 │  │                  │  │               │  │
│  │ • Mission Plan  │  │ • Video Receiver │  │ • GPS Logger  │  │
│  │ • Telemetry     │  │ • YOLOv8 Engine  │  │ • Audio Alert │  │
│  │ • Parameters    │  │ • OpenCV Viz     │  │ • Map Overlay │  │
│  └────────┬───────┘  └────────┬─────────┘  └───────┬───────┘  │
│           │                   │                      │          │
│           └───────────────────┼──────────────────────┘          │
│                               │                                 │
└───────────────────────────────┼─────────────────────────────────┘
                                │
                    ┌───────────┴──────────┐
                    │  COMMUNICATION BUS   │
                    │  MAVLink + Video     │
                    └───────────┬──────────┘
                                │
┌───────────────────────────────┼─────────────────────────────────┐
│                        DRONE (AIRBORNE)                         │
│                               │                                 │
│  ┌────────────────┐  ┌───────┴────────┐  ┌──────────────────┐  │
│  │ PX4 Autopilot  │  │ Raspberry Pi   │  │ Mini-Drone Dock  │  │
│  │                 │  │                │  │                  │  │
│  │ • Navigation   │  │ • Camera Grab  │  │ • Servo Control  │  │
│  │ • Stabilization│  │ • H.264 Encode │  │ • Latch Status   │  │
│  │ • Waypoint Mgr │  │ • UDP Stream   │  │                  │  │
│  └────────────────┘  └────────────────┘  └──────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## PX4 Autopilot Configuration

### Key Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `NAV_ACC_RAD` | 2.0 m | Waypoint acceptance radius |
| `MPC_XY_VEL_MAX` | 5.0 m/s | Max horizontal velocity |
| `MPC_Z_VEL_MAX_UP` | 3.0 m/s | Max ascent rate |
| `MPC_Z_VEL_MAX_DN` | 1.5 m/s | Max descent rate |
| `RTL_RETURN_ALT` | 30 m | RTL (Return to Launch) altitude |
| `COM_LOW_BAT_ACT` | RTL | Action on low battery |
| `GF_ACTION` | RTL | Action on geofence breach |
| `COM_DL_LOSS_T` | 10 s | Data link loss timeout |
| `MIS_DIST_1WP` | 500 m | Max distance to first waypoint |

### Mission Script (MAVSDK Python)

```python
"""
Autonomous serpentine search pattern generator and executor.
Takes 4 GPS waypoints as input and generates a lawnmower coverage path.
"""

import asyncio
from mavsdk import System
from mavsdk.mission import MissionItem, MissionPlan
import math

async def generate_coverage_path(waypoints, spacing_m=10, altitude=25):
    """
    Generate a serpentine coverage path from 4 corner waypoints.
    
    Args:
        waypoints: List of 4 (lat, lon) tuples defining the search area
        spacing_m: Spacing between flight lines (based on camera FOV)
        altitude: Flight altitude in meters
    
    Returns:
        List of MissionItem objects
    """
    # Sort waypoints to form a proper rectangle
    wp1, wp2, wp3, wp4 = waypoints
    
    # Calculate the number of passes needed
    width = haversine_distance(wp1, wp4)
    num_passes = int(width / spacing_m) + 1
    
    mission_items = []
    
    for i in range(num_passes):
        # Interpolate start and end points for this pass
        fraction = i / max(num_passes - 1, 1)
        
        if i % 2 == 0:  # Forward pass
            start = interpolate(wp1, wp4, fraction)
            end = interpolate(wp2, wp3, fraction)
        else:  # Reverse pass
            start = interpolate(wp2, wp3, fraction)
            end = interpolate(wp1, wp4, fraction)
        
        mission_items.append(
            MissionItem(
                start[0], start[1], altitude,
                speed_m_s=5.0,
                is_fly_through=True,
                gimbal_pitch_deg=-90,  # Camera pointing down
                gimbal_yaw_deg=0,
                camera_action=MissionItem.CameraAction.START_VIDEO,
                loiter_time_s=0,
                camera_photo_interval_s=0,
                acceptance_radius_m=2.0,
                yaw_deg=float('nan'),
                camera_photo_distance_m=0
            )
        )
        
        mission_items.append(
            MissionItem(
                end[0], end[1], altitude,
                speed_m_s=5.0,
                is_fly_through=True,
                gimbal_pitch_deg=-90,
                gimbal_yaw_deg=0,
                camera_action=MissionItem.CameraAction.NONE,
                loiter_time_s=0,
                camera_photo_interval_s=0,
                acceptance_radius_m=2.0,
                yaw_deg=float('nan'),
                camera_photo_distance_m=0
            )
        )
    
    return mission_items


async def run_mission():
    """Connect to the drone and execute the search mission."""
    drone = System()
    await drone.connect(system_address="udp://:14540")
    
    # Wait for connection
    print("Waiting for drone connection...")
    async for state in drone.core.connection_state():
        if state.is_connected:
            print("✓ Drone connected!")
            break
    
    # Wait for GPS fix
    print("Waiting for GPS fix...")
    async for health in drone.telemetry.health():
        if health.is_global_position_ok and health.is_home_position_ok:
            print("✓ GPS fix acquired!")
            break
    
    # Define search area (4 GPS waypoints)
    search_area = [
        (28.6139, 77.2090),  # WP1 - Top Left
        (28.6120, 77.2090),  # WP2 - Bottom Left
        (28.6120, 77.2120),  # WP3 - Bottom Right
        (28.6139, 77.2120),  # WP4 - Top Right
    ]
    
    # Generate coverage path
    mission_items = await generate_coverage_path(
        search_area, spacing_m=10, altitude=25
    )
    
    # Upload and start mission
    mission_plan = MissionPlan(mission_items)
    await drone.mission.upload_mission(mission_plan)
    print(f"✓ Mission uploaded: {len(mission_items)} waypoints")
    
    await drone.action.arm()
    print("✓ Armed")
    
    await drone.mission.start_mission()
    print("✓ Mission started — autonomous search in progress")
    
    # Monitor mission progress
    async for progress in drone.mission.mission_progress():
        print(f"  Mission progress: {progress.current}/{progress.total}")
        if progress.current == progress.total:
            print("✓ Mission complete!")
            break
    
    # Return to launch
    await drone.action.return_to_launch()
    print("✓ Returning to launch")


if __name__ == "__main__":
    asyncio.run(run_mission())
```

---

## ML Detection Pipeline

### Model Training

We used **YOLOv8** (Ultralytics) trained on aerial/drone imagery datasets:

```python
from ultralytics import YOLO

# Load pre-trained YOLOv8 model
model = YOLO('yolov8n.pt')  # nano variant for speed

# Train on custom aerial SAR dataset
results = model.train(
    data='sar_dataset.yaml',
    epochs=100,
    imgsz=640,
    batch=16,
    name='sar_detector',
    device=0  # GPU
)
```

### Dataset Configuration (`sar_dataset.yaml`)
```yaml
path: ./datasets/sar_aerial
train: images/train
val: images/val
test: images/test

names:
  0: human
  1: animal
```

### Real-Time Detection Script

```python
"""
Ground station ML detection pipeline.
Receives video stream from drone and runs YOLOv8 inference.
"""

import cv2
from ultralytics import YOLO
from datetime import datetime
import json

class SARDetector:
    def __init__(self, model_path='best.pt', confidence=0.5):
        self.model = YOLO(model_path)
        self.confidence = confidence
        self.detections_log = []
    
    def process_frame(self, frame, drone_gps=None):
        """
        Run detection on a single frame.
        
        Args:
            frame: OpenCV image (BGR)
            drone_gps: (lat, lon) tuple from telemetry
        
        Returns:
            Annotated frame, list of detections
        """
        results = self.model(frame, conf=self.confidence, verbose=False)
        detections = []
        
        for result in results:
            for box in result.boxes:
                cls = int(box.cls[0])
                conf = float(box.conf[0])
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                
                label = self.model.names[cls]
                color = (0, 255, 0) if label == 'human' else (0, 255, 255)
                
                # Draw bounding box
                cv2.rectangle(frame, (x1, y1), (x2, y2), color, 2)
                cv2.putText(
                    frame,
                    f'{label} {conf:.2f}',
                    (x1, y1 - 10),
                    cv2.FONT_HERSHEY_SIMPLEX,
                    0.6, color, 2
                )
                
                detection = {
                    'class': label,
                    'confidence': conf,
                    'bbox': [x1, y1, x2, y2],
                    'timestamp': datetime.now().isoformat(),
                    'gps': drone_gps
                }
                detections.append(detection)
                
                # Alert on human detection
                if label == 'human' and conf > 0.7:
                    print(f"🚨 HUMAN DETECTED! Confidence: {conf:.1%} | GPS: {drone_gps}")
                    self.detections_log.append(detection)
        
        return frame, detections
    
    def save_detections(self, filepath='detections.json'):
        """Save all logged detections to a JSON file."""
        with open(filepath, 'w') as f:
            json.dump(self.detections_log, f, indent=2)


def main():
    """Main detection loop — receives drone video and processes it."""
    detector = SARDetector(model_path='best.pt', confidence=0.5)
    
    # Open video stream from Raspberry Pi
    # GStreamer pipeline for receiving UDP stream
    gst_pipeline = (
        'udpsrc port=5600 ! application/x-rtp,payload=96 ! '
        'rtph264depay ! h264parse ! avdec_h264 ! '
        'videoconvert ! appsink'
    )
    cap = cv2.VideoCapture(gst_pipeline, cv2.CAP_GSTREAMER)
    
    if not cap.isOpened():
        # Fallback to direct camera or file for testing
        cap = cv2.VideoCapture(0)
    
    print("✓ Detection pipeline started")
    
    while True:
        ret, frame = cap.read()
        if not ret:
            continue
        
        # Get current GPS from telemetry (would come from MAVLink in real setup)
        drone_gps = (28.6130, 77.2105)  # Example coordinates
        
        # Run detection
        annotated_frame, detections = detector.process_frame(frame, drone_gps)
        
        # Display
        cv2.imshow('SAR Detection - Ground Station', annotated_frame)
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    # Save all detections
    detector.save_detections()
    cap.release()
    cv2.destroyAllWindows()


if __name__ == '__main__':
    main()
```

---

## Video Streaming Pipeline

### Drone Side (Raspberry Pi)
```bash
#!/bin/bash
# stream_video.sh - Start video stream from Pi Camera to Ground Station

GCS_IP="192.168.1.100"  # Ground station IP
PORT=5600

raspivid -t 0 -w 1280 -h 720 -fps 30 -b 2000000 -o - | \
  gst-launch-1.0 -v \
    fdsrc ! \
    h264parse ! \
    rtph264pay config-interval=1 pt=96 ! \
    udpsink host=$GCS_IP port=$PORT
```

### Ground Station Side
```bash
#!/bin/bash
# receive_video.sh - Receive and display video stream

gst-launch-1.0 -v \
  udpsrc port=5600 ! \
  application/x-rtp,payload=96 ! \
  rtph264depay ! \
  h264parse ! \
  avdec_h264 ! \
  videoconvert ! \
  autovideosink sync=false
```

---

## Dependencies

```
# requirements.txt
ultralytics>=8.0.0
opencv-python>=4.8.0
numpy>=1.24.0
mavsdk>=1.4.0
pymavlink>=2.4.0
```
