---
layout: post
title: Barometric Altimetry
---

During the component selection process, the board assembler JLCPCB didn't have many options for pressure sensors. I really liked the accuracy and low noise characteristics of the BMP581 sensor in the first design, but unfortunately wasn't in stock at the time. After a little research I found the DPS310 sensor from Infineon, in part thanks to the very helpful chart detailed [here](https://hackaday.io/project/86912-explog-exploration-logger/log/123294-choosing-the-right-sensors). I've never used this sensor before, but it appears to stack up pretty well compared with the Bosch sensors, and was available at a reasonable price. 

Using barometric pressure and temperature is an easy way to estimate altitude, but this method of altitude measurement often suffers from relatively large offset errors that don't affect GPS. During my research, I found that most examples of altitude estimation in code were using hypsometric formula. While simple, this formula has accuracy issues at the typical height many HAB flights can reach. After lots of digging, I came across an invaluable [paper](https://iopscience.iop.org/article/10.1088/1742-6596/2131/2/022053/pdf) with a proposed new formula that should be more accurate at higher altitudes:

$$
\frac{p(z)}{p_0} = e^{\frac{-mg_0 R}{k T_0} \cdot \frac{1}{b_0} \left( \frac{z}{R} \right)^{b_0}}
$$

Where $$ p(z) $$ is the gas pressure at an altitude of $$ z $$,

$$ p0 $$ is ground level pressure,

$$ m $$ is the average mass of one gas molecule,

$$ k $$ is Bolzmann's constant,

$$ T0 $$ is absolute temperature and ground level,

$$ g0 $$ is the intensity of ground level gravity,

$$ G $$ is acceleration of gravity

$$ M $$ and $$ R $$ are the mass and the radius of the Earth, and

$$ b0 $$ is an experimental 'magic' value 143.17


When solved for height above ground, the formula now becomes

$$
AGL = R \times \left( \left(1 - \ln\left( \frac{p(z)}{p_0} \right) \cdot \frac{k}{b_0 g_0 R} \cdot T_0 \right) ^{\frac{1}{b_0}} - 1 \right)
$$

This equation more accurately models the temperature lapse rate of the atmosphere, which is usually implemented as a piecewise linear approximation when using the hypsometric version. Weather induced local pressure deviations can also be responsible for large changes in the reported altitude, and I imagine much of this error can be compensated by measuring local atmospheric temperature. After my first successful launch, I will compare the altitude from both equations with the reported GPS altitude to see definitively which one has the edge. 

#### Implementation Tangent
Out of curiosity I decided to benchmark the altitude calculation function and discovered that a single iteration took over 20,000 CPU cycles to complete. To me this seemed unnecessarily inefficient, especially since the STM32's processor includes a floating point unit. Digging deeper, I found that the majority of the processing time was taken up with the exponential `pow()` operation. However, the way the formula is constructed makes for a near worst-case scenario for the `pow()` computation. The pressure ratio is used as the base operand, which makes it highly sensitive to rounding noise in the least significant digits. This is compounded by raising this base to a floating point exponent which further increases reliance on floating point precision. When these formulas are implemented with `powf()` (32b version of `pow()`), the effective altitude resolution is on the order of meters, not exactly ideal. 

There is a way to kill two birds with one stone, computing the result via interpolation. Cubic spline interpolation can be used to roll several compound functions into one, and with enough nodes can very accurately describe the 'correct' implementation. One of the most basic curves is [cubic hermite spline](https://en.wikipedia.org/wiki/Cubic_Hermite_spline), which is comprised of four basis functions that govern the overall shape. For whatever reason, I wasn't able to find a fast, concise implementation of cubic interpolation in C, so I just decided to write it from scratch. 

```cpp
template<typename T, uint8_t SIZE>
class Hermite
{
    public:
        constexpr Hermite(T in_min, T in_max, T (*func)(T), T (*deriv)(T)=0)
        {
            float span = in_max-in_min;
            for(uint8_t i = 0; i < SIZE; i++)
            {
                p[i] = in_min + (i/(SIZE-1.0))*span;
                f[i] = func(p[i]);
            }
            float dt = p[1]-p[0];

            for(uint8_t i = 0; i < SIZE; i++)
            {
                if(deriv)
                    fprime[i] = deriv(p[i]) * dt;

                // compute approx. deriv when not given
                else                                                
                {
                    // deriv from 1st and 2nd points
                    if(i == 0)
                        fprime[i] = (f[1]-f[0])/(SIZE-1.0);    

                    // deriv from adjacent points     
                    else if(i == SIZE-1)
                        fprime[i] = (f[i]-f[i-1])/(SIZE-1.0);

                    // deriv from last two points
                    else
                        fprime[i] = (f[i+1]-f[i-1])/(SIZE-1.0);
                }
            }
            dt_inv = 1.0/dt;
        }

        constexpr T compute(T x)
        {
            t = x*dt_inv;       

            // get index of first interpolation node: 
            i = t-p[0]*dt_inv;     

            // compute relative position inside interval             
            t -= p[i]*dt_inv;     

            t2 = t*t;
            t3 = t2*t;
            t3x2_sub_t2x3 = t3*2.0 - t2*3.0;

            // floating point multiply-accumulate is only 1 cycle with FPU
            pt = t3x2_sub_t2x3 * f[i];
            pt += t3 * fprime[i];
            pt -= t2 * fprime[i];               
            pt -= t2 * fprime[i];
            pt -= t3x2_sub_t2x3*f[i+1];
            pt += t3 * fprime[i+1];
            pt -= t2 * fprime[i+1];
            pt += t * fprime[i];
            pt += f[i];

            // check if input is within range
            overmax = (x >= p[SIZE-1]);         
            undermin = (x < p[0]);

            // we don't like branching!
            pt *= !(overmax | undermin);   

            // constrain result to defined output range     
            pt += overmax * f[SIZE-1];          
            pt += undermin * f[0];

            return pt;                          
        }

    private:
        T p[SIZE];                 // interpolation 'x' values
        T f[SIZE];                 // f(x) 'y' values
        T fprime[SIZE];            // f'(x)
        T dt_inv;                  // inverse of dt, 1/dt
        T t;                       // relative position (0-1) between nodes
        T pt;                      // interpolated result f(t)
        T t2, t3, t3x2_sub_t2x3;   // temporary variables
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

I made use of this fast interpolator such that it accurately follows the relationship of pressure ratio (p/p0) to altitude for a given exponent value, which happens to be relatively constant. In the end, I was able to reduce the computation time from ~250uS down to ~2uS all while improving altitude resolution. 250uS isn't too much in the grand scheme, but I can easily see this number increasing by an order of magnitude if the processor doesn't contain a FPU. 

The barometric pressure sensor I chose for Floatini V2 Nano is only specified down to ~30kPa, which will probably cause the altitude estimations past 30,000ft to have large errors. There a few sensors such as MS5607 and MS5611 that can measure much lower pressures, but I chose not to use these for cost reasons. This means that above 30,000ft the GPS will have to be relied on as the primary altitude source. I'm guessing the barometer will still give valid readings below 30kPa, but they're likely outside the specified accuracy values. With this assumption, I will be testing out an experimental complementary filter which fuses GPS and barometric altitude measurements. GPS is used as a noisy, zero-offset source and barometer readings are used to measure short term altitude variations. By experimenting with different filter cutoff frequencies I should be able to compensate for imperfect barometer linearity, without the noise associated with GPS.
