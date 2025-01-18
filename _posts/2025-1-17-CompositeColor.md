---
layout: post
title:  "Composite Color Video via IQ modulation"
description: Color Composite video generation technique
date:   2025-01-17
---


# Color Composite Video via IQ modulation

## Synopsis 

While no longer very popular today, composite video (best known by the "yellow" RCA jack) is commonly used by older VHS players and retro computers. While spending time within the retrocomputing community, I was intrigued by homebrew computer methods of video generation. 

VGA is very popular as its quite easy to generate and monochrome composite is quite popular too. Most televisions or monitors are pretty lenient on the signals that they require to display *something*, so hacking together that works is not too difficult (take for example the Sinclair ZX80 or 81 which clocks out video signals from a shift register). The signals lack many parts to the standard composite waveform, though, so they may not necessarily be very pretty, stable, or work on picky monitors. 

There's plenty of information on how monochrome composite works that can definitely explain it better than I can, so I will forgo that. I found many interesting and creative ways that people have generated color composite video, but they have some drawbacks. I'll show an useful way to do color composite video generation with few components for homebrewers that should be reliable against temperature variation, component variation etc... Despite having not tested and built this (yet, hopefully soon!), I will showcase it. I am most definitely not the first person who conceptualized this way, but I cannot find any resources online which showcase this in a easy to consume way. 

## Color composite video signal


Composite video signals can come in a few general flavors depending on your region... there are some subtypes for specific regions (NTSC, PAL, SECAM). I'll mostly be focusing on NTSC systems and briefly on PAL systems. SECAM color generation is done much differently and is a whole other situation on its own. 

Generally, composite color signals are decomposed into hue, saturation, and luminance/brightness. The color component signals (hue and saturation) are superimposed on the monochrome luminance signal. The color signals are represented by a high frequency waveform called the color carrier, and the monochrome/luminance signal is the average offset of this high frequency waveform. The phase of the color carrier governs the hue, and the amplitude governs the saturation. In NTSC, the frequency is fixed at ~3.579 MHz. Thus, this signal can be represented as the mathematical sum of the color waveform and the monochrome waveform. 



![image of color composite waveform](https://i.sstatic.net/cIQVf.png)

This image is from [EDN magazine](https://www.edn.com/measuring-composite-video-signal-performance-requires-understanding-differential-gain-and-phase-part-1-of-2/). 

These signals are designed to interfere with each other as little as possible (with varying degrees of success, see [dot crawl](https://en.wikipedia.org/wiki/Dot_crawl) and [artifact colors](https://en.wikipedia.org/wiki/Composite_artifact_colors)) so that color signals can be ignored by monochrome televisions for backwards compatibility.

NTSC and PAL signals both use Quadrature Amplitude Modulation (QAM). All this means is that the color signals are represented by a wave (color carrier) where the phase and amplitude of this wave encodes the hue and saturation, respectively. Since phase is relative, it is always compared to the colorburst.  [This](https://sagargv.blogspot.com/2011/04/ntsc-demystified-part-2-color-encoding.html) site is really quite informative and has some nice images. 

The generation of monochrome signals is quite widely available elsewhere, such as in the TV typewriter cookbook. I will mostly be focusing on color video generation. 

## One generation method via delay line

Since this QAM signal requires some variable phase shift of the color carrier to encode a hue, a very popular method in generating this color is using a tapped delay line. This color carrier is then added to the luminance waveform (typically with a few resistors)

A delay line imparts a small amount of delay and thus phase shift on the input signal, and multiple delay lines are strung together. This creates many copies of the input signal with various phase shifts. Then, the signal with the correct phase shift for a given color can be selected from via some multiplexer. 

A simple delay can be implemented using inverters. Each inverter has some propagation delay and by stringing many buffers/inverters together, a tapped delay line can be made. These digital gates can only accept square wave inputs and outputs, but these square waves can be filtered using a low pass filter to achieve an acceptable sine wave-like color carrier. 

The [TV Typewriter Cookbook](https://www.tinaja.com/ebooks/tvtcb.pdf) shows an example of this method in Chapter 8: 
![pg 206 TV Typewriter Cookbook](https://cx237.github.io/CadenXu/images/TVTCookbook.png)


The delay of the inverters though, unfortunately, are pretty dependent on PVT: chip (**P**rocess), supply voltage (**V**oltage), and **T**emperature. Thus, replacing the 4050 hex with a 74 series part may yield much different results. Each 4050 hex buffer has different delay too, looking at the datasheet. 

![4050 datasheet excerpt from TI](https://cx237.github.io/CadenXu/images/4050pd.png)

[Datasheet](https://www.ti.com/lit/ds/symlink/cd4049ub.pdf) excerpt from Texas Instruments

These parts have a typical propagation delay and a maximum propagation delay specified for each part, and the actual delay may be anything in between. Furthermore, the propagation delay is evidently dependent on supply voltage. This means that depending on part and power supply the output color can be different. This is undesirable. 

Additionally, if you wanted a larger gamut of colors, you'd need more delays, more multiplexors, and each delay would have to be shorter. Consequently, each delay would have to be more accurate and stable, making this design less practical for a design that needs, say, 256 colors.

## Another generation method using IQ modulation

### Basically, what is IQ modulation

 A hand-wavey mathematical explanation is that any sinusoidal wave (at any phase and amplitude) of a single frequency can can be composed of a sine wave of that single frequency summed with a cosine wave of that single frequency. By controlling the amount of cosine (**I**n-phase) and sine (**Q**uadature) components you add, you control the amplitude of the output waveform and the phase of the output waveform. People may be familiar that this is just the polar representation of a rectangular complex sinusoid. [Here's](https://www.desmos.com/calculator/jjnpxndodz) a quick demo in Desmos for a visual. 

```math
$$
A(t)\cdot \cos({\omega}t +\phi)=A_{cos}(t)\cdot \cos({\omega}t) + A_{sin}(t)\cdot \sin({\omega}t) 
$$
```

For those who like a practical analogy, think of a water faucet with a hot water control and a cold water control. You can change the flow rate and temperature of the faucet (amplitude) and the temperature (phase) by altering both the hot tap and the cold tap. 

This ties back to composite video generation as our goal is just generating a waveform a defined phase and amplitude. Now how do you do this in electronics and why may this be better?

### Generation of sine and cosine

First of all, we will need to generate sine and cosine components. Since a sine is a 90 degree offset from a cosine via the relation:

```math
$$
\cos({\omega}t) = \sin({\omega}t+90\degree) 
$$
```
We need circuitry that performs 90 degree phase shifts.

Doing a 90 degree phase shift (AKA Hilbert transform) that works for all frequencies (wideband) is quite annoying in electronics. This [site](https://markimicrowave.com/technical-resources/application-notes/top-7-ways-to-create-a-quadrature-90-phase-shift/) has a few ways, some of them quite complex. Fortunately, we need to do so for only one frequency (the color carrier at 3.579 MHz). This can be done with some tuned RC or RLC filter networks. Even better, most TVs can tolerate filtered square waves in lieu of sinusoids. We can generate a 3.579\*2 MHz square wave and do phase shifting using a well-known design using only a few flip flop chips. 


![Sine and cosine generator](https://markimicrowave.com/assets/7737bbd5-a3e1-4097-a27e-c0bd1a38be5b)

![waveforms](https://markimicrowave.com/assets/dabbdc2e-1e26-4db0-9c1e-24f9cabe531b)
images from Marki-Microwave


### Controlling sine and cosine amplitude

To control the sine and cosine amplitude, we need to multiply the sine-like square waves with some value and the cosine-like square waves with a digital value, controlled by circuitry within a homebrew computer. We could either build two analog multipliers and two DACs to do this task, or we can build a multiplying DAC (MDAC).

Preliminarily, I tried a design like this. After fiddling with falstad circuit simulator for a while, I created a [MDAC design](https://tinyurl.com/22ompjze) and strung two of them together. Notice that our MDAC has to perform two quadrant multiplication. This means that our output has to swing between $`+\sin({\omega}t)`$ and $`-\sin({\omega}t)`$, or $`+\cos({\omega}t)`$ and $`-\cos({\omega}t)`$.

Intuitively, this means the output has of each MDAC has to be able to "flip" the signal depending on the digital input. This is such that we can get all 360 degrees of phase shift (refer back to the desmos demo, notice that we need to multiply by a negative number to get all phase shifts).


![preliminary MDAC design](https://cx237.github.io/CadenXu/images/falstadMDAC1.png)

This design uses the inverting and noninverting outputs of the flip flops to drive an R2R DAC. The R2R DAC selects between the inverting copy and noninverting copy to be able to do the 4 quadrant multiplication.  Here, I've opted to use 4.7kOhm instead of 5kOhm resistors to stick with the [E preferred numbers](https://en.wikipedia.org/wiki/E_series_of_preferred_numbers).  The outputs have DC bias to them, but this can be removed easily with a capacitor.  

Although this design works, it could be optimized further. Since our sines and cosines are already "digital" as in square waves, we can stay in digital-land and use XOR gates to do our inversion. [Here](https://tinyurl.com/2avukhhv) is a design that does that. This makes things to implement than analog switches. The commonly shown truth table of XOR gates is of following:

| A   | B   | Y   |
| --- | --- | --- |
| 0   | 0   | 0   |
| 0   | 1   | 1   |
| 1   | 0   | 1   |
| 1   | 1   | 0   |

Notice that when $`A = 0, B = Y`$, and when $`A = 1, B = \lnot Y`$. This means that rather than selecting between the noninverting and inverting outputs of the flip flops, we can choose to invert or not using a XOR gate. This makes things a bit cleaner to implement in hardware. 

![preliminary MDAC design](https://cx237.github.io/CadenXu/images/falstadMDAC2.png)

This poises us to add the sine and cosine components, or mix the hot and cold waters, however you want to look at it. 


### Combining sines and cosines

This can simply be done with a few resistors between the in-phase and quadrature outputs. I won't go too far into the exact implementation of this, since your monochrome video generator has its own implementation, and thus its own impedances and voltage levels. One thing to note is that you may want to filter the output at this point to make it more sine-ish. Again, your exact filter implementation will have to take into consideration your monochrome circuitry. 


### Encoding colors

Essentially, rather than the color circle defined in Sagar's blog which shows the correspondence between color, phase, and amplitude (in polar coordinates, for those who are mathematically minded):
![polar color chart](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhCWyjWQRex64g7FzAwxHqrN9kcJgg27D8pnsN0kXbH2kYCYC0pvBiElhzfF30fv_exvVVw2dGEgq4wphFyrf6uEhwjA4g8p6xW3bcl5BPDGblce0zoXm8mIIg0U4UkDPXxWOe1xSjSOMNm/s1600/NTSC_vectorograph.png)

from [Sagar's blog](https://sagargv.blogspot.com/2011/04/ntsc-demystified-part-2-color-encoding.html)

We use one that is rectangularly defined in terms of how much sine and cosine component there is. This color space is known as YIQ and should be easily encoded in a lookup table or a ROM.

![YIQ plane](https://upload.wikimedia.org/wikipedia/commons/8/82/YIQ_IQ_plane.svg)

from [Wikimedia](https://commons.wikimedia.org/wiki/File:YIQ_IQ_plane.svg)
### Benefits and some other notes

The goal of this configuration is to be less sensitive to component variations. Resistors cheaply and commonly come in 5% tolerances, and 1% tolerances, so your output phase and amplitude should be much more stable. As long as [CMOS output](https://www.allaboutcircuits.com/textbook/digital/chpt-3/logic-signal-voltage-levels/) XOR gates are used, this design should work well, so it should be compatible with CD4000 series, 74HC, HCT, AC, you name it!

Furthermore, this design scales exponentially for colors as using 2 74HC136 chips (4x XOR gates) should theoretically give you $2^{4} *2^{4} = 256$ colors, although your actual performance may vary due to resistor tolerances. 

On a retrocomputing sidenote, this IQ modulation technique may have been used in the VIC chips from Commodore ! Al Charpentier, credited in the invention of the VIC-I and VIC-II published this patent back in 1983: https://patents.google.com/patent/US4551682A/en. This outlines a "sine-cosine generator is provided for use in color television signal generators"... this probably is for IQ modulation!

