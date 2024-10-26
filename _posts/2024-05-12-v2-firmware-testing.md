---
layout: post
title: Firmware Development - Data Storage
---

For a balloon mission to be successful, the tracker needs a reliable way to store non volatile information. There are lots of ways to manage persistent data, but I'd like to share the thoughts I had before I committed to my current implementation. 

### Settings Storage
Floatini V2 integrates an I2C EEPROM for storing settings and other mostly static data (although I'm not quite sure what at the moment). A more cost constrained design could easily move data storage to the NOR flash chip or even the STM32's internal storage, but I wanted a dedicated chip for a few reasons. The first being increased write endurance compared to flash technology. The M95C64 chip I chose for this design is rated to 4 million cycles. This equates to a minimum lifetime of over 1000h (1 write per second), making the consideration of write wear largely irrelevant. Secondly, during the firmware development process a decent amount of time is spent getting the data partitioning right. Making the settings part of the same storage medium adds further software complexity. 

In retrospect I think I2C is an inferior interface for an EEPROM however. While writing EEPROM driver code, I managed to get the I2C bus stuck into a arbitration lost (ARLO) state, which I had not previously considered. ARLO is a fairly broad error, but generally this will happen when an I2C device pulls SDA low when the bus is in use by two other devices. This unusual state can cause undefined behavior in I2C peripherals, and it's difficult to guarantee the validity of any data on the bus after a collision occurs. Sometimes this can be accomplished by clocking SCL 9 times with SDA held high, but it didn't work for me. After some more reading, sometimes the only way to recover is via a complete power cycle to all devices on the bus. Eventually I discovered the ARLO was caused by a bug in my driver code, but definitely made me reconsider the reliability of I2C. Additionally, only a single I2C device has to fail with SDA pulled low, and it takes down the entire bus. In the next version, I'd prefer to go with an SPI EEPROM instead for these reasons. 

### Data Logging
For recording mission data, I have a 64Mbit NOR flash chip. NOR flash is inherently more power efficient than NAND, but has significantly less capacity. For this reason I decided against the virtual filesystem route. There are some great open source filesystem libraries specifically for embedded, but it seemed like too much overhead for this use case, especially since nearly all accesses will be sequential. Virtual filesystems are a lot less deterministic as well, something I'd rather avoid. Sometimes simpler is better.

In the code, the flash chip is partitioned into two sections, log data and event data. For now, log entries are composed of a 256-byte block of packed datum values, although I may update this to support 512 byte entries in the future. NOR flash architecture dictates that individual bytes may only be programmed once until erasure (some chips allow writing individual bits within a byte from 1 to 0 but this gets complicated). Making the log entries the same size as a flash page ensures the page programming duration remains consistent, so it's easy for the application to detect if/when something goes wrong. The time to program one byte is generally the same as an entire page, so log entries are multiples of 256 for maximum efficiency. 

### Data Log Entry

| Byte 0            | Byte 1   |     | Byte n+1 | Byte n+2   |     | Byte 255   |
|-------------------|----------|-----|----------|------------|-----|------------|
| 0xFF (reserved)   | datum[0] | ... | datum[n] | 0xFF (pad) | ... | 0xFF (pad) |

A continuous datalog captures a large amount of diagnostic data, but don't always capture events that happen inside the log interval. Because of this, I dedicated a second portion of flash for event logging. Timestamped events paint a more complete picture of what was going on at the time, so I opted for a time-delta approach with 1ms resolution. Each event record consists of a 2 byte time delta (ms since previous event) and a single byte which is the event ID. If there hasn't been an event in 65.535 seconds, a null 'padding' event is generated to reset the time delta to zero. 

### Event Entry

| Byte 0            | Byte 1          | Byte 2   |
|-------------------|-----------------|----------|
| time delta[15:8]  | time delta[7:0] | Event ID |

It takes 85 records to fill a page of flash, with 1 byte to spare, which accounts for up to 90 minutes. It's difficult to preemptively estimate how many events will be generated, but let's assume a worst case scenario of 1 event per second. One flash page will last 85 seconds, so a 256kB flash partition offers 24 hours of recorded events -- more than enough for this application. I'll probably write events to flash immediately, since appending them to a 256b page buffer means all unflushed events would be lost if an unexpected reset occurs. And of course, the most informative events for troubleshooting happen just prior to a reset. 

### Event Log Page

| Byte 0            | Byte 1   | Byte 4   |     | Byte 3n+1  |     | Byte 253   |
|-------------------|----------|----------|-----|------------|-----|------------|
| 0xFF (reserved)   | Entry[0] | Entry[1] | ... | Entry[n]   | ... | Entry[84]  |


On boot, there is the problem that the application doesn't know where the last log/event entries left off. I could keep track of the current datalog pointers in EEPROM, but chose to search manually to reduce external dependencies. The simplest way to initialize the flash page pointers is to simply iterate through all flash pages until the next empty one is found. The SPI peripheral has a 48MHz clock, and it takes just under 1uS to read a single byte, or ~44uS to read an entire page. Since there are 32,768 total pages, reading out every page could take up to 1.5 seconds. This number also doesn't consider the amount of time necessary for the STM32 to check if the page is empty. A much faster approach is to implement a binary search algorithm that only needs to check (at most) $$ log_2(n) $$ pages, which is 15 in this case. In the code, the binary search only has to check if the first byte is 0xFF (not programmed), which is I reserved for this purpose. In this way, I was able to get the flash initialization time under 2mS. 

