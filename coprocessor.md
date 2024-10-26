
Floatini Nano V2 Hardware Overview
----------------------------------------
Having learned from some of my mistakes with V1, I got started on version 2 hardware. The main goals of this version were power reduction and easier software development, and this led to an almost complete redesign. I also planned on submitting this design for JLCPCB's board assembly service, which meant selecting parts from their online catalog. 

Processor
One of the first design decisions I made was the main processor, since a lot of subsequent decisions originate here. Being familiar with the STM32 family from other projects, I started shopping around for a general purpose MCU with relatively low cost. More specifically, I wanted something with USB support for easy firmware updates, plenty of peripherals, and a decent amount of flash/ram. After shopping around I eventually landed on the STM32F411 offering, which had everything I needed for the lowest price. In addition, this MCU has been around for awhile and has lots of example code out there. The Cortex M4F processor runs at a maximum of 100MHz, with 512K flash/128K ram. This is probably overkill for this use case but it's worth the peace of mind sometimes, especially when developing untested firmware. Also, current consumption is significantly less than an ESP32, right around 25mA at 100MHz. For programming the MCU, I added pads for connection to an ST-Link programmer in addition to a micro USB port. This configuration makes it easy to upload firmware fast via the debug port, and still have the USB available for a virtual COM port. This processor can run from the internal RC oscillator, but needs an external crystal for reliable USB communication. I settled on an 8MHz crystal from NDK that's rated for-40C. In the previous design, I had issues getting the ESP32's crystal oscillator to reliably start. In retrospect, I'm pretty sure the crystal loading capacitors were calculated incorrectly. Luckily, ST has an informative [https://www.st.com/resource/en/application_note/an2867-guidelines-for-oscillator-design-on-stm8afals-and-stm32-mcusmpus-stmicroelectronics.pdf](appnote) for choosing the correct capacitors. Maintaining a stable system clock is also critical for parsing GPS data. A clock mismatch of only a few percent is enough to cause GPS checksum errors, and subsequently no position data. Future designs may be able to circumvent the need for a crystal with automatic baud rate detection, but for this version I need something reliable.

Storage
On the ESP32 platform, storing nonvolatile settings is easily managed with the Preferences library, which provides an abstraction layer for storing variables in flash memory. The STM32F411 processor unfortunately doesn't have enough flash memory to implement nonvolatile storage with adequate wear leveling. This problem is easily solved by adding an external I2C EEPROM, which are very cheap. I went with a M24C64 EEPROM with 8Kbyte capacity, which should be plenty for what I will need. The M24C series has a write endurance of 4 million cyles which means I don't have to worry about implementing a wear leveling algorithm, another plus. Version 1 of the HAB tracker had an SD card connector, which served dual purpose as a storage medium for images and datalogs. Since this version won't be integrating a camera, inexpensive NOR flash is the better choice. It's more reliable, consumes significantly less power, and has a smaller PCB footprint. W25Q64JF from Winbond has 8Mbyte capacity, which works out to just over 9 hours of log data, assuming a 256 byte record once per second.

New Power Supply
After re-reviewing my low temperature battery options, I discovered that Energizer's AA lithium batteries are a better choice due to their better energy density compared to a CR123A. The downside is that the boost converter will have to work a little harder since the nominal voltage is ~1.4V instead of 3.0V. In practice, this means I need to find a boost converter capable of operating down to 0.8V or less in order to extract as much energy as possible. There aren't too many boost converters available that can work this low. Another thing I had to watch out for was minimum startup voltage. Several boost converters I was considering needed a minimum of 1.8V to reliably start, and then could operate much lower due to internal bootstrap. Eventually I found TPS61021A from Texas Instruments. This chip can start up from an input as low as 0.5V, and can deliver plenty of power. Conveniently it operates at a much higher frequency than the old MCP1640 so I can use a much smaller inductor. Like the V1 design, the boost converter output is filtered through an LDO that cleans up some of the noise. To reduce power consumption at low load currents, most switching converters transition to psuedo-pwm mode, only charging up the output capacitors when the output voltage drops too low. This manifests a low frequency (~kHz) sawtooth waveform imposed on the output. For digital circuits this usually isn't an issue, but this will negatively impact the GPS and analog sensor readings. I went with a 3.3V TPS7A20 regulator which has excellent PSRR and noise characteristics, and doesn't add much cost. I configured the boost converter output for 3.6V which should give the LDO just barely enough headroom to adequately filter out noise. 5V USB power is fed in via a diode to this 3.6V node; this created some issues that I'll detail later. 

Sensors
During the component selection process, JLCPCB didn't have many options for pressure sensors. I was impressed with the noise level of the BMP581 sensor in the first design, but wasn't in stock at JLCPCB. After a little research I settled on the DPS310 sensor from Infineon, in part thanks to the very helpful chart detailed [https://hackaday.io/project/86912-explog-exploration-logger/log/123294-choosing-the-right-sensors](here). DPS310 stacks up pretty well compared to the Bosch sensors, and happened to be available at a reasonable price. On a whim, I decided to add a low cost accelerometer to the design. This will let me experiment with more accurate burst and landing detection. At the moment I don't know how how useful these features will actually be, but in the future it could be used to implement a low power mode that wakes when movement is detected. For measuring battery current, I went with a INA199 current sense amplifier. This chip is a low offset opamp with integrated differential gain resistors. I could have saved a few cents by using an off the shelf op amp and discrete resistors, but having matched resistors is critical for measurement accuracy. The INA199 solution has a smaller PCB footprint and should have excellent DC accuracy, so I just went with it. Battery voltage is sensed via a simple voltage divider. As a side note, the STM32's positive ADC reference is VDD, which is 3.3V. In order to get accurate readings, the mcu first reads the internal voltage reference to figure out the exact voltage of VDD. It then uses this number to accurately scale the current and voltage measurements appropriately. Ambient air temperature is a key datapoint to collect during a mission, and one of the simplest ways to accomplish that is with an thermistor. They're low cost and only require 1 pin from the MCU. The processor and pressure sensor both have internal temperature sensors, which version 1 planned on using. However, since they are board mounted, those sensors won't give an accurate air temperature reading, which is critical for computing barometric altitude. A thermistor can also report temperatures below -40C, which is the lower limit for most silicon based sensors. Ublox still makes the most reliable GPS receivers on the market, and MAX-M10S is a big improvement over the older MAX-M8 chip. Power consumption is reduced significantly and now hasa builtin LNA for better passive antenna performance. I've always been impressed with how well this receiver works so it's a no brainer for me. Since GPS receivers are sensitive to high frequency harmonics, I added series resistors to the UART and PPS pins which limit the rise/fall times. I also kept the W3011 passive antenna because it's always served me well in past projects. The onboard sensors cover all the basic high altitude metrics, but in the likely event that additional sensors are needed I provisioned a QWIIC connector for additional I2C devices.

Miscellaneous
I kept the SX1262 radio module from NiceRF from the previous design, mostly because it includes a TCXO. LoRa modules that don't have a TCXO are typically forced to operate at higher bandwidths because of temperature-induced frequency drift. This is undesirable because higher bandwidths severely limit the maximum transmission range. Given the complete failure of the pcb trace cutdown mechanism of V1, I had to come up with a completely different approach that had a better chance of working. Perusing Aliexpress, I discovered some tiny ceramic heating elements that seemed like they would work perfectly. They're made of a small coil of nichrome wire encased in a hollow ceramic core. I plan on feeding the end of the balloon tether through center of this heating core, which is held in place with a drop of candle wax. When power is sent through the heater core, the element softens the wax to let the string pull out, severing connection to the balloon. My main concern with this approach is whether the battery will be able to supply enough power to melt the wax at low temperatures. I will probably experiment with insulation techniques around the heating core so the element doesn't lose too much heat to the surrounding air. Element power is enabled via a low side N channel mosfet which is driven by the MCU. After landing, I anticipate degraded tracker GPS accuracy if it doesn't land upright. I provisioned two MCU pins to drive a piezo buzzer to make recovery easier. These pins will be driven out of phase so the piezo element sees 6.6V peak to peak. For development and debugging purposes there are 4 green LEDs which provide indication of radio transmit/receive status, GPS fix, and processor usage. I also added a micro slide switch as an input to the MCU, which allows software-configured functionality. For my purposes, I will probably assign this switch to set the current tracker mode: in one position, the tracker is in 'mission' mode, and continuously records and transmits data. In the opposite position, the tracker is configured to be in 'debug' mode, and doesn't record telemetry data.

Telemetry Considerations
One of the main features I want from this tracker is bidirectional radio communication. With this in mind, I designed the telemetry protocol to support on-the-fly configuration via uplink commands. This also enables the chase vehicle to pick and choose which data the tracker sends. Since the radio modem's transmit power is relatively low, I've come up with a plan for when radio communication is lost. Since radio parameters are dynamically configurable, there is a potential to get into an unrecoverable state if there's an issue with the configuration process. A simple solution to this is via implementation of 'fallback' state for both radios. 

![Telemetry Protocol](/assets/protocol-flowchart.png)

If either radio hasn't received a valid packet from the other for a certain duration, both will revert to a hard coded set of frequency, bandwidth, and spread factor. These fallback parameters should be chosen for long range, low data rate in case the 'normal' radio parameters doesn't have enough range. Finally, I've decided to have the ground station query telemetry packets from the tracker during normal operation (both radios can 'hear' each other). If the radio link fails and both revert to the fallback state, the tracker will begin broadcasting telemetry packets on its own. I'm doing it this way is for the following reasons: If the communication link is lost for a short time (tracker obstructed by buildings, terrain, etc), it will be wasting transmit power unnecessarily. Querying the tracker, however, doesn't have this downside since it will only transmit telemetry if and when it receives a request. Then, if communication is totally lost (fallback radio parameters), the tracker can blindly send out telemetry. This is to guard against circumstances where the base station's transmitter (or beacon's receiver) is damaged/inoperable. The flow chart below details the tracker's internal state machine.



Converting Atmospheric Pressure to Altitude
Using barometric pressure and temperature is an easy way to estimate altitude, but this method of altitude measurement often suffers from relatively large offset errors that doesn't affect GPS. During my research, I found that most examples of altitude estimation in code were using hypsometric formula. While simple, this formula has accuracy issues at the typical height many HAB flights reach. After lots of digging, I came across an invaluable paper with a proposed new formula that should be more accurate at higher altitudes: link This equation more accurately models the temperature lapse rate of the atmosphere, which is usually implemented as a piecewise linear approximation when using the hypsometric version. Weather induced local pressure deviations can also be responsible for large changes in the reported altitude, and I imagine much of this error can be compensated by measuring local atmospheric temperature. After my first successful launch, I will compare the altitude from both equations with the reported GPS altitude to see definitively which one has the edge. Algorithm Tangent:Out of curiosity I decided to benchmark the altitude calculation function and discovered that a single iteration took over 20,000 CPU cycles to complete. To me this seemed unnecessarily inefficient, especially since the STM32's processor includes a floating point unit. Digging deeper, I found that the majority of the processing time was taken up with the exponential pow() operation. However, the way the formula is constructed makes for a near worst-case scenario for the pow() computation. The pressure ratio is used as the base operand, and is highly sensitive to significant digits. This is compounded by raising this base to a floating point exponent which further increases reliance on floating point precision. When these formulas are implemented with powf() (the 32bit float version of pow()), floating point precision alone reduces altitude resolution to several meters. But there is a way to kill two birds with one stone: interpolation. So I wrote a cubic interpolation class that accurately models the relationship of pressure ratio (p/p0) to altitude for a given exponent value,which happens to be relatively constant. In the end, I was able to reduce the computation time from ~250uS down to ~2uS all while improving altitude resolution. 250uS isn't that much in the grand scheme, but I can easily see this computation time increasing by an order of magnitude if the processor doesn't have a floating point unit. Unfortunately, the barometric pressure sensor I chose for Floatini V2 Nano is only specified down to ~30kPa. This will probably cause the altitude estimations past 30,000ft to have large errors. There a few sensors such as MS5607 and MS5611 that can measure much lower pressures, but I omitted them from this hardware version for cost reasons. This means that above 30,000ft the GPS will be the altitude source. I'm guessing the barometer will still give valid readings below 30kPa but are likely outside the calibration range. With this assumption, I will be testing out an experimental complementary filter which fuses GPS and barometric altitude measurements. GPS is used as a noisy, zero-offset source and barometer readings are used to measure short term altitude variations. By experimenting with different filter cutoff frequencies I should be able to compensate for imperfect barometer linearity, without the noise associated with GPS.

```cpp
template<typename T, uint8_t SIZE>
class Hermite
{
    public:
        constexpr Hermite(T domain_min, T domain_max, T (*func)(T), T (*deriv)(T)=nullptr)
        {
            for(uint8_t i = 0; i < SIZE; i++)
            {
                p[i] = domain_min + (i/(SIZE-1.0))*(domain_max-domain_min);
                f[i] = func(p[i]);
            }
            float dt = p[1]-p[0];

            for(uint8_t i = 0; i < SIZE; i++)
            {
                if(deriv)
                    fprime[i] = deriv(p[i]) * dt;
                else                                                // compute approx. deriv when not given
                {
                    if(i == 0)
                        fprime[i] = (f[1]-f[0])/(SIZE-1.0);         // deriv from 1st and 2nd points
                    else if(i == SIZE-1)
                        fprime[i] = (f[i]-f[i-1])/(SIZE-1.0);       // deriv from adjacent points
                    else
                        fprime[i] = (f[i+1]-f[i-1])/(SIZE-1.0);     // deriv from last two points
                }
            }
            dt_inv = 1.0/dt;
        }

        constexpr T compute(T x)
        {
            t = x*dt_inv;                       
            i = t-p[0]*dt_inv;                  // get index of first interpolation node
            t -= p[i]*dt_inv;                   // compute relative position inside interval
            t2 = t*t;
            t3 = t2*t;
            t3x2_sub_t2x3 = t3*2.0 - t2*3.0;

            pt = t3x2_sub_t2x3 * f[i];
            pt += t3 * fprime[i];
            pt -= t2 * fprime[i];               // float multiply-accumulate is only 1 cycle with FPU
            pt -= t2 * fprime[i];
            pt -= t3x2_sub_t2x3*f[i+1];
            pt += t3 * fprime[i+1];
            pt -= t2 * fprime[i+1];
            pt += t * fprime[i];
            pt += f[i];

            overmax = (x >= p[SIZE-1]);         // check if input is within range
            undermin = (x < p[0]);

            pt *= !(overmax | undermin);        // we don't like branching!
            pt += overmax * f[SIZE-1];          // constrain result to defined output range
            pt += undermin * f[0];
            return pt;                          
        }

    private:
        T p[SIZE];                          // interpolation 'x' values
        T f[SIZE];                          // f(x) 'y' values
        T fprime[SIZE];                     // f'(x)
        T dt_inv;                           // inverse of dt, 1/dt
        T t;                                // relative position (0-1) between interpolation nodes
        T pt;                               // interpolated result f(t)
        T t2, t3, t3x2_sub_t2x3;            // temporary variables
        int32_t undermin, overmax, i;
};
```

Now, to use it in the application:

```cpp
// define function to approximate
constexpr float ln(float x)
{
    return logf(x);
}

// ... and it's derivative
constexpr float ln_deriv(float x)
{
    return 1.0f/x;
}

// instantiate interpolation object
Hermite<float, 16> fast_ln(1, 10, ln, ln_deriv);

float ln_3 = fast_ln.compute(3);
```

<!-- 
template<typename T, uint8_t SIZE>
class Hermite
{
    public:
        Hermite(const T domain_min, const T domain_max, std::initializer_list<T> func, std::initializer_list<T> deriv={})
        : Hermite(domain_min, domain_max, func.begin(), (deriv.size() >= SIZE) ? deriv.begin() : nullptr)
        {
        }

        Hermite(const T domain_min, const T domain_max, const T* func, const T* deriv=nullptr)
        {
            for(uint8_t i = 0; i < SIZE; i++)
            {
                p[i] = domain_min + ((T)(i)/(SIZE-1.0))*(domain_max-domain_min);
                f[i] = func[i];
            }
            for(uint8_t i = 0; i < SIZE; i++)
            {
                if(deriv)
                    fprime[i] = deriv[i] * (p[1]-p[0]);
                else                                                // generate derivative approximation when not specified
                {
                    // fprime[i] = 0;
                    if(i == 0)
                        fprime[i] = (f[1]-f[0])/(SIZE-1.0);
                    else if(i == SIZE-1)
                        fprime[i] = (f[i]-f[i-1])/(SIZE-1.0);
                    else
                        fprime[i] = (f[i+1]-f[i-1])/(SIZE-1.0);
                }
            }

            dt_inv = 1.0/(p[1]-p[0]);
            temp1 = p[0]*dt_inv;
        }

        Hermite(T domain_min, T domain_max, T (*func)(T), T (*deriv)(T)=nullptr)
        {
            for(uint8_t i = 0; i < SIZE; i++)
            {
                p[i] = domain_min + ((T)(i)/(SIZE-1.0))*(domain_max-domain_min);
                f[i] = func(p[i]);
            }
            for(uint8_t i = 0; i < SIZE; i++)
            {
                if(deriv)
                    fprime[i] = deriv(p[i]) * (p[1]-p[0]);
                else                                                // generate derivative approximation when not specified
                {
                    if(i == 0)
                        fprime[i] = (f[1]-f[0])/(SIZE-1.0);
                    else if(i == SIZE-1)
                        fprime[i] = (f[i]-f[i-1])/(SIZE-1.0);
                    else
                        fprime[i] = (f[i+1]-f[i-1])/(SIZE-1.0);
                }
            }
            dt_inv = 1.0/(p[1]-p[0]);
            temp1 = p[0]*dt_inv;
        }

        T getDomainMin()
        {
            return p[0];
        }

        T getDomainMax()
        {
            return p[SIZE-1];
        }


    private:
        T p[SIZE];
        T f[SIZE];
        T fprime[SIZE];
        T dt_inv, temp1;
        T t, t2, t3, t3x2_sub_t2x3, pt;
        int32_t undermin, overmax, i;
}; -->


Data Logging
Floatini V2 Nano makes use of a 64Mbit NOR flash chip for recording mission data. NOR flash is inherently more power efficient than NAND, but has significantly less capacity. For this reason I decided against the virtual filesystem route. There are some great open source filesystem libraries specifically for embedded, but it seemed like too much overhead for this use case, especially since nearly all accesses will be sequential. The flash chip is partitioned into two sections, log data and events. For now, log entries are composed of a packed 256 byte block of all datum values, although I may update this to support 512 byte entries in the future. NOR flash architecture dictates that individual bytes may only be programmed once until erasure (some chips allow writing individual bits within a byte from 1 to 0 but this gets complicated). The difference in programming time between one byte and a full page is generally insignificant, so log entries are multiples of 256 for maximum efficiency. A continuous datalog captures a large amount of diagnostic data, but doesn't always capture events that happen inside the log interval. In service of this, there is a second section of flash dedicated for event logging. Timestamping events paints a more complete picture of what was going on at the time, so I opted for a time-delta approach with 1ms resolution. Each event record consists of a 2 byte time delta (ms since previous event) and single byte which is the event ID. If there hasn't been an event in 65.535 seconds, a null 'padding' event is generated to handle the overflow condition. It takes 85 records to fill a page of flash, with 1 byte to spare, which accounts for up to 90 minutes. It's difficult to preemptively estimate how many events will be generated, but let's assume a worst case scenario of 1 event per second. Then one flash page will only last 85 seconds, so a 256kB flash partition offers 24 hours of recorded events -- more than enough for this application. On boot, the next flash address in sequence for data and event records isn't known, and must be discovered by reading through the flash. In the code, I've implemented a simple binary search that looks at the first byte of a page to determine if it has already been written to, since flash will always erase to all 0xFF. Binary search is significantly faster than a linear search, reducing flash parsing from a worst case of 1000ms down to 10ms. 

Watchdog
Protecting the system from an unexpected software lockup is important for increasing system reliability. Luckily, the STM32 platform contains a dedicated watchdog timer peripheral that must be periodically 'tapped'. The watchdog is clocked via the LSE internal clock to guard against the scenario where the main HSE/HSI oscillator fails. Once started, this mechanism is very robust against adverse conditions, but still has a critical weakness. Initialize watchdog first, or after other peripherals? If a hardware failure causes some peripheral to take longer than expected, getting stuck in an endless reboot cycle is non-ideal. Initializing the watchdog timer after other peripherals leaves the system vulnerable to getting stuck in an undefined state. There isn't a perfect answer to this dilemma, but I lean toward the latter option. A 'smart' way to circumvent this would be to use a boot failure counter that limits the maximum number of cyclic reboots, maybe even the ability to disable failed non-critical components. On subsequent hardware versions, I'd like to move the watchdog off chip entirely which overcomes this potential issue. 

Nonvolatile Data
I decided to incorporate an I2C EEPROM for storing settings and other non-volatile data. A more cost constrained design could easily move data storage onto the NOR flash chip or even the STM32's internal storage, but I wanted a dedicated chip for a few reasons. The first being increased write endurance compared to flash technology. The M95C64 chip I chose for this design is rated to 4 million cycles. This equates to a minimum lifetime of over 1000h with 1 write per second, making the consideration of wear leveling irrelevant. Secondly, during the firmware development process a decent length of time is spent getting the data partitioning right. Making the settings part of the same storage medium adds further software complexity. In retrospect I think I2C is the wrong mode of communication though. While debugging some issues in the EEPROM driver, I managed to get the I2C bus stuck into a arbitration lost (ARLO) state, which apparently is only recoverable via power cycle after further investigation. I eventually discovered this bus state was caused by a bug in the I2C driver code I had written, but the point still stands. The I2C bus is also more susceptible to single device failures. If a single I2C device on the bus decides to erroneously pull SDA or SCL low, the entire bus becomes inoperable. In the next version, I'll be going with an SPI EEPROM instead for these reasons. 

Battery monitoring
Measuring battery voltage is easily achieved via a simple voltage divider which the STM32's internal ADC can read directly, but measuring battery current requires a few external components. I opted for a INA199 low offset differential amplifier chip, which measures low side battery current. Usually it's preferable to measure high side current, but I didn't want the battery to get drained by unnecessarily by the gain resistors. The INA199's input common mode voltage extends down to -0.1V, so I was able to position the current sense resistor below ground to reduce design complexity. I chose the combination of a 10mOhm resistor and the amplifier variant with a gain of 50. The STM32's positive ADC reference is 3.3V, which means this setup can theoretically measure up to 6.6A. Looking back I should have chosen the -A3 variant with a gain of 200, but at the time I massively overestimated the maximum battery current draw. 

Temperature Sensor
For measuring ambient air temperature, I chose a leaded NTC thermistor approach instead of a digital IC. Thermistors exhibit a non-linear, yet predictable change in resistance against temperature. While somewhat primitive, they can measure a much wider range compared to most digital temperature sensors. Also since the sensor is located off PCB in this configuration, ambient air accuracy will be a little better. At upper levels of the atmosphere, the air is much thinner and likewise has much higher thermal conductivity. Because of this, sensors exposed to direct sunlight will read slightly higher temps than those in the shade. Supporting hardware for thermistors is minimal, just a reference resistor and reference voltage. Luckily the STM32's ADC os referenced tp VDD, which the thermistor-resistor voltage divider can draw from. In this way, ADC measurements can be translated directly into a ratio between reference resistance and thermistor resistance. Once the sensor's resistance has been calculated, this value gets plugged into the Hart-Steinhart formula, which NTC thermistors closely approximate. There are two key specifications thermistors have: resistance at 25C, and a Beta parameter which describes the approximate 'slope' of resistance vs temperature. More specifically, beta can be calculated via the following formula:

LoRa Radio
All communication to the tracker happens via a SX1262 radio transceiver module. I chose NiceRF module since they offer a 16x16mm module with TCXO, which has a huge impact on sensitivity during extreme temperature changes. The sensitivity of the LoRa receiver is inversely proportional to the bandwidth selected, and choosing a narrower bandwidth reduces the peak transmit current, improving battery life.



Release Mechanism
--------------------------
After trying several ideas I had around a heater-based cutdown mechanism, it was clear that this approach just wasn't going to work reliably at low temperatures. The battery can only supply a limited amount of current, and I found that 1W of power just wasn't enough melt through a polymer-based balloon tether. A micro sized solenoid could work,but most weren't designed to operate from a single AA cell, and most significantly increased the total weight. The minimum energy required to release the balloon is relatively minimal, and most solenoids have way more pull strength than is necessary. Given these facts, I decided to come up with my own purpose-built magnetic release that weighs almost nothing and requires very little power to operate. To accomplish this I'll be using an unshielded inductor with a ferrite core, which is mounted to a flex PCB. The flexPCB has a long tab which gets folded into a loop with a small Neodymium magnet on the end. When current is run through the inductor, an opposing magnetic field is generated which repels the magnet, releasing the balloon. 

![Magnetic Release](/assets/magnetic-cutdown-render.png)

This mechanism should be more reliable and only use a fraction of the energy used by the heater method. It's surprisingly difficult to search for unshielded inductors with the right geometry for this application, since shielded inductors are almost always superior. However, I found the LPO2506 series from Coilcraft to be nearly perfect. They are constructed from ceramic with an inner ferrite core which is about 5mm in diameter, and a thickness of 1.5mm. The magnetic specs aren't listed in the datasheet, but most ferrites saturate in the 0.25-0.4T range. This means by choosing a magnet with a similar flux density, the attraction force of the magnet can be roughly nullified by running the inductor near saturation. In addition, since the inductor will be pre-biased 'against' the field generated by the coil, it can be pushed past it's normal saturation current. This in turn means that not only will the magnet lose attraction to the ferrite, but it will actually see a repulsive force. Using the magnet calculator from K&J Magnetic's website, I calculated the surface flux density of a 5x1mm disc magnet to be ~0.25T. This works out great, since this means the ferrite core will be just barely saturated in the opposite direction (or close to it). Assuming the core saturates at 0.25T, this means all we have to do is pass through the 'normal' (unbiased) saturation current through the inductor to bring the net field close to zero. By somewhat arbitrarily choosing the 220uH inductor from this series, the DC resistance is listed as 3.7ohms. Assuming a battery voltage of 1.0V this works out to 0.24A through the inductor, just barely past the listed saturation current. In short,when 1.0V is applied to the inductor, the attraction force of the magnet should be close to zero, releasing the balloon. Of significant importance is the consideration of maximum holding force this mechanism can withstand. K&J magnetics also has a pull force calculator, and a 5x1mm disc magnet has a axial pull strength of 0.49lbs. The shear pull strength of a magnet happens to be about half of its axial pull strength as detailed here: https://www.kjmagnetics.com/blog.asp?p=shear-force. Meaning, this magnet will be able to withstand ~110g of vertical pull from the balloon (even without considering friction between magnet and inductor). I think this should be plenty of margin for this mechanism to work without releasing prematurely. There is one potential downside to this configuration however. The maximum payload weight is fundamentally limited by inductor selection. Due to core saturation, increasing magnet size doesn't increase holding strength: it only reduces the repulsion force when the inductor is powered. Payloads over 110g would need an inductor with a larger cross sectional area or be forced to use a different release mechanism altogether. 





vinp = R2 (vdd-vlow)/(R1+R2+Rsw) Measure op amp output voltage when lower vref is bat-, record reading. This value is PGA offset voltage times PGA gain error. offset voltage sum of op amp (internal) os plus error in vref voltage divider. 
We don't care if Rsw changes with temperature, as it only manifests as a Vos error in the end. calibration requirements:- two ground-referenced voltages needed (for offset and PGA gain cal)- both ref voltages must be >0V
- PGA in- can be muxed to groundvdd measurement voltage:    (3.0/3.2768)*1.4 = 1.28173828125upper, lower: 0.030000, 0.012000Errors, 1: 0.23%, 2: 0.33%, 3: 0.47%, 4: 0.32%, total: 0.03%R1: 232000.000000, R2: 169000.000000, R3: 2430.000000, R4: 1620.000000R1: 0.572769, R2: 0.417232, R3: 0.005999, R4: 0.004000

Watchdog Coprocessor
After much consideration, I decided to include a separate MCU to this design that acts primarily as a analog front end and watchdog for the RAK3172 LoRa module. This is in part due to the limited I/O ports of the LoRa module and also for more design flexibility. One microcontroller that caught my eye is the MSPM0L130x series from Texas Instruments. It's a Cortex M0 processor with lots of builtin analog peripherals that eliminate the need for a lot of discrete components, and can be found for less than $0.70 in quantity. 

Current sensor design
Having an accurate measurement of battery current provides another valuable datapoint that I'd like this design to have. The most cost effective way of sensing current in this instance is to incorporate the builtin PGA of the MSP. Using the internal PGA instead of an external op amp has some real accuracy benefits that matter in this case. While the on-chip gain resistors may not have the highest initial accuracy, they have excellent temperature drift characteristics. 




Analog Frontend
Due to the LoRa module's limited GPIOs, I opted for an inexpensive ADS1115 discrete I2C ADC. This chip is super handy because it integrates a builtin PGA, eliminating the need for a current sense amplifier for measuring battery current. 3 channels are dedicated for monitoring the supply voltages, Vbat, Vsys, and Vdd. The 4th channel is used to measure battery input current.



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

