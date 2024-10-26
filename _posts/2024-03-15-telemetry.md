---
layout: post
title: Telemetry Protocol
---

One of the main features I want from this tracker is bidirectional radio communication. With this in mind, I designed the telemetry protocol to support on-the-fly configuration via uplink commands. This also enables the chase vehicle to pick and choose which data the tracker sends. Since the radio modem's transmit power is relatively low, I've come up with a plan for when radio communication is lost. Since radio parameters are dynamically configurable, there is a potential to get into an unrecoverable state if there's an issue with the configuration process. A simple solution to this is via implementation of 'fallback' state for both radios. 

![Telemetry Protocol](/assets/protocol-flowchart.svg)

If either radio hasn't received a valid packet from the other for a certain duration, both will revert to a hard coded set of frequency, bandwidth, and spread factor. These fallback parameters should be chosen for long range, low data rate in case the 'normal' radio parameters doesn't have enough range. Finally, I've decided to have the ground station query telemetry packets from the tracker during normal operation (both radios can 'hear' each other). If the radio link fails and both revert to the fallback state, the tracker will begin broadcasting telemetry packets on its own. I'm doing it this way is for the following reasons: If the communication link is lost for a short time (tracker obstructed by buildings, terrain, etc), it will be wasting transmit power unnecessarily. Querying the tracker, however, doesn't have this downside since it will only transmit telemetry if and when it receives a request. Then, if communication is totally lost (fallback radio parameters), the tracker can blindly send out telemetry. This is to guard against circumstances where the base station's transmitter (or beacon's receiver) is damaged/inoperable. The flow chart below details the tracker's internal state machine.

