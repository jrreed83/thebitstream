---
title: How to Build a Basic QPSK Modulator
author: Joey Reed
date: '2024-12-08'
draft: false
summary: A quick and dirty introduction to baseband QPSK modulation using Python.
tags:
  - digital communications
jupyter: python3
format: hugo
math: true
---


**Quadrature Phase Shift Keying**, or QPSK for short, is a digital modulation technique that encodes information onto a carrier wave by applying $45^\circ$, $135^\circ$, $225^\circ$, or $315^\circ$ phase shifts at a fixed rate, called the **symbol rate**. Each phase shift is called a **symbol**. For QPSK, a symbol is represented by 2 bits.   

## Derivation

To understand QPSK, we need to start with analog phase modulation.  Analog modulation has been around for a long time and doesn't depend on fancy, sophisticated integrated circuits or computers.  Instead, it relies on cleverly arranging circuit elements to continously tweak some property of a sinusoidal signal.  

Fortunately for us, scientists and engineers discovered how to model analog systems mathematically.  The equation for phase modulating a carrier with frequency $f$ Hz with a **baseband signal** $\phi(t)$ is    

$$
y(t) = \cos(2\pi f t + \phi (t)).
$$

All the information is in the baseband signal.  The carrier just *carries* it to where it needs to go.

This formula is not a very useful way to develop QPSK modulation because the carrier and information signal appear together under the $\cos$ function. We need to rip them apart so the baseband signal can be synthesized digitally and converted to an analog waveform at the right moment.

Fortunately, the fix is simple. We just need to apply one of the angle sum formulas you probably learned in high school:

$$
\cos(x+y) = \cos(x)\cos(y) - \sin(x)\sin(y)
$$

Applying this to $y(t)$, we get

$$
y(t) = \cos (\phi (t))  \cos(2\pi f t) - \sin(\phi (t)) \sin(2\pi f t)
$$

I don't like all the parenthesis. To clean it up, let's substitute $I(t)$ for $\cos (\phi (t))$ and
$Q(t)$ for $\sin(\phi (t))$:

$$
y(t) = I(t)  \cos(2\pi f t) - Q(t) \sin(2\pi f t)
$$

$I(t)$ and $Q(t)$ are called  **in-phase** and **quadrature** signals, and show up frequently in digital communications.
 

## What's the Point?

This post is all about calculating $I(t)$ and $Q(t)$ at discrete moments in time. This is what makes it digital. In a real system, you'd use a **digital to analog converter** (DAC) and **reconstruction filter** to transform the digital samples into a smooth, continuous, analog waveform.  We'll talk about interfacing with DACs and designing reconstruction filters sometime in the future.

## Programming Style

Before we get started on the modulator, I want to comment on my programming style. I like my modem simulations to be as simple as possible. No fancy programming-language magic or crazy abstractions allowed. I want to be able to take my simulation code and easily convert it into a hardware description language (Verilog/System Verilog) or low-level programming language (C), without thinking too hard. So if you look at my code and wonder why I didn't use a certain language feature, that's why.

## My Preferred Toolchain

Python is my preferred language for writing modem simulations. It's simple, has decent performance as long as you use the right libraries, and doesn't require a lot of ceremony. You can just open up a text file and hack away.

Unlike some numerical-focused scripting languages I've used before (like Matlab or Julia), efficient arrays aren't built in. You need into import them through third-party libraries. This leads to some clunky syntax, but you get used to it. In my opinion, the benefits of using Python outweigh the annoyances.

## Modulator Simulation

The three packages I rely on to write effective simulations in Python are

1.  `numpy` for efficient arrays and math,
2.  `matplotlib` for plotting
3.  `scipy` for filter design

For this first post we can get away without using `scipy`. 

To kick things off, let's import the packages:

``` python
import numpy as np 
import matplotlib.pyplot as plt   
```

### From byte arrays to complex QPSK symbols

Let's select a payload. In a real system, the payload is the application specific data you want to send to the receiver. It could be sensor data, text messages, pretty much anything. Unless I have a compelling reason not to, I like my simulated payloads to be English phrases. It makes debugging easier. I'm a big fan of using "hex-speak" for this sort of thing. Hex-speak is a little language built by expressing unsigned integers in hexadecimal. Most of them are pretty funny, and just lighten the mood. Here are a few of my favorites: `0xDEADBEEF`, `0xFEEDBABE`, `0xDECAFBAD`, `0xBADF00D`. Let's combine them into one big hex-speak phrase and partition them into bytes:

``` python
payload = [
    0xDE, 0xAD, 0xBE, 0xEF, # DEADBEEF
    0xFE, 0xED, 0xBA, 0xBE, # FEEDBABE
    0xDE, 0xCA, 0xFB, 0xAD, # DECAFBAD
    0xBA, 0xAD, 0xF0, 0x0D  # BAADF00D
]
```

#### Bytes to Bits

Now that we have the payload, we can start progressively moving toward building the symbols. First, we need to convert the byte array to a bit array. This boils down to extracting the 8 bits from each byte, and concatenating them all together. There are several ways to do this in Python. Here's the version that most closely follows what you might do in a language like C.

``` python
num_chars = len(payload)
num_bits = num_chars * 8
payload_bits = np.zeros(num_bits)
k = 0
for i in range(num_chars):
    byte = payload[i]
    for j in range(8):
        payload_bits[k] = 1 & (byte >> j)
        k += 1
```

#### Bits to Symbols

Converting bits to symbols is pretty simple. The first thing we need to do is split the bit array in half. Half the bits will be used for the in-phase signal and the other half will be used for the quadrature signal. As long as you use the same splitting method in the receiver, you can do this however you want. I always split the bits based on whether the bit index is even or odd. The even bits become the in-phase bits and the odd bits become the quadrature bits:

``` python
i_bits = payload_bits[0::2]
q_bits = payload_bits[1::2]
```

Next, take a  bit from each array, and map the bit pair to a point in the `XY`-plane. Here's one way to do this that takes advantage of `numpy`'s vectorization capabilities.

``` python
i_symbols = 2 * i_bits - 1
q_symbols = 2 * q_bits - 1

iq_symbols = i_symbols + 1j * q_symbols
```

### Pulse Shaping Filter

Now that we've converted the payload to an array of symbols, it's time to start building the baseband waveform. First thing we need to do is select a suitable pulse-shaping filter. A pulse shaping filter is a special type of digital **finite inpulse response** (FIR) filter with properties that be tuned to achieve a certain bandwidth.

The first pulse-shaping filter I ever learned about was the **root-raised cosine filter**. My hand-rolled version (shown below), is not the prettiest code I've ever written, but it works.

``` python
def root_raised_cosine(
    rate_i=1,       # Input sample rate (always set to 1)
    rate_o=16,      # Ouput sample rate (interpolation factor)
    beta=0.5,       # Excess bandwidth parameter (between 0 and 1)
    delay=5         # Number of symbol periods it takes for the peak to occur
):

    samples_per_symbol = rate_o // rate_i 

    n = int(samples_per_symbol * delay)
    x = []

    # Add first element
    x = x + [1 + beta * ((4/np.pi) - 1)]
    for i in range(1,n+1):
        if i == (samples_per_symbol/(4*beta)):
            sin_ = np.sin(np.pi/(4*beta))
            cos_ = np.cos(np.pi/(4*beta))
            c1  = (beta / np.sqrt(2))
            c2  = 1+(2/np.pi)
            c3  = 1-(2/np.pi)
            xi  = c1 * ((c2 * sin_) + (c3 * cos_))
        else:
            sin_ = np.sin(np.pi * i * (1-beta) / samples_per_symbol)
            cos_ = np.cos(np.pi * i * (1+beta) /samples_per_symbol)
            c1  = 4 * beta * i / samples_per_symbol 
            c2  = np.pi * (i / samples_per_symbol) * (1 - c1**2)  
            xi  = (sin_ + (c1 * cos_)) / c2

        x = [xi] + x + [xi]


    return np.array(x)
```

The function arguments give you control over the bandwidth of the output baseband waveform. Rather than spend a lot of time explaining this, let's keep moving through the modulator design. I think this is the best way to see how everything fits together.

Okay, let's say we want to communicate at a rate of 1000 symbols per second and interpolate by 16 times. We'll keep `rate_i` at 1 and change `rate_o` to 16.  The ratio of `rate_o` to `rate_i` should equal the amount we want to interpolate by.

-   `delay` controls how many symbol periods you want the filter to last. I usually keep this at 5.
-   `beta` gives you fine-grained control of the bandwidth of the signal. In my experience, this is usually set to 0.25 or 0.5.  The higher the value, the higher the bandwidth.

Here's how we create the shaping filter for this scenario:

``` python
shaping_filter = root_raised_cosine(rate_o=16, rate_i=1, beta=0.5, delay=5)
```

and here's what the filter looks like as a function of symbol period. This is also called the filter's **impulse response**. There are 10 symbol periods because `delay` is set to 5. If `delay` was set to 23, it would span 46 symbol periods.

``` python
plt.figure(1)
inds = np.linspace(-5, +5, len(shaping_filter))
plt.plot(inds, shaping_filter)
plt.xlim([-5, 5])
plt.xticks(np.arange(-5, +6))
plt.xlabel("Symbol Period")
plt.ylabel("Amplitude")
plt.grid()
```
{{< figure src="index_files/figure-markdown_strict/cell-9-output-1.png">}}
<img src="index_files/figure-markdown_strict/cell-9-output-1.png" width="679" height="434" />

In case you like formulas, here's the rule governing the number of samples in the root-raised cosine filter:

$$
\text{number coefficients } = 1 + \frac{\text{ rate_o }}{\text{ rate_i }} \times \text{ delay } \times 2
$$

### Interpolation Process

The next step in the process is to interpolate the symbols with the filter. There's an inefficient, easy way, to do this and an efficient, more complex way to do this. Let's start with the easy way.

#### The easy way

For the easy way, we upsample the symbol array by a factor of 16 and apply the filter using `numpy`'s convolution function:

``` python
num_symbols = len(iq_symbols)
iq_symbols_upsampled = np.zeros(num_symbols * 16, dtype=complex)
iq_symbols_upsampled[::16] = iq_symbols

baseband_waveform = np.convolve(iq_symbols_upsampled, shaping_filter)
```

If you're not familiar with convolution, you can think of it as a walking multiply and accumulator. The filter walks across the input one sample at a time, point-wise multiplies the filter coefficients with a segment of the filter input input coefficients, and adds the results.

To plot the output with respect to time, we need to calculate the number of samples per second (aka sample rate) with a unit conversion:
$$ 
\frac{\text{samples}}{\text{second}} = \frac{\text{samples}}{\text{symbol}} \times \frac{\text{symbols}}{\text{second}}
$$

Here's the accompanying code:

``` python
samples_per_symbol = 16
symbols_per_second = 1000
samples_per_second = samples_per_symbol * symbols_per_second
```

Now let's plot the waveform.

``` python
num_baseband_samples = len(baseband_waveform)
t_ms = 1000*np.arange(num_baseband_samples) / samples_per_second
plt.plot(t_ms, np.real(baseband_waveform), label="I")
plt.plot(t_ms, np.imag(baseband_waveform), label="Q")
plt.xlabel("Time (ms)")
plt.ylabel("Amplitude")
plt.legend()
```
{{< figure src="index_files/figure-markdown_strict/cell-12-output-1.png" >}}
<img src="index_files/figure-markdown_strict/cell-12-output-1.png"/>

Can you spot why this is inefficient? Upsampling distributes the samples over a large array full of zeros. Unfortunately, the convolution function doesn't know anything about this. It's going to do what it normally does (which is multiply and accumulate), even if most of the entries are 0. This is a huge waste of time and energy. If we know for a fact that an element in an array is 0, we should just skip over it.

#### The hard way

This brings us to the harder, but more efficient way of doing this using **multirate signal processing**. There's no way I can explain everything about multirate signal processing here. It's an entire area of specialization. What I can do is give some motivation, and code to demonstrate the process.  If you're interested in learning more about it, here's a list of some authoritative books:

* [Multirate Signal Processing for Communications Systems - Fredric J Harris](https://www.amazon.com/Multirate-Processing-Communication-Systems-Publishers/dp/877022210X/ref=asc_df_877022210X/?tag=hyprod-20&linkCode=df0&hvadid=693296405172&hvpos=&hvnetw=g&hvrand=4559866533785259274&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9192171&hvtargid=pla-1161835612443&psc=1&mcid=08126222a4593f0dac4cbeedbdc04b2d&tag=hyprod-20&linkCode=df0&hvadid=693296405172&hvpos=&hvnetw=g&hvrand=4559866533785259274&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9192171&hvtargid=pla-1161835612443&psc=1)
* Multirate Signal Processing - Ronald Crochiere and Lawrence Rabiner
* Multirate Systems and Filter Banks - P.P Vaidyanathan
* Wavelets and Filter Banks - Gilbert Strang and Truong Nguyen

Mulirate signal processing is all about restructuring the workload in a problem to improve
computational efficiency. On a beefy processor, this might not matter. But if you're system needs to run in real-time, on resource constrained hardware, processing overhead becomes a concern.

To apply multirate signal processing to the pulse shaping task, we transform the 161 element, single dimensional array into a two-dimensional *filter bank* consisting of 16 rows and 11 columns. Yes, the number of rows must match the number of samples per symbol.

``` python
filter_bank = np.reshape(
    np.concatenate((shaping_filter, np.zeros(15))),
    (16, 11),
    order="F"        
)
```

Tacking on 15 zeros at the end of the shaping filter makes the reshaping workout.The filter bank is applied to the input using a custom algorithm. No upsampling required.

``` python
iq_symbols_1 = np.concatenate((iq_symbols, np.zeros(16)))
buffer = np.zeros(11, dtype=complex)
baseband_samples_fb = np.zeros(len(iq_symbols_1) * samples_per_symbol, dtype=complex)
k = 0
for i in range(len(iq_symbols_1)):
    buffer[10] = buffer[9]
    buffer[ 9] = buffer[8]
    buffer[ 8] = buffer[7]
    buffer[ 7] = buffer[6]
    buffer[ 6] = buffer[5]
    buffer[ 5] = buffer[4]
    buffer[ 4] = buffer[3]
    buffer[ 3] = buffer[2]
    buffer[ 2] = buffer[1]
    buffer[ 1] = buffer[0]
    buffer[ 0] = iq_symbols_1[i]

    for index in range(samples_per_symbol):
        baseband_samples_fb[k+index] = np.sum(buffer * filter_bank[index,:])

    k += 16
```

The workload improvements are pretty incredible:

-   Easy way: ~300 multiplies and additions for a single output sample.
-   Hard way: ~20 multiples and additions for a single output sample.

Not bad right? The only price you pay is a more complex implementation. When you
get used to how multirate signal processing works though, it it's not too bad.

And here's what the in-phase and quadrature signals look like when we use multirate
processing.

``` python
num_baseband_samples_fb = len(baseband_samples_fb)
t_ms = 1000*np.arange(num_baseband_samples_fb) / samples_per_second
plt.plot(t_ms, np.real(baseband_samples_fb), label="I")
plt.plot(t_ms, np.imag(baseband_samples_fb), label="Q")
plt.xlabel("Time (ms)")
plt.ylabel("Amplitude")
plt.legend()
```
{{< figure src="index_files/figure-markdown_strict/cell-15-output-1.png" width="675" height="429" >}}
<img src="index_files/figure-markdown_strict/cell-15-output-1.png"/>

Besides being slightly different lengths, the easy and hard ways give exactly the same results.

### How do we Validate the Implementation?

A great question to ask at this point is

> "yeah, but how do you know this is correct?"

Great question😊!

My short, but terrible answer to this question is

> "I can tell by how the waveform looks".

We'll give an actual answer in the next post in this series.

## Conclusion

If you read until the end, thank you. This was much longer than I originally intended, but there was a lot to cover:

1.  We went through some math that described how QPSK modulation works
2.  Described how to go from a digital payload represented as an array of bytes to an array of QPSK symbols.
3.  Discussed pulse shaping filters.
4.  Touched on the benefits of multirate signal processing.

Next time, we'll justify why the baseband waveform we synthesized properly encodes the payload.
