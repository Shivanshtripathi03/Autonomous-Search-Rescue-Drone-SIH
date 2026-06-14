# 🚀 Mini-Drone Deployment System

This document covers the design, operation, and engineering of the deployable mini-drone subsystem — the key innovation of our SAR platform.

---

## Concept

In disaster scenarios like building collapses or earthquakes, large drones cannot enter:
- **Collapsed structures** — Gaps in rubble too small for the primary drone
- **Partially standing buildings** — Windows, doorways, corridors
- **Dense vegetation or debris** — Forest cover after landslides
- **Underground spaces** — Basements, parking structures

Our solution: carry a **miniature FPV drone** on the primary drone that can be **deployed mid-mission** to explore these confined spaces.

---

## Operational Workflow

### Phase 1: Docked Transit
```
┌─────────────────────────────────────────────┐
│           PRIMARY DRONE (Flying)            │
│                                             │
│    ┌───────────────────────────────────┐    │
│    │          Docking Bay              │    │
│    │   ┌───────────────────────┐      │    │
│    │   │     MINI DRONE        │      │    │
│    │   │   (Powered Down)      │      │    │
│    │   │   Magnets Engaged     │      │    │
│    │   │   Latch Secured       │      │    │
│    │   └───────────────────────┘      │    │
│    └───────────────────────────────────┘    │
│                                             │
│    Status: Autonomous search in progress    │
└─────────────────────────────────────────────┘
```

### Phase 2: Deployment Trigger
The operator decides to deploy the mini-drone when:
- Visual inspection of the video feed shows a structure that needs interior exploration
- The ML model detects a possible survivor signal near a collapsed building
- The primary drone reaches a pre-marked POI (Point of Interest) that requires closer inspection

### Phase 3: Hover & Deploy
```
Step 1: Primary drone switches to HOLD mode (maintains GPS position)
Step 2: Primary drone descends to safe deployment altitude (~5-10m)
Step 3: Operator arms the mini-drone remotely
Step 4: Servo actuates → Latch opens → Mini-drone released
Step 5: Mini-drone motors spin up → Operator takes FPV control
```

### Phase 4: Tight-Space Search
```
                    ┌──────────────────────┐
                    │  Collapsed Building  │
                    │    ╔═══╗             │
                    │    ║   ║  ←── Gap    │
        ┌───┐       │    ║   ║             │
        │   │ ─────►│    ║ ☺ ║  Survivor!  │
        │Mini│       │    ║   ║             │
        │Dro│       │    ╚═══╝             │
        │ne │       │                      │
        └───┘       └──────────────────────┘
     
     Operator controls via FPV
     Camera feed → Ground Station → ML Detection
```

The mini-drone:
- Is manually piloted via FPV goggles or a monitor
- Navigates through gaps, windows, corridors
- Transmits live video to the ground station
- Ground station ML model can process this feed for detections
- Operator marks locations of interest for rescue teams

### Phase 5: Re-attachment
```
Step 1: Operator flies mini-drone back to primary drone position
Step 2: Primary drone remains in HOLD (hovering in place)
Step 3: Mini-drone approaches from below the docking bay
Step 4: Magnetic alignment pulls mini-drone into dock
Step 5: Servo latch re-engages to secure the mini-drone
Step 6: Mini-drone disarmed
Step 7: Primary drone resumes autonomous search from last waypoint
```

---

## Docking Mechanism Design

### Mechanical Design

```
            PRIMARY DRONE UNDERSIDE
    ┌──────────────────────────────────────┐
    │                                      │
    │    [N]──────────────────────[N]       │  ← Neodymium magnets (dock side)
    │     │   ┌──────────────┐    │        │
    │     │   │  Servo Latch │    │        │
    │     │   │   ┌──────┐   │    │        │
    │     │   │   │ Pin  │   │    │        │
    │     │   └───┴──────┴───┘    │        │
    │    [N]──────────────────────[N]       │
    │     ▲         ▲              ▲       │
    │     │    Alignment Pins      │       │
    │     │    (Guide Rails)       │       │
    └─────┴────────────────────────┴───────┘
    
    
            MINI DRONE TOP PLATE
    ┌──────────────────────────────────────┐
    │                                      │
    │    [S]──────────────────────[S]       │  ← Matching magnets (attract)
    │     │   ┌──────────────┐    │        │
    │     │   │  Latch Slot  │    │        │
    │     │   └──────────────┘    │        │
    │    [S]──────────────────────[S]       │
    │     ▲         ▲              ▲       │
    │     │    Alignment Recesses  │       │
    └─────┴────────────────────────┴───────┘
```

### Components

| Part | Material | Purpose |
|------|----------|---------|
| Dock Frame | PLA/PETG (3D printed) | Structural mount on primary drone |
| Alignment Pins | Aluminum dowel | Guide mini-drone into correct position |
| Alignment Recesses | 3D printed | Corresponding slots on mini-drone |
| Magnets (×8) | N52 Neodymium (6mm disc) | Passive alignment force |
| Servo Latch | SG90 servo + pin | Active lock/release mechanism |
| Latch Slot | 3D printed | Receives the latch pin |

### Magnetic Force Calculation
- Each N52 neodymium disc magnet (6mm × 3mm) provides ~1.5 kg pull force
- 4 magnet pairs = ~6 kg combined pull force
- Mini-drone weight: ~80-100g
- **Safety factor: ~60x** — ensures reliable docking even in wind

---

## Communication Architecture During Deployment

```
                                      GROUND STATION
                                    ┌───────────────────┐
              MAVLink Telemetry ───►│  QGroundControl    │
              (Primary Drone)       │  • Primary status  │
                                    │  • Position hold   │
    ┌─────────┐                     │                    │
    │ Primary │ ──── Video ────────►│  ML Detection      │
    │ Drone   │    (Raspberry Pi)   │  • Primary cam     │
    │ (HOLD)  │                     │                    │
    └────┬────┘                     │                    │
         │                          │                    │
    ┌────┴────┐                     │                    │
    │  Mini   │ ──── FPV Video ───►│  Mini-Drone Feed   │
    │  Drone  │    (Analog/Digital) │  • Tight-space cam │
    │ (MANUAL)│                     │                    │
    └─────────┘                     │                    │
         ▲                          │                    │
         │       RC Control ◄──────│  RC Controller     │
         └─────────────────────────│  (Manual pilot)    │
                                    └───────────────────┘
```

### Frequency Allocation
To avoid interference, each link operates on a separate frequency:

| Link | Frequency | Protocol |
|------|-----------|----------|
| Primary Telemetry | 915 MHz | MAVLink (SiK Radio) |
| Primary Video | 2.4 GHz (Wi-Fi) | UDP/RTSP (Raspberry Pi) |
| Mini-Drone RC | 2.4 GHz (ELRS/FrSky) | RC protocol (different channel) |
| Mini-Drone FPV | 5.8 GHz | Analog/Digital FPV |

---

## Servo Control Code

```python
"""
Mini-drone dock servo control via Pixhawk AUX output.
Triggered by MAVLink command from the ground station.
"""

import asyncio
from mavsdk import System

# Servo channel for the dock latch (AUX OUT 1)
DOCK_SERVO_CHANNEL = 9  # AUX1 = channel 9 in PX4

# Servo positions (PWM microseconds)
LATCH_LOCKED = 1000   # Latch engaged (mini-drone secured)
LATCH_OPEN = 2000     # Latch released (mini-drone free)

async def deploy_mini_drone(drone):
    """Release the mini-drone from the docking bay."""
    print("⚙️  Switching to HOLD mode...")
    await drone.action.hold()
    await asyncio.sleep(2)  # Stabilize
    
    print("⚙️  Opening dock latch...")
    await drone.action.set_actuator(DOCK_SERVO_CHANNEL, LATCH_OPEN)
    await asyncio.sleep(1)
    
    print("✓ Mini-drone released!")
    print("📡 Operator: Take manual FPV control of the mini-drone")

async def dock_mini_drone(drone):
    """Re-engage the dock latch after mini-drone returns."""
    print("⚙️  Closing dock latch...")
    await drone.action.set_actuator(DOCK_SERVO_CHANNEL, LATCH_LOCKED)
    await asyncio.sleep(1)
    
    print("✓ Mini-drone docked and secured!")
    print("▶️  Resuming autonomous mission...")
    await drone.mission.start_mission()
```

---

## Design Considerations

### Weight Budget
| Component | Weight |
|-----------|--------|
| Mini-drone (with battery) | ~80g |
| Docking mechanism (dock side) | ~50g |
| Servo + mounting hardware | ~20g |
| **Total additional payload** | **~150g** |

The primary hexacopter can easily handle this additional payload with negligible impact on flight time.

### Failure Modes & Mitigation

| Failure | Mitigation |
|---------|-----------|
| Mini-drone fails to release | Redundant servo / manual release via secondary command |
| Mini-drone fails to re-attach | Primary drone can RTL without mini-drone; mini-drone lands independently |
| Magnets not aligning | Operator adjusts approach angle; alignment pins provide guidance |
| Primary drone drifts during deployment | GPS HOLD mode maintains position within ±1m |
| Communication loss with mini-drone | Mini-drone has failsafe: auto-land after 5s signal loss |
