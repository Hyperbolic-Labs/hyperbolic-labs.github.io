---
layout: post
title: V2 Hardware - Other Sensors
---

### Barometer
During the component selection process, the board assembler JLCPCB didn't have many options for pressure sensors. I really liked the accuracy and low noise characteristics of the BMP581 sensor in the first design, but unfortunately wasn't in stock at the time. After a little research I found the DPS310 sensor from Infineon, in part thanks to the very helpful chart detailed [here](https://hackaday.io/project/86912-explog-exploration-logger/log/123294-choosing-the-right-sensors). I've never used this sensor before, but it appears to stack up pretty well compared with the Bosch sensors, and was available at a reasonable price. 

### Accelerometer
On a whim I decided to add a low cost accelerometer to the design, [LIS2DW12](https://www.st.com/resource/en/datasheet/lis2dw12.pdf). It has an I2C interface, consumes very low power, and conveniently in a tiny 2x2mm LGA package. This will let me experiment with some burst/landing detection ideas I have. At the moment I don't know how how useful these features will actually be, but in the future can see it being used to implement a low power mode which wakes the CPU when movement is detected.

### Battery monitoring
Measuring battery voltage is easily achieved via a simple voltage divider which the STM32's internal ADC can read directly, but measuring battery current requires a few external components. I opted for a INA199 low offset differential amplifier chip, which measures high side battery current. 
For measuring battery current, I went with a INA199 current sense amplifier. This chip is a low offset opamp with integrated differential gain resistors. I could have saved a few cents by using an off the shelf op amp and discrete resistors, but having matched resistors is critical for measurement accuracy. The INA199 solution has a smaller PCB footprint and should have excellent DC accuracy, so I just went with it. Battery voltage is sensed via a simple voltage divider. As a side note, the STM32 positive ADC reference is VDD, which is 3.3V. In order to get accurate readings, I first measure the internal voltage reference to determine the exact VDD voltage. This value can then be used to accurately scale the current and voltage measurements appropriately. 


![](/assets/current-sensor-sch.png)

I chose the combination of a 10mOhm resistor and the amplifier variant with a gain of 50. The STM32's positive ADC reference is 3.3V, which means this setup can theoretically measure up to 6.6A. Looking back I should have chosen the -A3 variant with a gain of 200, but at the time I massively overestimated the maximum battery current draw. 

The built in sensors on Floatini V2 should be enough for all the basic high altitude metrics, but additional sensors can be used via the QWIIC I2C interface. 