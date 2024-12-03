---
title: How to Build a Basic QPSK modulator
author: Joey Reed
date: '2024-11-27'
draft: false
summary: A quick and dirty introduction to QPSK modulation.
tags:
  - communications
  - digital modulation
  - filters
jupyter: python3
format: hugo
math: true
---


"QPSK" stands for *Quadrature Phase Shift Keying*. It's a digital modulation technique, that encodes information into a high frequency carrier signal introducing phase shifts of $45^\circ$, $135^\circ$, $225^\circ$, or $315^\circ$ at specific times. Each phase shift is called a *symbol* and for QPSK, a symbol is represented by 2 bits. Higher order Quadrature Amplitude Modulation (QAM) schemes use more bits per symbol.

Phase shift keying (PSK) is the discrete time version of analog phase modulation. Analog modulation predates the invention of integrated circuits, even transistors. They're built out of discrete circuit elements and as a result, tend to be simpler and consume less power than their digital counterparts. However, digital modulators are programmed with processors, making them far more flexible and precise.

## Derivation

The equation for phase modulating a carrier sitting at $f$ Hz with an information bearing baseband signal $\phi (t)$ is

$$
  y(t) = \cos(2\pi f t + \phi (t)).
$$

Turns out that this isn't a very helpful expression for developing a QPSK modulator because the carrier and baseband signals are intertwined. If we can rip them apart, the baseband signal can be synthesized digitally and converted to an analog waveform at transmission time, using common and inexpensive circuit components.

Fortunately, the fix is simple. All we need is one of the angle sum formulas you probably learned in highschool:

$$
  \cos(x+y) = \cos(x)\cos(y) - \sin(x)\sin(y)
$$

If we apply this to $y(t)$, we get

$$
  y(t) = \cos (\phi (t))  \cos(2\pi f t) - \sin(\phi (t)) \sin(2\pi f t)
$$

I don't like all the parenthesis. To clean it up, let's express this in terms of "I/Q modulation":

$$
  y(t) = I(t)  \cos(2\pi f t) - Q(t) \sin(2\pi f t)
$$

where $I(t)$ and $Q(t)$ are the so-called in-phase and quadrature signals. These signals pop up frequently in digital communications, and in
our particular case

$$
    I(t) = \cos (\phi (t)) \text{ and } Q(t) = \sin(\phi(t)).
$$

### What's the Point?

## A Simulation

Now it's time to start implementing a basic modulator in Python. The three most common packages I use for communications simulations
are `numpy`, `matplotlib`, and `scipy`. For this first one, we'll manage without `scipy`.

``` python
import numpy as np 
import matplotlib.pyplot as plt   
```

Next, we need to settle on a sequence of bits that will ultimately modulate the carrier. There are many ways to do this, but I'm a big fan of using "hex-speak" for bit sequences. Hex-speak numbers are unsigned integers that also spell out words when expressed in hexadecimal. Most of them are pretty funny, and just lighten the mood. It's a good thing. Here are a few of my favorites: `0xDEADBEEF`, `0xFEEDBABE`, `0xDECAFBAD`, `0xBADF00D`. We'll use a 32-bit hex-speak word:

``` python
message_hex = 0xBADDB0BA
```

Now we need to convert this number to an array of bits. Here's a way to do this using bitwise operations:

``` python
message_bits = np.array([(1 & message_hex >> i) for i in range(32)])
```

If you're not used to looking at bitwise operations before, this might look a little cryptic. All we're really doing is taking the $i^\text{th}$ bit and
plopping it in the $i^\text{th}$ position of the array.

The next step is to convert the bits to QPSK symbols. Typically, the even bits make up the in-phase signal and the odd-bits make up the quadrature signal.

``` python
I_bits = message_bits[0::2]
Q_bits = message_bits[1::2]
```

After the bits are partitioned, we build the array of complex symbols.

``` python
symbols = I_bits + 1j * Q_bits
```
