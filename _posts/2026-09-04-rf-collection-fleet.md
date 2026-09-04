---
layout: post
title: Ambient-RF Data Collection Fleet
subtitle: Hardware, Firmware, and Provisioning for 20 Autonomous FM Sensing Units
thumbnail-img: /assets/img/eNS-tumb.png
tags: [Altium, ESP32, Si4732, RF, Embedded Firmware, PCB Design]
---

## Overview

Research into ambient-RF sensing needs one thing before it needs anything else: data, from many physical locations, recorded consistently, over months. That requirement is deceptively hard to satisfy. A single benchtop receiver tells you about one room. Renting twenty commercial SDR nodes is prohibitively expensive. What the work actually needed was a fleet — cheap enough to build lots of, reliable enough to leave unattended in strangers' homes, and simple enough that a non-technical volunteer could set one up without a phone call.

I designed and built that fleet: a custom Si4732 receiver PCB, ESP32 firmware with offline-first logging, a captive-portal provisioning flow aimed at people who have never heard of an ESP32, and a cloud ingest path for the data that comes back.

This post covers the collection rig end to end — the hardware, the bring-up problems that shaped it, the firmware architecture, and what it took to make twenty of them manufacturable. The signal-processing work that consumes this data is a separate story.

Key outcomes:

- **Custom two-layer receiver PCB** in Altium, built around the Si4732 FM tuner with an ESP32 carrier, DS3231 RTC, microSD, and an SMA antenna front end
- **Offline-first firmware** treating the SD card as the source of truth and WiFi as opportunistic, so a unit with no network is fully functional rather than degraded
- **Captive-portal provisioning** designed for non-technical placers, then revised against real user testing that found four failures I would not have predicted
- **Production diagnostics** giving a single `BOARD: PASS/FAIL` verdict, so twenty assembled boards can be verified without reading firmware

### System Flow

1. A volunteer plugs the unit in and connects a phone to its WiFi access point
2. The captive portal collects network credentials (optional), coordinates, and a location label
3. The unit records a placement record to SD immediately, before any network activity
4. Periodic scanning bursts sweep the FM broadcast band, capturing per-channel signal metrics
5. Each reading is CRC-checked and appended to a segmented, size-bounded log on the SD card
6. When WiFi is available, records sync to a cloud database; when it isn't, the card is collected by hand

---

## Why a Custom Board

The first working receiver was a breadboard: an ESP32 dev board, an Si4732 breakout, a microSD module, and an RTC module, joined by jumper wires. It proved the concept and it was never going to leave the bench.

Three problems made a PCB non-negotiable:

**Reproducibility.** Twenty hand-wired units means twenty different sets of contact resistances, wire lengths, and stray capacitances. When a unit in the field produces anomalous data, I need to be confident the anomaly is in the environment, not in that particular unit's jumper wires.

**Reliability over months.** A breadboard connection that works on the bench fails silently after a season of thermal cycling in someone's basement. Solder joints do not.

**Assembly cost.** Hand-building twenty units is roughly a week of tedious, error-prone work. A panel of assembled boards is a purchase order.

The board is deliberately conservative: two layers, a solid ground pour, through-hole headers for the ESP32 DevKitC so a failed module can be swapped, and SMT for everything else so it can be machine-assembled.

> ![eNS Altium Render](/assets/img/eNS-3d.png)
*Image: Altium 3D render of the assembled board*

---

## The Si4732 Bring-Up

Most of the schedule risk in this project lived in one chip. The Si4732 is an excellent FM receiver and a genuinely unforgiving part to bring up, and three separate problems each cost real time. They are worth writing down because none of them were obvious from the datasheet alone.

### Bus Mode Will Not Latch Itself

The Si4732 selects its control interface — 2-wire, 3-wire, or SPI — by sampling GPO1 and GPO2 at the moment its reset line goes high. The chip has internal pull resistors intended to make this work without external components.

They are approximately 1 MΩ, and in practice they are far too weak to establish a reliable level against board capacitance and leakage. The symptom is the worst kind: the chip is *sometimes* on the I2C bus. It enumerated perhaps one boot in four, which reads like a marginal solder joint rather than a logic problem, and sent me looking in the wrong place for a while.

The fix is to actively drive the straps — GPO1 high, GPO2 low — across the reset rising edge, then release both to high-impedance immediately afterward, because those same pins become chip *outputs* once the interface is selected. Holding them would be a driver fight. That handoff is a few lines of firmware and it took a full debugging session to arrive at.

### Reading a Response Destroys It

The second problem produced beautifully clean, entirely empty data.

Every command returned correct protocol framing — clean ACK, correct clear-to-send bit, error flag never spuriously set — with a response payload of all zeros. That combination is diagnostic in itself: damaged silicon does not usually keep perfect bus manners while failing only the data. It looked like reading too early, and it was.

The obvious correction is to poll a status command until the chip reports ready, then read the real response. That made it *worse*, in a revealing way: one trailing byte began changing between otherwise identical runs while every other byte stayed zero.

The status poll is a different command. Issuing it while a response from an earlier command is still pending appears to invalidate that pending response — the poll was destroying the very data it was waiting for.

What works is to poll by re-reading the *same* pending response repeatedly, checking its own status byte, and never issuing an unrelated command in between. Obvious in hindsight; not obvious at 1 AM.

### A 32.768 kHz Crystal Will Not Start on a Breadboard

The third problem was the most instructive, because the datasheet had already answered it and I had not read closely enough.

With the reference crystal fitted, tuning commands completed cleanly and then reported a read-back frequency of zero — the phase-locked loop had nothing to lock to. Occasionally, one tune in many would succeed.

The reference characteristics table specifies a maximum of **3.5 pF** of board capacitance. Breadboard row-to-row stray capacitance alone is typically 2–5 pF before you count jumper wires. A 32.768 kHz tuning-fork crystal is also the most startup-sensitive part in the circuit: very high equivalent series resistance, very low drive level, very high Q. Excess stray capacitance kills loop gain and it simply will not reliably start.

The workaround sidesteps the oscillator entirely — drive the reference input from an ESP32 LEDC hardware timer output. Which then exposed a fourth problem: on that particular hand-soldered unit, the primary reference pin was dead. A scope confirmed the clock was present at the pin; the chip ignored it. Rerouting the same signal to the chip's alternate clock input via a configuration bit fixed it immediately.

That history is why the production board provisions **both** clock fallbacks as always-populated zero-ohm jumpers, and why the firmware exposes reference-clock selection as a runtime, NVS-persisted setting rather than a compile-time constant. If one assembled board out of twenty needs a different reference path than the rest, that is a serial command, not a reflash and not a soldering iron.

> ![Clock Solution](/assets/img/eNS-clocks.png)
*Image: schematic capture of the backup clock solutions*

---

## Hardware Design

### Pin Budget

The ESP32's GPIO constraints drove much of the layout:

- **ADC2 is unusable while WiFi is active**, which puts the thermistor on an ADC1 pin
- **Strapping pins** (0, 2, 5, 12, 15) determine boot mode and cannot be casually reused; anything pulled the wrong way at power-on bricks the boot
- The microSD runs on the **default VSPI pins**, which happen to land in exactly the right roles — a small piece of luck that avoided remapping
- The Si4732 and DS3231 share one **I2C bus** at distinct addresses, so the two peripherals cost two pins total

### Antenna Front End

The antenna is a 770 mm telescopic whip on an SMA connector — a quarter wave at 98 MHz is 765 mm, so the physical antenna is close to resonant across the band without additional electrical lengthening.

The matching network is a series coupling capacitor and a shunt inductor. Rather than guess the inductor value, the board is designed to have it selected on the bench: the selection criterion is the flattest response across the whole broadcast band rather than peak sensitivity at any single frequency, because this application cares about consistent relative measurements across channels far more than it cares about maximum absolute sensitivity on one.

### Layout Notes

- Solid ground pour on the bottom layer, power pour on top
- Clearance maintained under the ESP32 module's own PCB antenna — a ground pour under a 2.4 GHz antenna detunes it, and the near-field boundary is roughly 20 mm at that frequency
- ESD protection on the exposed antenna port
- A user button and status LED, both reachable through the enclosure

> ![ENS Layout](/assets/img/eNS-Layout.png)
*Image: PCB layout*

---

## Firmware Architecture

### Offline-First Is Not a Fallback

The single most important architectural decision: **the SD card is the source of truth and the network is opportunistic.**

This is not a graceful-degradation story. A unit with no WiFi at all is a fully supported, permanent operating mode — not a broken unit. Some volunteers have no network to offer, some have a network they would rather not share, and some have one that works intermittently. All three are normal. Data reaches the database either over WiFi or by physically collecting the card, and the firmware treats those as equally valid paths.

Concretely, that means the placement record is written to the card *before* any network activity is attempted, and a scanning burst never waits on a network operation.

### Bounded, Recoverable Storage

An unattended logger that runs for months has two failure modes to design against: filling the card, and corrupting the log on power loss.

**Filling the card** is handled by segmented circular storage. The log is split into fixed-size segments; when the card approaches full, the oldest segment is deleted. Storage is bounded by construction — the unit never fills, and it degrades by losing the oldest data rather than by stopping.

**Corruption** is handled by per-record CRC-16 and a recovery scan at boot. Power can be pulled at any instant, including mid-write. On the next boot the firmware walks the active segment, validates each record, and reports how many it recovered. A partial trailing record is discarded rather than trusted.

### Staying Alive Unattended

Nobody is going to power-cycle these. The firmware tracks its own reset reason and counts unstable boots; repeated early crashes escalate into a safe mode that keeps the unit reachable rather than letting it crash-loop indefinitely. A hardware watchdog covers hangs. A thermal guard with hysteresis pauses scanning if the board runs hot, and logs both the pause and the resume as durable records rather than only printing them to a serial port nobody is watching.

The hysteresis matters more than it sounds: without it, a temperature sitting exactly at the threshold produces continuous pause/resume chatter and a log full of noise.

---

## Provisioning for People Who Are Not Engineers

The unit is set up by whoever agreed to host it. They will do it once, on their phone, and they did not sign up for a technical experience. The flow has to work in about two minutes with no instructions.

The unit boots as a WiFi access point. Connecting to it opens a captive portal: choose a network, enter coordinates and a label for the location, review, confirm.

Then I had someone who had never seen the project run through it while I watched, and it found four things I would not have caught:

**The portal did not auto-open on their Android.** Captive-portal detection is inconsistent across Android versions and manufacturers, and there is no firmware fix that covers all of them. The mitigation is a QR code and a printed fallback address on the setup card that ships with each unit.

**They did not know which button to press.** The menu offered several options with equal visual weight. Colour-coding the primary action fixed it.

**Their keyboard had no minus sign.** They entered a positive longitude. Every site in this project is in North America, where longitude is always negative — so the firmware now corrects a positive longitude automatically. But it *states* the correction on the review screen rather than doing it silently, because a genuinely transposed digit should still be visible before the placer confirms. Silent correction of user input is how you turn one class of error into a harder one.

**The location label was too coarse.** Asked to name the location, they entered the name of their city. That is useless for telling two units apart. Rather than adding more required fields — friction for someone doing a stranger a favour — a single-word entry now triggers a soft, non-blocking suggestion to add a landmark.

None of these are bugs in the engineering sense. All four would have degraded the dataset.

*[Image: portal screenshots — setup page and review page]*

---

## Data Pipeline

Records reach a hosted Postgres database through a REST endpoint, with row-level security restricting the embedded credential to insert-only on every table. The key that ships inside twenty units that live in other people's homes should be able to add data and nothing else — it cannot read the corpus back, and it cannot modify or delete anything.

Every record carries a per-unit monotonic sequence number, which makes inserts idempotent: a sync interrupted halfway can safely retry the whole batch without creating duplicates. For units with no network, a separate import path walks the raw log segments off a collected SD card, performs the same validation and recovery, and produces the same table structure.

Fleet health — last contact, record counts, placement metadata, per-unit notes — is visible through an internal dashboard, so a unit that quietly stopped reporting three weeks ago is noticed in three weeks rather than at collection time.

---

## Making Twenty of Them

### Production Diagnostics

Twenty assembled boards need a fast, unambiguous answer to "is this board good?" — from someone who should not have to read C++ to find out.

The firmware carries a serial diagnostic mode that is **always available, not a separate build**. That is deliberate: a compile-time test mode becomes useless the moment a unit is sealed in an enclosure and shipped, which is exactly when field service needs it. The cost is one buffer check per loop.

A single command runs every automatic test and ends in one `BOARD: PASS/FAIL` line. Each failure names the command that explains it, and each individual test prints what to physically check — not just what failed. A failing RTC test, for example, distinguishes "this module is not on the bus" from "the time is readable but the oscillator-stop flag is set," which on a first power-up is expected behaviour rather than a fault, and is exactly the kind of thing that gets a good board scrapped by someone who does not know that.

One test is worth describing because of what it *cannot* do. The bus-mode strap check holds the receiver in reset — those pins are chip outputs otherwise, so testing them live would measure the firmware's own drive fighting the chip's — and checks each strap against an internal pull-up and pull-down to catch a short to either rail, plus a cross-check that the two nets are not bridged to each other. It cannot see past the series resistors to the chip's own pins, because the microcontroller's internal pull dominates the chip's much weaker one; an open joint at the far end reads identically to a good one.

That limitation is printed in the test's own output. A board test that overstates its coverage is worse than no test, because it converts "unknown" into "believed good."

### A Lesson in Verifying Your Own Verification

While reviewing the assembly files, I checked the user LED's tune-across-the-band test and found that the existing receiver test tuned a single frequency and read signal quality — but never verified the chip's reported read-back frequency.

That is precisely the failure documented above: the tune completes cleanly and reports zero. The test written to catch a dead reference clock could not actually catch it. It now tunes several channels across the band and asserts the read-back tracks each request, which is the check that has a chance of failing when something is genuinely wrong.

### Assembly and Enclosure

The boards are machine-assembled. Reviewing the fab's polarity-confirmation renders caught a reversed LED footprint before the run — worth cross-checking against the actual manufacturer datasheet rather than trusting either the CAD library's silkscreen or a rendered preview, since in that case the two disagreed and the library was wrong.

The enclosure is a two-part 3D-printed shell with a ventilated panel, printed in PLA with the visible faces oriented against the build plate.

> ![eNS Unit](/assets/img/eNS-Render-1.png)
*Image: Final production model*

> ![eNS unit without lid](/assets/img/eNS-2.png)
*Image: Final production unit without lid

---

## Future Revisions

- **Segment rotation is not yet tested end to end.** At the current segment size and record rate, a segment takes roughly fifty hours of continuous running to roll over, and full circular overwrite takes weeks — so it cannot be verified by simply letting a unit run overnight. It needs either a build with a reduced segment size or a pre-populated card that fakes a near-full state.
- **Better reference-clock validation.** The frequency-offset field is currently unreliable on the external-clock path used during bring-up. Whether the production crystal fixes it can only be answered on assembled hardware, and there is no automatic bound on it yet.
- **Power measurement.** These run on USB power today. Battery operation would open up placements with no convenient outlet, but that requires a real power budget I have not yet measured.
- **Remote firmware update.** Currently a unit needs to be physically reached to update. For twenty units in twenty homes, over-the-air updates would pay for themselves the first time a bug ships.
