# MultiJACK

The purpose of this project is to increase available audio DSP power available to JACK within a single multicore motherboard, using multiple JACK processes in concert, connected via IP transport of some sort.

## History and Theory

This project began shortly after I noticed that I was using 75% of JACK DSP capability, while pushing an octocore AMD X3 with 8G RAM to only about 25% CPU, fairly evenly spread out across all cores, including a large amount of audio synthesis as well as soundfont rendering.

There have been many failed attempts so far.  Noteworthy JACK devs have suggested publicly that it cannot work at all.

Based on both input and experience, it is suggested that NET and NETONE cannot do this at all, because the whole design of JACK is to be as absolutely timing-synchronized as possible: single JACK, NET, and NETONE all behave similarly, not leaving any wiggle-room (CPU and I/O cycle flexibility) left over.  NET and NETONE work nicely with multiple motherboards connected by Ethernet, again building effectively one big lockstepping JACK tarantula with legs in each motherboard, and this is a great way to combine the horsepower of multiple motherboards; and I have thought about buying four or ten Raspberry Pis; but my application needs major CPU for synthesis and rendering and requires portability.  So.  It does appear, that the wiggle room we need, may have to exist in the form of resampling.

Traditional JACK lore has it that resampling is terrible, awful, quality-destroying tarantula-poison.  However, Fons Adriaensen  and his Zita project:

http://kokkinizita.linuxaudio.org/linuxaudio/index.html

have long been bucking traditional trends, and have been increasing our options for maximum quality for quite a while now.  And 
zita-njbridge appears to be a method quite close at hand, to give us (a) IP transport between JACK servers along with (b) synchronization-independence, with Zita-class resampling for our wiggle room.

## Current Structure

At the moment an initial foundation exists and is working.  This is not in production condition, I will be working on some Python scripting for efficient automatic startup and control, but this will not play ball with PulseAudio or Cadence for the known future, major work on them would be required.  I do use qjackctl to discover and prove jackd command lines and latency settings, but only for this, because it too is not designed for this purpose.

### Script HARD

HARD starts the JACK server connected to real audio hardware.  I have a USB stereo audio device in use, whose ALSA name (see /proc/asound/cards) is "Device", thus:

  #/bin/bash
  /usr/bin/jackd -nHARD -dalsa -r48000 -p512 -n3 -Xraw -D -Chw:Device -Phw:Device
  
### Script SOFT

SOFT starts the first JACK server not connected to audio hardware.  It runs the dummy driver.  This script is invoked with a single command-line option, which is the soft server number.  zita-njbridge supports a theoretical maximum of 64 total servers, which means 1 hard and 63 soft.  It is unclear whether multiple hard servers could be supported; this might be possible by containerization, so that each hard server would be given its own multicast IP.

  #/bin/bash
  /usr/bin/jackd -nPART$1 -ddummy -r48000 -p512
  
