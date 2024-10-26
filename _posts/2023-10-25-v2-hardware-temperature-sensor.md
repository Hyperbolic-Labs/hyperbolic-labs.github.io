---
layout: post
title: V2 Hardware - Temperature Sensor
---

Ambient air temperature is a key datapoint to collect during a mission, and one of the simplest ways to accomplish that is with an thermistor. They're low cost and only require 1 pin from the MCU. The processor and pressure sensor both have internal temperature sensors, which version 1 planned on using. However, since they are board mounted, those sensors won't give an accurate air temperature reading, which is critical for computing barometric altitude. A thermistor can also report temperatures below -40C, which is the lower limit for most silicon based sensors. 

### Temperature Sensor
For measuring ambient air temperature, I chose a leaded NTC thermistor approach instead of a digital IC. Thermistors exhibit a non-linear, yet predictable change in resistance versus temperature. While somewhat primitive, they can measure a much wider range compared to most digital temperature sensors. Also since the sensor is located off PCB in this configuration, ambient air accuracy should be a little better. At upper levels of the atmosphere, the air is much thinner and likewise has much lower thermal conductivity. Sensors exposed to direct sunlight will read slightly higher temps than those in the shade. Supporting hardware for thermistors is minimal, just a reference resistor and reference voltage. 

![NTC Sensor](/assets/ntc-divider-sch.png)

Luckily the STM32's ADC os referenced tp VDD, which the thermistor-resistor voltage divider can draw from. In this way, ADC measurements can be translated directly into a ratio between reference resistance and thermistor resistance. Once the sensor's resistance has been calculated, this value gets plugged into the Hart-Steinhart formula, which NTC thermistors closely approximate. 

$$
\frac{1}{T} = A + B \ln(R) + C (\ln(R))^3
$$

where:
- $$ T $$ = Temperature in Kelvin (K)
- $$ R $$ = Resistance of the thermistor
- $$ A $$, $$ B $$, and $$ C $$ are Steinhart–Hart coefficients


There are two key specifications thermistors have: resistance at 25C, and a Beta parameter which describes the approximate 'slope' of resistance vs temperature. More specifically, beta can be calculated via the following formula:

$$
\beta = \frac{\ln\left(\frac{R_1}{R_2}\right)}{\frac{1}{T_2} - \frac{1}{T_1}}
$$

where:
- $$ R_1 $$ = Resistance of the thermistor at temperature $$ T_1 $$
- $$ R_2 $$ = Resistance of the thermistor at temperature $$ T_2 $$
- $$ T_1 $$ and $$ T_2 $$ = Temperatures in Kelvin (K)

Translation of this formula to code is fairly straightforward:

```cpp

#define G_NTC_VREF          (G_HW_VREG/1000.0f)  // NTC reference voltage (volts)
#define G_NTC_RDIV_H        56000.0f             // upper divider resistor, ohms
#define G_NTC_RDIV_L        30000.0f             // lower divider resistor, ohms
#define G_NTC_COEF_A_DEF    0.000801776f         // default coefficient A               
#define G_NTC_COEF_B_DEF    0.000264836f         // default coefficient B               
#define G_NTC_COEF_C_DEF    0.000000145f         // default coefficient C

int NTC_COEF_A = G_NTC_COEF_A_DEF;
int NTC_COEF_B = G_NTC_COEF_B_DEF;
int NTC_COEF_C = G_NTC_COEF_C_DEF;

float pTempSensorNTC::calc_sensor_resistance(float vadc)
{
    return (G_NTC_VREF*G_NTC_RDIV_L/vadc) - (G_NTC_RDIV_L+G_NTC_RDIV_H);
}

float pTempSensorNTC::fcalcTemp(float vadc)
{
    if(vadc == 0.0f)                             // ADC under-range error
        return -128.0f;  

    float r = calc_sensor_resistance(vadc);     // calculate voltage at ADC pin
    if(r <= 0.0f)                                // ADC over-range error
        return 127.0f;

    float lnr = logf(r);
    float denom = NTC_COEF_A + NTC_COEF_B*lnr + NTC_COEF_C*lnr*lnr*lnr;
    if(denom == 0)
        return -128.0f;
    return (1.0f / denom) - 273.15f;
}
```
The tricky part comes when attempting to calibrate the sensor. Thermistors are particulary difficult to calibrate due to their nonlinear response. The quickest way to improve accuracy is to apply single point calibration, but doesn't take into account the tolerance of the resistance curve. A more accurate calibration method is to record thermistor resistance at 3 different temperatures, and recomopute the Hart-Steinhart coefficients. After rearranging the formula to solve for coefficients, I ended up with a calibration utility which takes 3 temperature/resistance pairs as parameters.

```cpp
void pTempSensorNTC::calcCoefs(int8_t t1, int8_t t2, int8_t t3, 
                               uint32_t r1, uint32_t r2, uint32_t r3)
{
    // make sure t3 > t2 > t1
    if((t1 <= -273.15) || (t2 <= t1) || (t3 <= t2))
        return;

    // make sure r3 < r2 < r1
    if((r3 <= 0) || (r2 <= r3) || (r1 <= r2))
        return;

    // compute temporary variables
    double l1 = log(r1);
    double l2 = log(r2);
    double l3 = log(r3);
    double y1 = 1.0/(t1+273.15);
    double y2 = 1.0/(t2+273.15);
    double y3 = 1.0/(t3+273.15);
    double denom = l2-l1;

    if(denom == 0)
        return;

    double g2 = (y2-y1)/denom;
    denom = l3-l1;

    if(denom == 0)
        return;

    double g3 = (y3-y1)/denom;
    denom = (l1+l2+l3)*(l3-l2);

    if(denom == 0)
        return;

    NTC_COEF_C = (g3-g2)/denom;
    NTC_COEF_B = g2 - NTC_COEF_C * (l1*l1 + l2*l1 + l2*l2);
    NTC_COEF_A = y1 - (NTC_COEF_B + l1*l1*NTC_COEF_C) * l1;
}

```










