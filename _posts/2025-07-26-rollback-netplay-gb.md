---
title: Rollback netplay for Game Boy emulator
layout: post
---

In the [previous blog post](https://blog.rekawek.eu/2017/02/09/coffee-gb/), I described my work on the Nintendo Game Boy emulator, called [Coffee GB](https://github.com/trekawek/coffee-gb/). To this day, it remains one of my most beloved projects. Every year or two, I spend some time tweaking it, adding features or fixing compatibility with particularly stubborn ROMs.

One thing that I wanted to add for a long time is the netplay support. The real Game Boys can be connected to each other by a link cable and there’re multiple games supporting multiplayer this way. Even Tetris, bundled with the console in 1989, can be started in the 2-player mode \- both players are sent with the same bricks and whoever plays longer, wins.

The Game Boy link cable is a very simple medium – it's a serial cable, with 3 signal wires: TX, RX and clock (connected consoles need to negotiate which one is a clock source). It shouldn’t be a problem to join two emulators with a TCP connection and use it to send these signals, right?

<!--more-->

## Approach 1: naive wire-over-TCP approach

I started by creating the SerialEndpoint implementation that sends data through the network connection. It wasn’t that hard and it worked immediately\! I was able to start two emulators side-by-side, connect them through the loopback interface (no place like 127.0.0.1) and launch Tetris in the 2-player mode.

![Two emulators connected with TCP](/files/rollback-netplay-gb/wire-over-tcp.png)

<small>Connected Game Boys image: [https://www.ebay.com.au/itm/185131929721](https://www.ebay.com.au/itm/185131929721) (accessed 2025-07-28)</small>

Assured by this easy win, I tried to run emulators on two different machines in my home network. The result was… not so good? The game was unstable, there were some random characters on the screen, it wasn’t really playable. I tried to optimize the networking code, send data by bytes and not bits, connect through ethernet and not Wi-Fi, but it still didn’t work very well.

I assumed that the amount of data sent by the 30-year game is pretty small, so it shouldn’t be a problem to run it over a broadband connection, but apparently the issue wasn't throughput, but latency. The serial port in the classic Game Boy works with the clock frequency 8192 Hz. Therefore, it requires a single bit to be sent within 0.12 ms of latency. Even if we assume we can pack 8 bits together into a byte, we still require a network connection latency of \<0.97 ms. And for the faster Game Boy Color, the serial port clock frequency is 64 times higher (524288 Hz). Game Boy games weren’t programmed to handle any higher latency (why would they be?) and they weren’t really memory safe, so any delay in the incoming data results in memory corruption.

At first I thought that the emulator can run with any defined speed, so it can be simply slowed down whenever the transmission occurs. However, this would require CPU tick-by-tick synchronization between two instances and that’s a lot of ticks to synchronize (internal clock works with 4.19MHz speed), so at best, it would be a very laggy experience, if usable at all.

No, there must be a better way.

## Approach 2: let’s merge two Game Boys together

OK, here’s an idea. The link cable requires sub-millisecond latency, but what actually matters for the netplay are the button press events, and these can be easily sent with much higher latency. What if we emulate **both** Game Boys for each player and only send button presses between them? We’d create some kind of hybrid console, consisting of two Game Boys running the same game. One of them would receive button presses from the player, while the other would be controlled remotely. It’d also be hidden, so no screen emulation, no sound, it’d be only used as a source of serial link communication.

![Two GBs running in a single emulator](/files/rollback-netplay-gb/merged-gbs.png)

This way we no longer need to solve the sub-millisecond link problem. Instead we end up with something simpler, that has potentially already been solved for other emulators \- a two-player machine that needs to synchronize button presses. The fact that in our case this machine actually consists of two Game Boys, where one is hidden, doesn’t really change anything.

Implementing this was a bit hard from the software engineering point of view. From the beginning I tried to separate emulation logic from the IO (sound via Java Audio, video and joypad via Swing), but configuring two separate instances, where one received button presses from Swing (but also sends them through TCP) and the second receives them from TCP made the dependency graph a nightmare. I decided to get rid of all the ad-hoc event listeners and migrate to the EventBus pattern, which allowed me to **programmatically** manage the dependencies between particular components. It made the development much easier and I was finally able to test the idea.

I hoped that this would just work. I assumed that even if the button presses don’t synchronize *ideally* on the same CPU tick, it shouldn’t matter, because these are old games and precision on this level isn’t required. As you can tell from the length of this blog post \- I was wrong again.

So, 2-player Tetris worked, but it produced different tetrominoes for each player. This was strange and it got me thinking. Tetris blocks are chosen randomly, but how Game Boy can do anything random, if it’s fully deterministic? There’s no RTC in it, or any sensor that can read some kind of noise. The only non-deterministic thing is… user input - button presses. So the game must read button presses, count the number of ticks when this happens and treat the count as a random seed. But this means that the presses actually **must** be synchronized up to a single tick \- otherwise both players will be running a different game. Great, just great.

## Approach 3: time machine AKA rollback

OK, so we actually need to synchronize up to a single tick, otherwise we lose the deterministic execution, which is required for the netplay, as both emulators will run something different.

Maybe it doesn’t need to be a single tick. A single frame can be enough. We can only send button presses on the full frame and the user shouldn’t notice a difference. Game Boy runs with \~60 FPS, so both instances need to be synchronized up to 1/60 s. Easy peasy? Not really.

If one emulator is running slower, by a few frames, and it sends a button press, it’ll be too late to apply it on the second emulator \- it can’t go back in time\! Therefore both emulators should synchronize the execution of every frame, which would lead to a quite complex protocol. Also, it’d only work with the \<16ms latency, which is doable at home, but over the internet may be problematic and certainly won’t lead to a smooth experience.

However, synchronizing button presses between two emulators should be a solved problem, right? After all, many emulators offer smooth netplay. How are they doing it? I was asking myself (and Google) the same question until I found the video called [Netplay in RetroArch](https://www.youtube.com/watch?v=zkFUp7aQmFM). The video presents a great idea: what if we *can* go back in time after receiving a button press from the past?

This idea is called rollback netplay and it’s based on the emulator’s ability to create a snapshot of the internal state in every frame, together with the local button states. If we receive a remote button press from the past, we can just rewind emulation, apply the event and fast-forward again to the current frame, applying all local buttons pressed in the meantime. Usually we don’t need to go back too far \- just a few frames, so the whole operation is transparent to the user. If the remote event comes with a future frame id \- that’s fine too, we can fast-forward the local emulator state to that frame and just apply it.

![Game Boy event log](/files/rollback-netplay-gb/event-log.png)

This whole idea resembles how Git or any distributed database works. There’s a commit list (list of snapshots) and a remote commit, rebasing the history. Luckily, in this case we won’t have any conflicts, because the local player is fully in control of the Game Boy 1 and the remote player controls the (invisible) Game Boy 2 in our hybrid, two-console system. They only communicate via the local serial link (which also has a tiny state that needs to be rewound).

Rollback netplay requires fast snapshot and restore functions. Coffee GB already had snapshot support, but it was based on Java serialization, which is pretty slow. It was enough to save state before a hard end-level boss fight, but not enough to run on every frame. To make it better, I implemented the [Memento pattern](https://en.wikipedia.org/wiki/Memento_pattern) on every class containing emulation state: CPU, sound, GPU, timer, etc. This is much faster and doesn't require serialization or reflection.

Finally, after joining it all together and fixing a few bugs I ran Bust-a-Move Millennium (which is good for testing, because it displays both player screens at the same time) \- and it worked\! No garbage on the screen, no desynchronization, everything is working even on a high-latency network. I used [Toxiproxy](https://github.com/shopify/toxiproxy) to simulate latency around 120 ms and Bust-a-Move was still playable.

There were a few loose ends, like battery save support (these need to be transmitted too, before the emulation starts), but finally after a few days I was able to get it to the desired state. I’m quite happy with the result. I’m already thinking about a feature that I can add in the next 8 years\!

<iframe width="672" height="378" src="https://www.youtube.com/embed/X2ykd3_oGuw?si=SOdvhfIKXYTNGlPJ&amp;start=0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
