---
layout: post
title: Barometric Altimetry
---

What does this platform offer?
---------------------------------
While a high altitude balloon can be constructed with off the shelf parts, Floatini combines all these basic components into a lightweight, self-contained unit. This increases reliability significantly by eliminating wires. Floatini is also preprogrammed with mission proven firmware which I've made open source. Settings and telemetry data are dynamically configurable via CLI or radio link, and you can view live telemetry data in your browser (no internet connection required!). If you want to attach more sensors,the firmware is easily modified to include additional datapoints. Floatini provides a reliable, tested platform to launch your own high altitude balloon without breaking the bank. 

How does it work?
--------------------------------
Launching your own high altitude balloon with Floatini is designed to be as simple as possible. All you need to successfully track your balloon is Floatini V2 Nano plus a base station, which can be a T-Beam board or a second Floatini. Floatini's communication protocol also allows for redundant ground receivers in case one fails. 

### Hardware Features:
- Lightweight: 11g
- Optional balloon release mechanism
- Uses AA Lithium battery as power source
- 915MHz long range LoRa transceiver with TCXO
- High performance Ublox MAX-M10 GPS receiver
- Integrated omni-directional GPS antenna
- DPS310 barometric pressure sensor
- 3 axis accelerometer detects descent and landing
- QWIIC connector for adding external sensors
- Piezo buzzer to make post-landing location easier
- 4 temperature sensors
- 8 hours of continuous data logging (1sec interval)    
- Battery voltage and current reporting
- Firmware updates via micro USB connector
- U.fl LoRa antenna connector
- 4 status LEDs make troubleshooting easier
- SWD pads for programming/debugging

### Software Features
- Pre-launch configuration via CLI or browser
- Configure on the fly via LoRa link
- View live telemetry data in browser (no internet connection)
- Support for external QWIIC sensors (work in progress)
- Adaptive telemetry protocol reduces risk of losing payload
- Flexible ground station setup
- Supports redundant ground station receivers
- Configuration settings persist across reboot

