---
layout: post
title: Edge AI Intersection Control System
subtitle: A real-time, vision-enhanced traffic controller built on the Jetson Orin Nano
cover-img: 
thumbnail-img: assets/img/jetson-intersection.jpg
tags: [AI, Machine Learning, Embedded Programming]
author: Dylan Cunliffe
---

# Smart Intersection Control System
*A real-time, vision-enhanced traffic controller built on the Jetson Orin Nano*

---

# Smart Intersection Control System
*A real-time, vision-enhanced traffic controller built on the Jetson Orin Nano*

---

## 🚦 Overview

For this project, I designed and built a complete **smart intersection simulation** using:

- A **Jetson Orin Nano Super Dev Kit**
- **YOLOv8 (ONNX)** real-time object detection
- **Push-button input** for pedestrian crossing
- **LED traffic-light modules**
- A **Python controller** that synchronizes everything using shared files and atomic writes

The system behaves like a real intersection: vehicles on a “road” are detected through a camera, and a pedestrian “walk” button requests crossing. The controller resolves when to allow pedestrians based on traffic presence, minimum-green constraints, and safety clearances.

---

## 📸 Project Setup

> **[Insert the following photos]:**
> - *PHOTO 1 — Full intersection setup (Jetson, LEDs, button, camera)*
> - *PHOTO 2 — Close-up of wiring to the GPIO header*
> - *PHOTO 3 — LED traffic lights operating during detection*
> - *PHOTO 4 — Terminal showing YOLO output*

---

## 1. System Architecture

The system runs using **three cooperating components**:

```text
Camera → YOLOv8 Detector → Shared File → Main Intersection Controller → LEDs + Walk Signal
                                      ↑
                           Pedestrian Button (GPIO)
```

### Components

- **YOLOv8 Object Detection Script**
  Continuously writes `"1"` or `"0"` to `/tmp/side_detected.txt` using atomic file writes.

- **Button Reader**
  Reports the pedestrian button’s state (active-high or active-low depending on wiring).

- **Main Controller**
  Reads both inputs and decides whether to allow:
  - Normal vehicle flow
  - Pedestrian walk phase
  - Delayed crossing if vehicles are present

---

## 2. Object Detection Module (YOLOv8 ONNX)

This module captures frames, detects vehicles, and writes out the shared integer flag.

> *[Insert photo: Camera view with YOLO bounding boxes]*

### Code — YOLO Detector (`yolo_detect.py`) (excerpt)

```python
import cv2
import tempfile
import os
from ultralytics import YOLO

MODEL_PATH = "yolov8n.onnx"
SHARED_FILE_PATH = "/tmp/side_detected.txt"
CONFIDENCE_THRESHOLD = 0.45
VEHICLE_CLASSES = [2, 3, 5, 7]  # car, motorcycle, bus, truck

def write_status(detected):
    content = "1" if detected else "0"
    with tempfile.NamedTemporaryFile(mode='w', delete=False, dir='/tmp') as tmp:
        tmp.write(content)
        tmp.flush()
        os.fsync(tmp.fileno())
        tmp_name = tmp.name
    os.replace(tmp_name, SHARED_FILE_PATH)

model = YOLO(MODEL_PATH, task='detect')
cap = cv2.VideoCapture(0)

# Make window resizable
cv2.namedWindow("YOLOv8 Detection", cv2.WINDOW_NORMAL)
cv2.resizeWindow("YOLOv8 Detection", 1280, 720)

while True:
    ret, frame = cap.read()
    results = model(frame, verbose=False, conf=CONFIDENCE_THRESHOLD)
    car_detected = any(int(b.cls[0]) in VEHICLE_CLASSES for b in results[0].boxes)
    write_status(car_detected)
    annotated = results[0].plot()
    cv2.imshow("YOLOv8 Detection", annotated)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
cap.release()
cv2.destroyAllWindows()
```

---

## 3. Pedestrian Button Input

### Wiring (examples)

**Active-low (recommended)**
* `3.3V` → `10 kΩ` → `GPIO pin`
* `GPIO pin` → `Button` → `GND`
* *Logic:* Unpressed → HIGH, Pressed → LOW

**Active-high**
* `GPIO pin` → `Button` → `3.3V`
* `GPIO pin` → `10 kΩ` → `GND` (pull-down)
* *Logic:* Unpressed → LOW, Pressed → HIGH

> *[Insert circuit diagram/photo: Pedestrian Push-Button Wiring]*

### Code — Button Reader (active-high example)

```python
import Jetson.GPIO as GPIO
import time

PIN_BUTTON = 37
GPIO.setmode(GPIO.BOARD)
GPIO.setup(PIN_BUTTON, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)

while True:
    print(GPIO.input(PIN_BUTTON))  # 1 = pressed (active-high), 0 = not pressed
    time.sleep(0.1)
```

---

## 4. Main Controller Logic

The controller manages:
* **LED traffic lights** (Main + Side: R/Y/G)
* **Pedestrian requests** (latch + guaranteed pedestrian time)
* **Vehicle requests from YOLO** (latched, adaptive side-green extension)
* **Safety phases** (yellow + all-red intervals)

> *[Insert diagram: State Machine Diagram for Intersection Logic]*

### Code — Main Controller (excerpt)

```python
# pin assignments (BOARD numbers)
PIN_MAIN_R = 15
PIN_MAIN_Y = 16
PIN_MAIN_G = 13
PIN_SIDE_R = 7
PIN_SIDE_Y = 11
PIN_SIDE_G = 12
PIN_BUTTON = 22
YOLO_FLAG_PATH = "/tmp/side_detected.txt"

# timing
MIN_MAIN_GREEN = 5.0
MAIN_YELLOW_TIME = 2.0
ALL_RED_TIME = 1.0
SIDE_PED_TIME = 8.0
SIDE_MIN_GREEN = 4.0
SIDE_MAX_GREEN = 15.0

# read YOLO flag
def read_yolo_car_present():
    try:
        with open(YOLO_FLAG_PATH, "r") as f:
            return f.read().strip() == "1"
    except:
        return False
```

### State transition example

```python
if state == "MAIN_GREEN":
    if (req_pedestrian or req_car) and time_in_state >= MIN_MAIN_GREEN:
        state = "MAIN_YELLOW"
        set_lights(0,1,0, 1,0,0)  # main yellow, side red
```

---

## 5. Results & Demo

> *[Insert video: Full demonstration — Car detected → Pedestrian waits → Crossing allowed]*

### Behavior summary

1. When YOLO detects vehicles on the side street, a car request is latched. The main road completes the minimum green, then transitions to side green. Side green is extended adaptively while vehicles are present (up to a max).
2. Pedestrian button latches a guaranteed pedestrian time; if cars are present, the pedestrian waits until safe.
3. Uses atomic file writes to prevent race conditions between detector and controller.

---

## 6. Key Takeaways

**Skills & techniques demonstrated:**

* Embedded AI on **Jetson Orin Nano** (YOLOv8, ONNX, TensorRT acceleration options)
* Robust **GPIO** and device-tree handling for custom pinmux needs
* Designing deterministic **finite-state machines** for safety-critical timing
* **Atomic inter-process communication** via temp-file swap
* **Hardware prototyping:** LEDs, resistors, button wiring, circuit diagrams

---

## 7. Future Improvements

* Add redundant sensors (ultrasonic/loop) for safety
* Move detection → shared memory or IPC for lower latency
* Add a small pedestrian countdown display
* Improve model accuracy/speed with quantization (INT8 TensorRT)
* Create a polished GitHub repo + documentation and downloadable images/videos

---

### 📁 Repo / Media Placeholders

* **GitHub repo:** [Insert link once uploaded]
* **High-res photos:** [Insert links or assets]
* **Demo video:** [Insert link or embed]
