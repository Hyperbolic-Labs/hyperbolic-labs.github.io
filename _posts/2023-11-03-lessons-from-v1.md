---
layout: post
title: Lessons from V1
---

The first attempt at a lightweight LoRa tracker was mostly a success, but there are a few things I'm going to change in V2. Most are fairly minor, but in the end I think all of them combined will make a big difference in the performance of V2.

### Processor
Floatini V1 makes use of two ESP32S3 microcontrollers, one to handle the radio and sensors and the second for managing the camera. While the S3 variant supports DFU, one mistake was that I didn't break out the UART pins for the secondary camera processor, so flashing the bootloader was impossible. I never could get it working, however the main processor was fully functional. The ESP32S3 has a lot going for it: plenty of storage and RAM, dual core, and extremely cost effective for what it offers. However due to its complexity, the ESP32 architecture is highly coupled to the HAL, and it feels unnecessarily bloated for this application. The ESP32S3 is power hungry as well, and the sleep modes are not very flexible. Even with the processor speed reduced to 80MHz and WiFi turned off, it uses around 40mA. The internal ADC peripheral is also pretty terrible, even for a 12bit ADC. It has significant offset, non-linearity, and lots of missing codes. While any one of these reasons isn't enough to change my mind, in combination I decided to abandon the ESP32 architecture for this project.

### Boost Converter
The boost converter chip I had chosen (MCP1640) gave me lots of issues in this design. When the battery voltage was <2.5V it had serious trouble keeping a steady 3.3V output, even with a modest current draw of 80mA. According to the datasheet, it is specified to operate down to an input voltage of 0.35V, so it was nowhere near the device limit. When slowly decreasing the input voltage below 2.5V, the current draw from the input would spike to 300-400mA, indicating a massive drop in efficiency. I'm really not sure what the root cause is, but it's possible that PCB layout had a detrimental effect. However it seems that others have had a similar experience with this chip also, so in the future I will avoid using this one.

### Power Supply
In an attempt to reduce EMI/RF noise for the GPS and LoRa transceiver, I used an LDO for each power domain of the circuit. In retrospect, probably lots of unnecessary cost and board space was wasted with this approach, and the added design complexity meant I needed a 6 layer PCB. A single LDO following the boost stage would probably have been sufficient for eliminating most of the boost converter noise. Another complicating factor was the LDO power-on sequence. By mistake, I discovered that driving the interrupt line from the barometer high, caused current to flow through the protection diodes of the sensor. In only a few seconds, the sensor was hot enough to burn my finger. Somehow it still survived, and it worked perfectly with the correct gpio initialization.

### Battery Holder
While not technically an issue, another item I would change is the battery/battery connector. The CR123A battery connector made up more than half of the total weight (not including battery). After some more  research I found there are alternate clip-style connectors that are significantly lighter. I'll use these in the second version instead.

### Cutdown Mechanism
The cutdown mechanism consisted of a transistor which turns on an SMD heating resistor, which was supposed to melt through the cord attaching the payload to the balloon. In practice, the PCB conducted away too much heat for this method to be a viable option, especially at low temperatures. 

### Status LEDs
This version used two RGB leds for status indication, one for the main processor, and one for the camera processor. Separating the status indication into several single-color LEDs would have been easier to read.  It's suprisingly difficult to discern colors with the less-than-ideal color mixing of 0606 LEDs. 

### Antenna
The integrated 915MHz antenna worked well enough, but I never found a combination of matching capacitors that resulted combination with low VSWR. Could be my testing method, but it's notoriously difficult to get antennas right, so I'd rather go with a tried and true off the shelf antenna.

### Miscellaneous
Deciding to incorporate the camera circuitry onto the same pcb as the tracker I think was a mistake. In the interest of long-term design flexibility, moving the camera to a completely separate PCB saves cost and weight for missions where I don't need a camera.

For the most part, V1 is fully functional, and I plan on doing a test launch once the firmware is in a testable state. All things considered, I'm happy that the basic functionality of V1 is operational. Going forward, the main hurdle will be the telemetry protocol implementation, which is still a preliminary proof of concept at this point.
