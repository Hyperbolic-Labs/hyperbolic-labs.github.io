---
layout: post
title: Telesonde Project
---

I've been always been fascinated with GPS trackers, and high altitude ballooning comes with wide array of engineering challenges that I find appealing. Of course, there are many tried and true platforms out there with varying degrees of utility. Most of the well documented examples that exist on the internet use APRS or WSPR networks for relaying telemetry data, and for good reason. These networks are wide reaching and the frequencies in use enable payload tracking across the globe, and lets users track the position of their balloon live via a web interface such as SondeHub. However there are a few ideas of my own that I haven't seen yet, so I'd like to documentmy design process for those who find it insightful. I'll go into all the important considerations that go into designing a high altitude tracking system from the ground up,and hopefully others can learn from my many mistakes.As with any project, it's critical to have concrete goals before you begin. Since I began this project 2 years ago the goals have morphed over time, but the same core concepts remain.

### Goal #1:    Detachable Payload
Ever since stumbling across Dave Ackerman's excellent series on LoRa tracking, I've wanted to experiment with bidirectional communication. One in particular is the the ability to detach the payload from the balloon if necessary. I happen to live near the ocean, and occasionally the wind direction will blow towards the coast which would make payload retrieval impossible. The ability to send a detach command to the payload is a great way to prevent it from drifting too far.

### Goal #2:    Dynamic Configuration
Of slightly less importance, I'd like the ability to change mission settings on the fly. There are dozens of things that can go wrong during a high altitude mission, and a single component failure can ruin the chances of recovering the payload. Received signal strength too low? Update the LoRa modulation parameters to get more range. Make a mistake in the pre-flight configuration? Fix it after the fact, etcetera.

### Goal #3:    Off-Grid Tracking
Where I live, mountains significantly limit cell coverage, meaning that the chase vehicle doesn't have a reliable internet connection. In order to maximize the probability of finding the payload after it lands, I'm implementing a custom point to point protocol over LoRa. This approach comes with its own set of downsides like not having dozens of redundant receive stations; however, I think most of these drawbacks can be overcome with some smart design choices. Reiterating, this HAB platform is realistically only suited for payloads that will be retrieved.

### Goal #4:    Flexible Software Platform
As a software engineer, having a polished and reliable firmware is something that carries a lot of weight for me. My goal is to create a clean, easily portable codebase that others can modify for easily implementing their own ideas, hopefully lowering the barrier to entry for this hobby. A lot of inspiration has been taken from the Meshtastic project, namely the abstraction layer between the application and the hardware it runs on. Ideally, I'd like to edit only a few implementation files whenever I develop new tracking hardware.

During my research I've come across some great pages of people documenting their high altitude balloon launches, and here are links to a few of the pages I found most useful:

- [https://www.daveakerman.com/](https://www.daveakerman.com/) - HAB tracking with Raspberry Pi and LoRa
- [https://tt7hab.blogspot.com/](https://tt7hab.blogspot.com/) - Excellent hardware testing & custom balloons
- [http://www.projecthorus.org/](http://www.projecthorus.org/) - Tracking & flight prediction tools
