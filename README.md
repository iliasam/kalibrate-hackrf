# kalibrate-hackrf

Kalibrate, or `kal`, can scan for GSM base stations in a given frequency band and
can use those GSM base stations to calculate the local oscillator frequency
offset.

Kalibrate-hackrf is a fork of Joshua Lackey's kalibrate originally for USRP, ported
to support the [HackRF](https://greatscottgadgets.com/hackrf/). 

The documentation below is adapted from the original website at
[http://thre.at/kalibrate](https://web.archive.org/web/20131226204943/http://thre.at/kalibrate)
(defunct since 2013).
  
**By iliasam:** this is pathched variant of this utility: https://github.com/scateu/kalibrate-hackrf/issues/6#issuecomment-2111687317  


## On Clocks

SDRs have a built-in
[oscillator](http://en.wikipedia.org/wiki/Electronic_oscillator) which is only
guaranteed accurate to within some range (for example, the USRP1 to 20 parts per million (ppm). (Although in practice
the USRP oscillator is normally closer to 3ppm.) This is sufficient for a number
of purposes, but not for others. Some SDRs (including the HackRF and USRP) also support external
oscillators. For example, the [Clock Tamer](http://code.google.com/p/clock-tamer/),
the [KSP 52MHz](https://web.archive.org/web/20100518194722/http://kestrelsignalprocessing.mybigcommerce.com/products/52MHz-clock-generator.html)
and the [FA-SY 1](https://web.archive.org/web/20101221022021/http://www.box73.de/catalog/product_info.php?products_id=1869)
are all accurate clocks.

Normally, these external clocks must be calibrated to ensure they are set as
accurately as possible. An oscilloscope is probably the best way to determine
the accuracy of these clocks. However a good oscilloscope is expensive and not
everyone has access to one. There are other ways to calibrate these devices --
the FA-SY 1 documentation discusses using the [WWVB](http://en.wikipedia.org/wiki/WWVB)
time signals. Similarly, GSM base stations are required to be accurate to within
0.05ppm and so they can also provide an accurate reference clock.

If you own a USRP daughterboard that covers the GSM frequencies in your area,
using a GSM base station is a particularly convenient.

## Description

A base station transmits a frequency correction burst on the Frequency Correction
CHannel (FCCH) in regular positions. The FCCH repeats every 51 TDMA frames and the
frequency correction burst is located in timeslot 0 of frames 0, 10, 20, 30, and 40.

A frequency correction burst consists of a certain bit pattern which, after
modulation, is a sine wave with frequency one quarter that of the GSM bit rate.
I.e., (1625000 / 6) / 4 = 67708.3 Hz. By searching a channel for this pure tone,
a mobile station can determine its clock offset by determining how far away from
67708.3Hz the received frequency is.

There are many ways which a pure tone can be identified. For example, one can
capture a burst of data and then examine it with a Fourier transformation to
determine if most of the power in the signal is at a certain frequency.
Alternatively, one can define a band-pass filter at 67708.3 Hz and then compare
the power of the signal before and after it passes through the filter. However,
both of these methods have drawbacks. The FFT method requires significant resources
and cannot easily detect the frequency correction burst edge. The filter method
either isn't accurate or cannot detect larger offsets. Multiple filters can be
used, but that requires significant resources.

`kal` uses a hybrid of the FFT and filter methods. The filter `kal` uses is an
adaptive filter as described in the paper:

G. Narendra Varma, Usha Sahu, G. Prabhu Charan, _Robust Frequency Burst Detection Algorithm for GSM/GPRS_.

An Adaptive Line Equalizer (ALE) attempts to predict the input signal byadapting
its filter coefficients. The prediction is compared with the actual signal and the
coefficients are adapted so that the error is minimized. If the input contains a
strong narrow-band signal embedded in wide-band noise, the filter output will be
a pure sine at the same frequency and almost free of the wide-band noise.

`kal` applies the ALE to the buffer and calculates the error between the ALE prediction
and the input signal at each point in the signal. `kal` then calculates the average
of all the errors. When the error drops below the average for the length of a
frequency correction burst, this indicates a detected burst.

Once we have the location of the frequency correction burst, we must determine what
frequency it is at so we can calculate the offset from 67708.3 Hz. We take the
input signal corresponding to the low error levels and run that through a FFT.
The largest peak in the FFT output corresponds to the detected frequency of the
frequency correction burst. This peak is then used to determine the frequency offset.

Any noise in the system affects the measurements and so `kal` averages the results
a number of times before displaying the offset. The range of values as well as the
standard deviation is displayed so that an estimate of the measurement accuracy
can be made.

## How

`kal` requires fftw3 and version 3.2 or higher of libusrp. `kal` also requires a
USRP and daughterboards appropriate for the desired GSM frequency band. An external clock is not required; `kal` can also calculate the offset of the built-in USRP clock.

`kal` uses the GNU Autoconf system and should be easily built on most *nix platforms.

		jl@thinkfoo:~/src$ cd kal-v0.4.1
		jl@thinkfoo:~/src/kal-v0.4.1$ ./bootstrap && CXXFLAGS='-W -Wall -O3' ./configure && make
		[...]
		jl@thinkfoo:~/src/kal-v0.4.1$ src/kal -h
		kalibrate v0.4.1, Copyright (c) 2010, Joshua Lackey

		Usage:
			GSM Base Station Scan:
				kal <-s band indicator> [options]

			Clock Offset Calculation:
				kal <-f frequency | -c channel> [options]

		Where options are:
			-s	band to scan (GSM850, GSM900, EGSM, DCS, PCS)
			-f	frequency of nearby GSM base station
			-c	channel of nearby GSM base station
			-b	band indicator (GSM850, GSM900, EGSM, DCS, PCS)
			-R	side A (0) or B (1), defaults to B
			-A	antenna TX/RX (0) or RX2 (1), defaults to RX2
			-g	gain as % of range, defaults to 45%
			-F	FPGA master clock frequency, defaults to 52MHz
			-v	verbose
			-D	enable debug messages
			-h	help
		jl@thinkfoo:~/src/kal-v0.4.1$ src/kal -s 850
		kal: Scanning for GSM-850 base stations.
		GSM-850:
			chan: 128 (869.2MHz -  13Hz)	power: 9400.80
			chan: 131 (869.8MHz +   7Hz)	power: 4081.75
			chan: 139 (871.4MHz +  10Hz)	power: 2936.20
			chan: 145 (872.6MHz -   7Hz)	power: 33605.48
		jl@thinkfoo:~/src/kal-v0.4.1$ src/kal -c 145
		kal: Calculating clock frequency offset.
		Using GSM-850 channel 145 (872.6MHz)
		average		[min, max]	(range, stddev)
		-   1Hz		[-8, 7]	(14, 3.948722)
		overruns: 0
		not found: 0


## ChangeLog

2016-6-6 Update:

Fix library not found for -lrt on OS X. 

Author: @rxseger 

2015-3-3 Update:

Add ability to specify channel when scanning to check power output for a single channel.

Author: [xorrbit](https://github.com/xorrbit)

2014-8-23 Update:

Thanks to chris.kuethe@gmail.com who did a bunch of tinkering with this port of kalibrate.

https://github.com/ckuethe/kalibrate-hackrf/

This now runs as fast as kalibrate-rtl

v0.4.1\. This version contains the following fixes:

*   Support for Mac OS X (i.e., support for POSIX shared memory rather than just SysV shared memory)
*   Specifying the FPGA master clock frequency actually works now.
*   Some constants subtly depend on the sample rate. I've removed the decimation option and now calculate that automatically.

The latest version should be the most accurate. I'd urge anyone running an old version to update.

Versions of `kal` previous to 0.4 used an algorithm that was extremely sensitive to noise. Version 0.4 of `kal` is more processor intensive, but also much more accurate.


## Credits

I'd like to thank Alexander Chemeris for his valuable input, ideas, and his testing of a large number of alpha versions. I'd also like to thank Mark J. Blair for his help getting `kal` to work with Mac OS X.

I'd also like to thank Mark J. Blair for his help getting `kal` to work with Mac OS X. (And also for helping me realize that I never actually used the `-F` parameter...)

Copyright (c) 2010, Joshua Lackey ([jl@thre.at](mailto:jl@thre.at))

Copyright (c) 2010, Joshua Lackey (jl@thre.at)

Copyright (c) 2012, Steve Markgraf (steve@steve-m.de) 


modified for use with hackrf devices,

Copyright (c) 2014, Wang Kang (scateu@gmail.com) 

http://hackrf.net  : A Chinese HackRF Community


## Windows pre-compiled version

Thanks to ikaya1965:
 - <https://github.com/ikaya1965/-smail-Kaya>

