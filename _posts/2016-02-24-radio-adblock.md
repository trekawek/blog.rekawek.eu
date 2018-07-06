---
title: Radio AdBlock
layout: post
---

Polish Radio Three (so-called Trójka) is famous for broadcasting good music and having non-offensive speakers. On the other hand it suffers from the number of commercial blocks between auditions. The ads, usually related to drugs or electronics are loud and irritating. Trójka accompanies me in home and at work for most of the time, so I wondered if there's something than can be done about the ads. It seems there is.


## digital signal processing

My aim is to create an app that mutes the ads. The commercial block starts and finishes with a jingle, so the potential software should recognize these specific sounds and turn off the volume between them.

<iframe width="100%" height="116" scrolling="no" frameborder="no" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/248847014&amp;color=ff5500&amp;auto_play=false&amp;hide_related=true&amp;show_comments=false&amp;show_user=false&amp;show_reposts=false&amp;liking=false&amp;sharing=false&amp;show_artwork=false"></iframe>

<iframe width="100%" height="116" scrolling="no" frameborder="no" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/248847022&amp;color=ff5500&amp;auto_play=false&amp;hide_related=true&amp;show_comments=false&amp;show_user=false&amp;show_reposts=false&amp;liking=false&amp;sharing=false&amp;show_artwork=false"></iframe>

I know that the area of maths/computer science dealing with these kinds of problems is called *digital signal processing*, but it always seems like a magic for me - so this is a great opportunity to learn something new. I spend a day or two trying to find out what mechanism can be used to analyze an audio stream looking for a jingle. And I found it, eventually - it's called *cross-correlation*.

<!--more-->

## Octave

People usually describes the cross-correlation referring to the MATLAB implementation. MATLAB is an expensive application that makes it easy to perform complex mathematical operations, including DSP operations. Fortunatelly, there's a free alternative to MATLAB, called [Octave](https://www.gnu.org/software/octave/). It seems it's quite easy to run cross-correlation on two audio files using Octave. All you have to do is to run following commands:

    pkg load signal
    jingle = wavread('jingle.wav')(:,1);
    audio = wavread ('audio.wav')(:,1);
    [R, lag] = xcorr(jingle, audio);
    plot(R);

it'll result in following plot:

![octave plot]({{ site.baseurl }}/assets/octave-result.png){:width="50%"}

As you can see, there's a peek describing the position of the `jingle.wav` within the `audio.wav`. What suprised me is the simplicity of the method - the `xcorr()` makes all the work, the rest of the Octave code is just for reading the files and displaying the result.

I wanted to reimplement the same algorithm in Java, so I'll have a tool that:

1. reads the audio stream from standard input (eg. provided by ffmpeg),
2. analyses it looking for the jingles,
3. outputs the same stream on stdout and/or muting it.

Using stdin and stdout will allow to connect the new *analyzer* with other apps, responsible for providing audio stream and playing the result.

## reading sound files

The first step of the Java implementation is to read the jingle (saved as a `.wav` file) into an array. `.wav` file contains some extra info, like headers, metadata, etc. while we need the raw data. The format I was looking for is PCM - it simply contains the list of numbers representing sounds. Converting a wav to PCM can be done using `ffmpeg`:

    ffmpeg -i input.wav -f s16le -acodec pcm_s16le output.raw

In this case each sample will be saved as 16-bit number, little endian. In Java such number is called `short` and the [`ByteBuffer`](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) class may be used to automatically transform the input stream into a list of `short` values:

    ByteBuffer buf = ByteBuffer.allocate(4);
    buf.order(ByteOrder.LITTLE_ENDIAN);
    buf.put(bytes);
    short leftChannel = buf.readShort(); // stereo stream
    short rightChannel = buf.readShort();

## xcorr reverse-engineering

In order to implement the `xcorr()` function in Java, I looked into the Octave's [source code](https://sourceforge.net/p/octave/signal/ci/default/tree/inst/xcorr.m). Without changing the final result, I was able to replace the `xcorr()` invocation with the following lines - they need to be rewritten to Java:

    N    = length(audio);
    M    = 2 ^ nextpow2(2 * N - 1);
    pre  = fft(postpad(prepad(jingle(:), length(jingle) + N - 1), M));
    post = fft(postpad(audio(:), M));
    cor  = ifft(pre .* conj(post));
    R    = real(cor(1:2 * N));

It looks quite scary, but most of the functions are trivial array operations. The heart of the cross-correlation is applying the *fast Fourier transform* on the sound sample.

## fast Fourier transform

As someone who didn't have earlier experience with DSP, I've simply treated the FFT as a function that takes the array describing the sound sample and returns an array containing complex numbers representing the frequencies. This minimalistic approach worked well - I was able to run the FFT implementation from the [JTransforms](https://github.com/wendykierp/JTransforms) package and got the same results as in Octave. I guess there's a bit of [cargo cult](https://en.wikipedia.org/wiki/Cargo_cult) here, but hey - it works!

## running xcorr on a stream

As you may see, the algorithm above assumes that the `audio` is an array, in which we are looking for the `jingle`. That's not exactly the case for the radio broadcast, where we have a continous stream of sound. In order to run the analysis, I created a round-robin buffer, slightly longer than the jingle I'm looking for. The incoming stream fills the buffer and once it's full, I run the cross-correlation test. If there's nothing found, I discard the oldest part of the buffer and then wait until it's full again.

I experimented a bit with the buffer length and got the best results with buffer 1.5 times bigger than the jingle size.

## putting it all together

Getting the stream in PCM format is easy and can be done using aforementioned `ffmpeg` - the command below redirects the stream into the `java` standard input and then outputs `Got jingle 0` or `Got jingle 1` when the respective sample was found in the stream.

```bash
ffmpeg -loglevel -8 \
       -i http://stream3.polskieradio.pl:8904/\;stream \
       -f s16le -acodec pcm_s16le - \
  | java -jar target/analyzer-1.0.0-SNAPSHOT-jar-with-dependencies.jar \
    2 \
    src/test/resources/commercial-start-44.1k.raw 500 \
    src/test/resources/commercial-end-44.1k.raw 700
```

## standalone version

I also prepared a simple standalone version of the analyzer, that connects to the Trójka stream on its own (without an external `ffmpeg`) and plays the result using `javax.sound`. The whole thing is a single JAR file and contains a basic start/stop UI. It can be downloaded here: [radioblock.jar]({{ site.baseurl }}/files/radioblock-1.3.1.jar). If you feel uneasy about running a foreign JAR on your machine (like you should do), all the sources can be found on my [GitHub](https://github.com/trekawek/radioblock).

![Radioblock]({{ site.baseurl }}/assets/radioblock-swing-ui.png)

Apparently, it works :)

<iframe width="100%" height="116" scrolling="no" frameborder="no" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/248995303&amp;color=ff5500&amp;auto_play=false&amp;hide_related=true&amp;show_comments=false&amp;show_user=false&amp;show_reposts=false&amp;liking=false&amp;sharing=false&amp;show_artwork=false"></iframe>

## further work

The final goal is to mute ads on a hardware amplituner, receiving a "real" FM signal rather than some internet streams. This will be covered in the [next blog post](/2016/02/27/radio-adblock-2/).

## update (June 2018)

* [Hacker News discussion](https://news.ycombinator.com/item?id=17385563)
* [Wykop discussion](https://www.wykop.pl/link/4385801/radio-adblock/)
* [reddit discussion](https://www.reddit.com/r/programming/comments/8to4wm/radio_adblock/)
* [Russian translation @ habr](https://habr.com/post/415469/)
