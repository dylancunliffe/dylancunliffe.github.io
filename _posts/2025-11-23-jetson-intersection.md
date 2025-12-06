---
layout: post
title: Edge AI Intersection Control System
subtitle: A real-time, vision-enhanced traffic controller built on the Jetson Orin Nano
thumbnail-img: assets/img/jetson-orin-nano-super-developer-kit-workloads-fg-i3of3-d.jpg
tags: [AI, Machine Learning, Embedded Programming]
author: Dylan Cunliffe
---

## 🚦 Overview

Modern urban intersections are chaotic environments. My project aims to demonstrate how low-cost embedded systems can contribute to **safer, more intelligent intersections**.

I created a **real time intersection awareness system** using the NVIDIA Jetson Orin Nano, a live camera feed, YOLOv8 object detection, a pedestrian-style button, and a set of high-brightness LEDs acting as “awareness indicators.” When the system detects a vehicle approaching from the side, the LEDs activate to warn a crossing pedestrian. The user can also press a button to simulate “requesting to cross.”

This prototype shows how computer vision and embedded hardware can work together to support **safer, more informed crossing decisions**, especially in complex or low-visibility areas.

For this project, I designed and built a complete **smart intersection simulation** using:

- A **Jetson Orin Nano Super Dev Kit**
- **YOLOv8 (ONNX)** real-time object detection
- **Push-button input** for pedestrian crossing
- **LED traffic-light modules**
- A **Python controller** that synchronizes everything using shared files and atomic writes

---

## Background & Motivation

### Traditional Traffic Detection  
Most legacy traffic intersections rely on **inductive loop sensors** embedded in the road. These work by detecting disturbances in a magnetic field when a vehicle sits above the loop. While reliable, they come with significant downsides:

- **Expensive installation** — requires cutting into pavement  
- **Difficult maintenance** — pavement cracks, weather damage, and resurfacing often break loops  
- **Single-purpose** — they detect only vehicle presence, not type, speed, or configuration  
- **No pedestrian awareness** — separate hardware is required  

Some systems use **radar**, **microwave sensors**, or **infrared**, but these add cost and still lack visual data.

### Why Vision-Based Detection?
Computer vision offers several advantages:

- **Non-invasive** — no trenching or installing in-road hardware  
- **Adaptable** — detect cars, bikes, pedestrians, buses, or anything a model is trained for  
- **Upgradeable** — improve via model updates instead of hardware replacements  
- **Cheaper for prototyping** — a single camera + embedded board replaces multiple sensors  

My motivation for this project came from wanting to re-create a complete AI-driven intersection system using **only low-cost hardware and open-source tools**—something that simulates real-world infrastructure challenges but is hands-on and understandable at a student level.

### Why the Jetson Orin Nano?

NVIDIA’s Jetson Orin Nano is a compact, power-efficient edge AI computer. It’s powerful enough to run **real-time YOLO detection** while simultaneously executing hardware control logic—making it ideal for embedded robotics, smart devices, and in this case, a vision-driven intersection controller.

---

## Project Setup

work in progress
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

### Modules  
1. **Vision Module (`yolo_detect.py`)**  
   - Reads camera frames  
   - Runs YOLOv8 inference  
   - Detects vehicles and updates `side_detected.txt`  

2. **Intersection Controller (`main_controller.py`)**  
   - Reads the shared detection file  
   - Monitors a pedestrian button  
   - Drives LEDs indicating whether it’s safe to cross  

File writing prevents corrupted reads and ensures consistent hardware behavior.

- **YOLOv8 Object Detection Script**
  Continuously writes `"1"` or `"0"` to `/tmp/side_detected.txt` using file writes.

- **Button Reader**
  Reports the pedestrian button’s state.

- **Main Controller**
  Reads both inputs and decides whether to allow:
  - Normal vehicle flow
  - Pedestrian walk phase
  - Delayed crossing if vehicles are present

---

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

### Components Used  
- Jetson Orin Nano Super Dev Kit  
- USB or CSI camera  
- High-brightness LEDs  
- 330–1kΩ resistors  
- Active-high pushbutton  
- Jumper wires + breadboard  

### Wiring  
- Button wired **active high**:  
  - One side → **3.3V**  
  - Other side → **GPIO input pin**  
  - Same pin → **10kΩ pulldown** to GND  
- LEDs driven through current-limiting resistors from GPIO outputs  

This ensures clean, stable digital readings with no floating.

---

> *[Insert circuit diagram/photo: Pedestrian Push-Button Wiring]*

## 3. Pedestrian Button Input

### Code — Button Reader

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

![State diagram](/assets/img/Gemini_Generated_Image_eoqri3eoqri3eoqr.png)

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

1. When YOLO detects vehicles on the side street, a car request is created. The main road completes the minimum green, then transitions to side green. Side green is extended adaptively while vehicles are present (up to a max).
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

* Add more cameras for a full intersection system
* Add redundant sensors (ultrasonic/loop) for safety
* Move detection → shared memory or IPC for lower latency
* Add a small pedestrian countdown display
* Improve model accuracy/speed with quantization (INT8 TensorRT)
* Create a polished GitHub repo + documentation and downloadable images/videos

---

### 📁 Repo

* **GitHub repo:** [Insert link once uploaded]
