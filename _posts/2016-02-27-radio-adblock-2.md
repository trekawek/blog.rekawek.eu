---
title: Radio AdBlock 2
layout: post
---

In the [previous post](/2016/02/24/radio-adblock/) I described the process of creating an ad-blocker for the polish radio station "Trójka". It uses cross-correlation and fast Fourier transform to detect the ad jingles in the internet radio stream, silences the commercials and outputs the result. In other words: it plays the internet radio stream without ads. Creating the app wasn't my final goal, though. I wanted to bring the same feature to my home amplituner (Yamaha R-N301), which receives "Trójka" in a traditional way via the FM broadcast.

![Yamaha R-N301]({{ site.baseurl }}/assets/yamaha-rn301.jpg){:width="50%"}

## missing parts

So, I already have the algorithm, but I'm missing 3 elements required to run it in the real world of my flat:

1. a device on which I can run the analysis,
2. the broadcast, which will be used as a source for the analyzer,
3. a way to turn the volume up and down on the amplituner.

## the device

That's the easy one. I already have Raspberry Pi 2, which works great as a home media center. It should be able to run the Java analyzer.

## the broadcast source

The internet broadcast is delayed by about 30 seconds, so I can't use it to analyze the signal and silence the FM tuner. No, I should analyze the real-time FM broadcast. How to receive it on a Raspberry Pi? Well, it will cost about $20. There's a lot of [cheap DVB-T USB sticks](http://www.amazon.co.uk/s/ref=nb_sb_noss/277-5562518-0958039?url=search-alias%3Daps&field-keywords=dvb-t+usb) on the market and apparently (because of the [RTL-SDR](http://www.rtl-sdr.com/) library) they can be used to receive all kinds of signals, FM radio stations included.

After plugging-in the stick, following command will receive the broadcast on 89.5Mhz:

    rtl_fm -f 89.5M -M wbfm

The stream will be redirected to the standard output as a 16-bit, little-endian, 32k, 1-channel PCM data, perfect for the Java analyzer.

## controlling the amplituner

Yamaha calls R-N301 a "Network HiFi Receiver", which means (among other things) it can be controlled from an iPhone app. [Apparently](https://github.com/wuub/rxv), Yamaha network-enabled devices exposes a RESTful interface, which can be called with `curl`. Following command will set the volume level to 55:

    curl -d '<?xml version="1.0" encoding="utf-8"?><YAMAHA_AV cmd="PUT"><Main_Zone><Volume><Lvl><Val>55</Val><Exp>0</Exp><Unit></Unit></Lvl></Volume></Main_Zone></YAMAHA_AV>' 192.168.1.101/YamahaRemoteControl/ctrl

What's more, it's possible to get the current amplituner state, so we can launch the analyzer only if the tuner is enabled and plays the appropriate station.

## all together

I've used a few bash scripts to detect the current amplituner state, run the analyzer and turn down the volume during commercial blocks. Scripts can be found on [my github](https://github.com/trekawek/radioblock/tree/master/scripts).

## demo

This is how the adblocker works in practice. The terminal tails the analyzer log file:

<iframe width="640" height="360" src="https://www.youtube.com/embed/hmdEd4WlKE8?rel=0" frameborder="0" allowfullscreen></iframe>
