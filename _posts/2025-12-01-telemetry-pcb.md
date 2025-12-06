---
layout: post
title: Automotive Sensor Telemetry PCB
subtitle: Designed PCB for EV telemetry, powered by an STM32 MCU to aggregate GPS, speed, and thermal sensor data via UART, I2C, and CAN protocols
thumbnail-img: assets/img/Screenshot 2025-12-05 184139.png
tags: [PCB Design]
author: Dylan Cunliffe
---

### **Overview**
I designed and engineered a robust, mixed-signal telemetry node intended for automotive diagnostics. The system aggregates real-time data from GNSS/GPS, digital and analog thermal sensors, and Hall-effect speed sensors, transmitting packetized data to a central vehicle controller via the **CAN Bus**.

![3d render of PCB](assets/img/Screenshot 2025-12-05 184139.png)
*Figure 1: 3D Render of the Telemetry Unit designed in Altium Designer.*

---

### **System Architecture**
The core of the system is the **STM32G0B1 (ARM Cortex-M0+)** microcontroller. I selected this MCU for its rich peripheral set (specifically FDCAN and multiple USARTs) and low power consumption.

The system is partitioned into three distinct electrical zones to minimize noise coupling:
1.  **High-Noise Power Area:** Input protection and DC-DC conversion.
2.  **Digital Logic Area:** MCU, Crystal Oscillator, and Status LEDs.
3.  **RF & Sensor Area:** GPS/GNSS path and sensitive sensor interfaces.

![Schematic Architecture](assets/img/Screenshot 2025-12-05 184651.png)
*Figure 2: Complete System Schematic showing logical partitioning.*

---

### **Key Engineering Challenges**

#### **1. Automotive Power Design & Protection**
A standard vehicle 12V rail is notoriously noisy. To ensure the 3.3V logic remained stable during voltage transients:
* **DC-DC Buck Conversion:** Replaced inefficient linear regulators with a Switching Regulator (Buck) topology to maximize efficiency and minimize heat.
* **Protection Circuitry:** Implemented a **Schottky diode** for reverse polarity protection (preventing damage during battery installation) and input bulk capacitance to filter alternator ripple.
* **Loop Optimization:** In the layout, I minimized the surface area of the high-current switching loop (Input Cap $\rightarrow$ Buck $\rightarrow$ Diode) to reduce radiated EMI.

#### **2. High-Speed Differential Signaling (CAN Bus)**
The communication backbone relies on a **TJA1042 Transceiver**. To ensure data integrity over long cable runs:
* **Split Termination:** Implemented a split termination network ($60\Omega$ + $60\Omega$ + 4.7nF) to improve common-mode noise rejection.
* **Transient Protection:** Added **TVS Diodes (PESD1CAN)** immediately at the connector entry to shunt high-voltage ESD spikes before they reach the transceiver.
* **Differential Routing:** Routed `CAN_H` and `CAN_L` as a tightly coupled differential pair with length matching to ensure phase synchronization and impedance control.

![CAN Bus Routing](path/to/your/zoom_image_can_bus.jpg)
*Figure 3: Differential Pair routing of the CAN Bus interface with TVS protection.*

#### **3. RF & Signal Integrity**
For the GNSS (GPS) module, the signal path from the module to the SMA connector required careful attention. I routed the RF trace away from the noisy switching power supply and ensured a continuous ground plane reference on the layer beneath to maintain characteristic impedance.

I also utilized **System Partitioning** for the thermal sensors. Instead of relying solely on onboard sensors, I designed external I2C interfaces with local power and pull-ups, allowing the unit to monitor remote vehicle components (like battery cells) rather than just PCB ambient temperature.

![PCB Zoning Strategy](assets/img/Gemini_Generated_Image_v5elc5v5elc5v5el.png)
*Figure 4: PCB Layout highlighting the strict zoning of Analog, Digital, and Power domains.*

---

### **Result**
The final design is a compact, 2-layer PCB routed in **Altium Designer**. It demonstrates a "Safety First" hardware philosophy, ensuring that the board not only functions on a bench but survives the harsh electrical environment of an automotive chassis.

### **Tools & Technologies**
* **EDA:** Altium Designer
* **MCU:** STM32G0B1 (ARM Cortex-M0+)
* **Protocols:** CAN (ISO 11898), UART, I2C
* **Hardware Skills:** Buck Converters, Differential Pair Routing, Impedance Control, DFM (Design for Manufacturing).
