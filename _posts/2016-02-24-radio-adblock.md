---
layout: post
title: Radio AdBlock
---

Polish Radio Three (so-called Trójka) is famous for broadcasting good music and
having non-offensive speakers. On the other hand it suffers from the number
of commercial blocks between auditions. The ads, usually related to drugs
or electronics are loud and irritating. Trójka accompanies me in home and
the work for most of the time, so I wondered if there's something than can
be done about the ads. It seems that there is.

## digital signal processing

The commercial block starts and finishes with two jingles, so the potential
software that mutes the ads should recognize these specific sounds and
turn off the volume between them. I knew that the area of maths/computer
science that deals with these kinds of problems is called *digital signal
processing*, but it always seems like a magic for me. I spend a day or two
trying to find what mechanism can be used to analyze an audio stream looking
for a jingle. And I found it, eventually - it's called cross-corellation.

## Octave

People usually describes the cross-corellation referring to the MATLAB
implementation. MATLAB is an expensive application that makes it easy to
perform complex mathematical operations, including DSP operations. Fortunatelly,
there's an free alternative to MATLAB, called
[Octave](https://www.gnu.org/software/octave/). It seems it's quite
easy to run cross-corellation on two audio files (so called *samples*) using
Octave. All you have to do is to run following commands:

    pkg load signal
    signal = wavread('jingle.wav')(:,1);
    enhance = wavread ('audio.wav')(:,1);
    [R, lag] = xcorr(signal, enhance);
    plot(R);

it'll result in following plot:

![octave plot](./assets/octave-result.png){:width="50%"}

As you can see, there's a peek describing the position of the `jingle.wav`
in the `audio.wav`. What suprised me is the simplicity of the method - the
`xcorr()` makes all the work, the rest of the Octave code is for reading files
and displaying the result.

## reverse-engineering

I wanted to reimplement the same algorithm in Java, so I'll have a tool that:

1. reads the audio stream from standard input (eg. provided by ffmpeg),
2. analyses it looking for the jingles,
3. prints the analysis results on standard output.

Using stdin and stdout will allow to connect the new *analyzer* with other
apps, responsible for providing audio stream and turning down the volume.

In order to do this I opened the `xcorr()` function [source code](https://sourceforge.net/p/octave/signal/ci/default/tree/inst/xcorr.m). The crucial lines:

    pre  = fft( postpad( prepad( X(:), length(X)+maxlag ), M) );
    post = fft( postpad( Y(:), M ) );
    cor = ifft( pre .* conj(post) );
    R = cor(1:2*maxlag+1);

It looks quite scary, but most of the functions are trivial array operations.
The heart of the cross-corellation is applying the *fast Fourier transform* on
the sound sample.
