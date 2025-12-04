---
layout: post
title: Automotive Sensor Telemetry PCB
subtitle: Designed PCB for EV telemetry, powered by an STM32 MCU to aggregate
GPS, speed, and thermal sensor data via UART, I2C, and CAN protocols
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [books, test]
author: Dylan Cunliffe
---


## Overview

I designed and engineered a robust, mixed-signal telemetry node intended for electric vehicle diagnostics. The system aggregates real-time data from GNSS/GPS, digital thermal sensors, and Hall-effect speed sensors, transmitting packetized data to a central vehicle controller via the CAN Bus.

The project focused on bridging the gap between prototype logic and automotive-grade reliability, with a heavy emphasis on power integrity, signal integrity, and EMI suppression.

[INSERT IMAGE 1: The "Hero Shot"] (Use the 3D Render of your board (image_cfb528.jpg). It looks professional and shows the density of the design immediately.)

## System Architecture

The core of the system is the STM32G0B1 microcontroller. I selected this MCU for its rich peripheral set (specifically FDCAN and multiple USARTs) and low power consumption.

The system is partitioned into three distinct electrical zones:

    High-Noise Power Domain: Input protection and DC-DC conversion.

    Digital Logic Domain: MCU, Crystal Oscillator, and Status LEDs.

    RF & Sensor Domain: GPS/GNSS path and sensitive sensor interfaces.

[INSERT IMAGE 2: Schematic Capture] (Use a clean screenshot of your final schematic (image_0acd7f.jpg) to show the logical complexity.)

## Key Engineering Challenges
**1. Automotive Power Design & Protection**

A standard vehicle 12V rail is quite noisy. To ensure the 3.3V logic remained stable during voltage transients:

    DC-DC Buck Conversion: Replaced inefficient linear regulators with a Switching Regulator topology to maximize efficiency and minimize heat.

    Protection Circuitry: Implemented a Schottky diode for reverse polarity protection (preventing damage during battery installation) and input bulk capacitance to filter alternator ripple.

    Loop Optimization: In the layout, I minimized the surface area of the high-current switching loop (Input Cap → Buck → Diode) to reduce radiated EMI.

**2. High-Speed Differential Signaling (CAN Bus)**

The communication backbone relies on a TJA1042 Transceiver. To ensure data integrity over long cable runs:

    Implemented Split Termination (60Ω + 60Ω + 4.7nF) to improve common-mode noise rejection.

    Added TVS Diodes (PESD1CAN) immediately at the connector entry to shunt high-voltage ESD spikes before they reach the transceiver.

    Differential Routing: Routed CAN_H and CAN_L as a tightly coupled differential pair with length matching to ensure phase synchronization and impedance control.

[INSERT IMAGE 3: Layout Zoom - CAN Bus] (Take a zoomed-in screenshot of your PCB layout (bottom left corner) showing the Connector, TVS Diodes, Transceiver, and the parallel traces connecting them. This proves you know how to route differential pairs.)
**3. RF & Signal Integrity**

For the GNSS (GPS) module, the signal path from the module to the SMA connector required careful attention. I routed the RF trace away from the noisy switching power supply and ensured a continuous ground plane reference on the layer beneath to maintain characteristic impedance.

I also utilized System Partitioning for the thermal sensors. Instead of relying solely on onboard sensors, I designed external I2C interfaces with local power and pull-ups, allowing the unit to monitor remote vehicle components (like battery cells) rather than just PCB ambient temperature.

[INSERT IMAGE 4: Layout Zoom - Zoning] (Take a screenshot of the whole PCB layout, but draw colored boxes over it in Paint/Photoshop. Red box over the Power section, Blue box over the MCU, Green box over the GPS. Label this "PCB Zoning Strategy for EMI Reduction".)
Result

The final design is a compact, 2-layer PCB routed in Altium Designer. It demonstrates a "Safety First" hardware philosophy, ensuring that the board not only functions on a bench but survives the harsh electrical environment of an automotive chassis.
Tools & Technologies

    EDA: Altium Designer

    MCU: STM32G0B1 (ARM Cortex-M0+)

    Protocols: CAN (ISO 11898), UART, I2C

    Hardware Skills: Buck Converters, Differential Pair Routing, Impedance Control, DFM (Design for Manufacturing).

[INSERT IMAGE 5: GitHub/Documentation Link] (Optional: If you upload the Gerber files or a PDF of the schematic to GitHub, put a link or a screenshot of the repo here.)
