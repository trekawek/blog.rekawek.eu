---
title: Macintosh System 6 and 7 images for IOmega ZIP
layout: post
---

As described in the [previous post](/2016/12/08/mac-plus/), I'm using SCSI IOmega ZIP (Z100S) drive as a mass storage for my Macintosh Plus. I prepared zip floppies with a few versions of the Macintosh System installed - if you're into retro computing it's fun to compare the performance and the features between different OSes.

I dumped the zip floppies into disk images, so they can be easily restored. All you need is an USB zip drive (Z100U) connected to your modern computer (Macbook Pro in my case). Images can be downloaded below:

* [System 6.0.8](/files/macos/system608-zip.image.gz)
* [System 7.0.1](/files/macos/system701-zip.image.gz)
* [System 7.1](/files/macos/system710-zip.image.gz)
* [System 7.5.5](/files/macos/system755-zip.image.gz)

<!--more-->

Besides from the system, images contain a fair amount of abandonware tools and games (MacWrite, MacPaint, MS Excel & Word, StuffIt, ZTerm, Civilization, Lemmings, etc.)

If you use OS X / Linux, the images can be written to the zip floppies using the `dd` command:

    gzip -dc system608-zip.image.gz | dd of=/dev/diskX

where /dev/diskX is the appropriate device. On the OS X it's a good idea to disable the automount feature, as the OS may corrupt the old HFS filesystem once it's written. The [Disk Arbitrator](https://github.com/aburgh/Disk-Arbitrator/releases) can be used for this purpose.

The images can be also used with emulators (like vMac). It's only required to extract the HFS filesystem (as the image also contains the IOmega drive, partition table which confuses the emulator):

    gzip -dc system608-zip.image.gz | dd bs=512 skip=491 of=system608-hfs.image

The resulting `system608-hfs.image` is an emulator-bootable image:

![System 7.5.5 on Mac Plus](/assets/mac/system755.png)

Basilisk II support using the disk images directly, but it'll only work with System 7.5.5.

I also prepared the same images in the native MacOS format (.hqx + DiskCopy 6):

* [System 6.0.8](/files/macos/system608-diskcopy.img.hqx)
* [System 7.0.1](/files/macos/system701-diskcopy.img.hqx)
* [System 7.1](/files/macos/system710-diskcopy.img.hqx)
* [System 7.5.5](/files/macos/system755-diskcopy.img.hqx)

## Update (February 2020)

People on the internet [noticed](https://www.reddit.com/r/VintageApple/comments/c8jzac/request_make_a_dsk_image/esnn3wv/) that the images are infected by the nVIR virus. I'm sorry for that, I must have got it with some vintage archives when I've been compiling these images. I've used Disinfectant to remove the virus and I've rebuilt all the files (`.image.gz` and `.img.hqx`).

Also, there's a very interesting blog post about preparing disk images on modern environment: [Setting up your vintage (classic) 68k Macintosh: using a 100MB SCSI Iomega Zip Drive](http://www.savagetaylor.com/2017/12/31/setting-up-your-vintage-classic-68k-macintosh-using-a-100mb-scsi-iomega-zip-drive/).
