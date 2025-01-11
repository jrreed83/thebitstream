---
title: How to Validate a Basic QPSK Modulator
author: Joey Reed
date: '2025-01-05'
draft: true
summary: An introduction to baseband QPSK modulation in Python.
tags: [digital communications]
jupyter: python3
format: hugo
math: true
ShowToc: true
---

This post will show you the steps I go through to verify that my QPSK modulator output is correct.  You'll probably want to check out my other QPSK article first: [How to Build a Basic QPSK Modulator](https://thebitstream.me/posts/how-to-build-qpsk-modulator/), otherwise this may not make sense.

## Visual Inspection

The first thing I do is plot the real (or in-phase) and imaginary (or quadrature) parts of the waveform and look for abnormalities.  What sort of abnormalities?  Hard to say.  Goood solid waveforms are smooth with low-amplitude ripples.  Any deviation from this behavior - like abrupt "glitches" - means there's a bug in one of your filter implementations.There isn't much you can do except look over your code, and plot as much data as you can stand.  I tend to plot the output of every processing block.

## Matched Filter

Once I'm satisfied with the appearance of the waveforms,  I start processing the samples like a digital receiver.  This begins with matched filtering.  A matched filter is designed to maximize signal-to-noise ratio.  Depending on the application, deriving a matched filter can take a lot of work.  In communications, it tends to be pretty straight forward if you assume a white noise channel.  I won't go into the math, but it turns out that you get the matched filter by time-reversing and complex conjugating the pulse-shaping filter.  In the last post, we used a root-raised cosine filter.  Because that filter is symmetric and has only real elements, the matched filter and pulse shaping filter are the same.

Now that I know what the matched filter is, I run the samples through it.  The input and output samples should look very similar.  If you plot them side by side, the only differences you'll see are

* The output is delay by half the length of the matched filter.
* The output has a more stable amplitude.


## Constellation Diagram

The next thing I do is pull out the *symbol peaks* from the matched-filter output and put them in a constellation diagram.  A symbol peak is the sample in a symbol period that has the biggest amplitude.   It's a bit misleading though, because it may not look like a peak relative to the entire waveform.  

Later on, when I start developing a legit receiver, I'll demonstrate how to efficiently pick off the symbol peaks with some control theory.  For now, all I'm going to do is figure out where the first symbol peak is, and grab every 16$^\text{th}$ sample until I hit the total number of symbols in the payload.  It's every 16$^\text{th}$ sample because the matched filter preserves the 16 samples/symbol used in the modulator design.

In a receiver, accurately estimating when the first symbol peak occurs can be challenging.

Once I get hold of the symbols, I plot the constellation diagram.  I'm looking for 4 small clusters.  How small?  The clusters should look like dots.  If the 4 clusters are too zuzzy, it's usually because my first symbol estimate is wrong.  Because of my naive method of selecting symbols based on constant spacing, when the first symbol is off, all the others are also slightly off.  As a result, you get a fuzzy constellation.  Fortunately, the fix is simple: adjust the index of the first symbol peak until the clusters shrink.   




## Bit Errors 


## The Code 

## Conclusion

