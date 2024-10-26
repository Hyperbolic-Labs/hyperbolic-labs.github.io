---
layout: post
title: V2 Hardware - GPS
---

Ublox still makes the most reliable GPS receivers on the market, and MAX-M10S is a big improvement over the older MAX-M8 chip. Power consumption is reduced significantly and now hasa builtin LNA for better passive antenna performance. I've always been impressed with how well this receiver works so it's a no brainer for me. Since GPS receivers are sensitive to high frequency harmonics, I added series resistors to the UART and PPS pins which limit the rise/fall times. I also kept the W3011 passive antenna because it's always served me well in past projects.






vinp = R2 (vdd-vlow)/(R1+R2+Rsw) Measure op amp output voltage when lower vref is bat-, record reading. This value is PGA offset voltage times PGA gain error. offset voltage sum of op amp (internal) os plus error in vref voltage divider. 
We don't care if Rsw changes with temperature, as it only manifests as a Vos error in the end. calibration requirements:- two ground-referenced voltages needed (for offset and PGA gain cal)- both ref voltages must be >0V
- PGA in- can be muxed to groundvdd measurement voltage:    (3.0/3.2768)*1.4 = 1.28173828125upper, lower: 0.030000, 0.012000Errors, 1: 0.23%, 2: 0.33%, 3: 0.47%, 4: 0.32%, total: 0.03%R1: 232000.000000, R2: 169000.000000, R3: 2430.000000, R4: 1620.000000R1: 0.572769, R2: 0.417232, R3: 0.005999, R4: 0.004000

Watchdog Coprocessor
After much consideration, I decided to include a separate MCU to this design that acts primarily as a analog front end and watchdog for the RAK3172 LoRa module. This is in part due to the limited I/O ports of the LoRa module and also for more design flexibility. One microcontroller that caught my eye is the MSPM0L130x series from Texas Instruments. It's a Cortex M0 processor with lots of builtin analog peripherals that eliminate the need for a lot of discrete components, and can be found for less than $0.70 in quantity. 

Current sensor design
Having an accurate measurement of battery current provides another valuable datapoint that I'd like this design to have. The most cost effective way of sensing current in this instance is to incorporate the builtin PGA of the MSP. Using the internal PGA instead of an external op amp has some real accuracy benefits that matter in this case. While the on-chip gain resistors may not have the highest initial accuracy, they have excellent temperature drift characteristics. 




Analog Frontend
Due to the LoRa module's limited GPIOs, I opted for an inexpensive ADS1115 discrete I2C ADC. This chip is super handy because it integrates a builtin PGA, eliminating the need for a current sense amplifier for measuring battery current. 3 channels are dedicated for monitoring the supply voltages, Vbat, Vsys, and Vdd. The 4th channel is used to measure battery input current.


Floatini Nano V2
----------------------------------------
Having learned from some of my mistakes with V1, I'm now getting started on Floatini V2 hardware. The main goals of this version are power reduction, easier firmware development, and lower cost. This necessitates a complete redesign unfortunately. To keep cost and weight low, I'm going to strip out all the non-critical components from V1, like the litany of LDOs and passive filters. DFU-capable fimware updates over USB are something I'd like to keep if possible. It just makes the firmware development setup much simpler since you don't need a dedicated programmer. Otherwise most of the main features will remain, with the exception of WiFi/BT and integrated camera. I also planned on submitting this design for JLCPCB's board assembly service, which means selecting parts from their online catalog. In subsequent posts I'll go over the hardware considerations for each of the major components as I progress.


Miscellaneous
I kept the SX1262 radio module from NiceRF from the previous design, mostly because it includes a TCXO. LoRa modules that don't have a TCXO are typically forced to operate at higher bandwidths because of temperature-induced frequency drift. This is undesirable because higher bandwidths severely limit the maximum transmission range. Given the complete failure of the pcb trace cutdown mechanism of V1, I had to come up with a completely different approach that had a better chance of working. Perusing Aliexpress, I discovered some tiny ceramic heating elements that seemed like they would work perfectly. They're made of a small coil of nichrome wire encased in a hollow ceramic core. I plan on feeding the end of the balloon tether through center of this heating core, which is held in place with a drop of candle wax. When power is sent through the heater core, the element softens the wax to let the string pull out, severing connection to the balloon. My main concern with this approach is whether the battery will be able to supply enough power to melt the wax at low temperatures. I will probably experiment with insulation techniques around the heating core so the element doesn't lose too much heat to the surrounding air. Element power is enabled via a low side N channel mosfet which is driven by the MCU. After landing, I anticipate degraded tracker GPS accuracy if it doesn't land upright. I provisioned two MCU pins to drive a piezo buzzer to make recovery easier. These pins will be driven out of phase so the piezo element sees 6.6V peak to peak. For development and debugging purposes there are 4 green LEDs which provide indication of radio transmit/receive status, GPS fix, and processor usage. I also added a micro slide switch as an input to the MCU, which allows software-configured functionality. For my purposes, I will probably assign this switch to set the current tracker mode: in one position, the tracker is in 'mission' mode, and continuously records and transmits data. In the opposite position, the tracker is configured to be in 'debug' mode, and doesn't record telemetry data.

LoRa Radio
All communication to the tracker happens via a SX1262 radio transceiver module. I chose NiceRF module since they offer a 16x16mm module with TCXO, which has a huge impact on sensitivity during extreme temperature changes. The sensitivity of the LoRa receiver is inversely proportional to the bandwidth selected, and choosing a narrower bandwidth reduces the peak transmit current, improving battery life.