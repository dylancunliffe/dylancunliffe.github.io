---
layout: post
title: Automotive Sensor Telemetry PCB
subtitle: Designed PCB for EV telemetry, powered by an STM32 MCU to aggregate GPS, speed, and thermal sensor data via UART, I2C, and CAN protocols
cover-img: 
thumbnail-img: assets/img/telemetry-pcb.png
tags: [PCB Design]
author: Dylan Cunliffe
---

# Sea-to-School Forecasting  
### Building a Real-Time Commute Prediction System from West Vancouver to UBC  
*Engineering Portfolio Project — Dylan Cunliffe*

---

## 📍 Overview

**Sea-to-School Forecasting** is a full end-to-end commute prediction system built to estimate travel time from **West Vancouver → UBC** using:

- A custom **ESP32 embedded device** for GPS data collection  
- A Python-based **prediction engine**  
- A **segment-based route model**  
- A pipeline for historical data ingestion and analysis  

The project combines embedded systems, data engineering, and predictive modelling into a polished engineering portfolio piece backed by real commute data.

---

## 🛠 Project Structure

````bash
.
├── data/
│   ├── raw_gps_logs/
│   └── processed_segments/
├── device/
│   ├── main.cpp
│   ├── sd_logging.cpp
│   └── gps_driver.cpp
├── src/
│   ├── prediction/
│   │   ├── predict.py
│   │   ├── segment_model.py
│   │   └── smoothing.py
│   ├── processing/
│   │   ├── parse_logs.py
│   │   └── clean_data.py
│   └── utils/
│       └── haversine.py
├── notebooks/
│   └── analysis.ipynb
├── README.md
└── requirements.txt
````

---

## 🎯 Motivation

The commute from **West Vancouver to UBC** is influenced by:

- Bridge congestion  
- Highway choke points  
- Time-of-day patterns  
- Campus-specific traffic flows  

The objective was to build a system that answers:

> **“If I left right now, how long would my commute take?”**

To accomplish this, I built:

1. A physical embedded device to record commute trajectories  
2. A data pipeline to clean and segment the route  
3. A prediction engine that models traversal durations  
4. A system that can be extended to real-time forecasting  

---

## 🚗 Embedded Device & Firmware

### Hardware  
- ESP32 DevKit  
- u-blox GPS module  
- SD card module  
- Custom wiring harness  

### Features  
- High-frequency GPS logging  
- Buffered SD writing  
- Automatic file rollover  
- Handling for temporary GPS loss  

### Example Firmware Snippet

````cpp
void logPosition() {
    if (gps.location.isValid()) {
        logFile.printf("%lu,%f,%f,%f\n",
            millis(),
            gps.location.lat(),
            gps.location.lng(),
            gps.speed.kmph()
        );
    }
}
````

### 📸 *Insert Photo Placeholder*  
**Photo of the ESP32 + GPS + SD hardware mounted in the vehicle.**

---

## 🧹 Data Processing Pipeline

Raw logs are stored as CSV files on the SD card. The data pipeline:

1. Loads raw GPS logs  
2. Removes invalid points  
3. Smooths jitter  
4. Computes cumulative distance  
5. Detects segment boundaries  
6. Produces per-segment traversal durations  

### Example Segment Extraction Code

````python
def extract_segment_durations(df, segments):
    durations = []
    for seg in segments:
        start, end = seg.bounds
        seg_df = df[(df.dist >= start) & (df.dist < end)]
        durations.append(
            seg_df.timestamp.iloc[-1] - seg_df.timestamp.iloc[0]
        )
    return durations
````

---

## 🧠 Prediction Engine

The route model treats the commute as a series of independent travel segments.

Each segment stores:

- Mean traversal time  
- Variance  
- Smoothed historical average  
- Last recorded duration  

### Example Model

````python
class RouteModel:
    def __init__(self, segment_stats):
        self.stats = segment_stats
    
    def predict_total_time(self):
        return sum(seg.mean for seg in self.stats)
````

### Major Fix  
Originally, each segment prediction **started at time 0**, causing stacking errors.  
The corrected version ensures each segment starts **when the previous one ends**.

---

## 📊 Data Analysis & Results

All analysis was done via Jupyter Notebook.

Suggested visualizations:

- Total commute time vs. date  
- Segment-level traversal time distributions  
- Box plots of morning vs afternoon traffic  
- Route animation using GPS points  

### 📸 *Insert Figure Placeholder*  
**Graph of predicted vs actual commute times.**

---

## 🧭 Route Segmentation

The commute is split into logical segments such as:

- Home → Highway  
- Highway → Lions Gate Bridge  
- Bridge → Downtown  
- Downtown → 4th Ave  
- 4th Ave → UBC  

Each segment boundary corresponds to GPS-detected distance thresholds.

---

## 🏗 Architecture

### System Flow  
1. **ESP32 device logs a commute**  
2. Data is saved to SD as `.csv`  
3. Python pipeline parses + cleans it  
4. Segments are computed  
5. Statistics updated  
6. Prediction engine outputs total commute time  

---

## 🚀 Future Work

- Integrate real-time traffic data APIs  
- Add weather + day-of-week predictors  
- Explore ML models (XGBoost, RNNs, etc.)  
- Build dashboard UI for live predictions  
- Automate segment detection using clustering  
- Improve hardware reliability & smoothing  
- Add cloud storage + long-term trend analysis  

---

## 📂 Repository

Full source code:  
**https://github.com/dylancunliffe/sea-to-school-forecasting**

---

## 📸 Media Placeholders

- *Photo of hardware*  
- *Screenshot of notebook visualizations*  
- *Diagram of route segmentation*  
- *Graph of predicted vs actual travel times*  

---

## 🧾 Closing Notes

This project demonstrates cross-domain engineering:

- Embedded systems  
- Real-world data collection  
- Time-series modelling  
- Route segmentation  
- Predictive analytics  

It forms the basis for a fully deployable real-time commute prediction system.

