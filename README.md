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

At the moment an initial foundation exists and is working.  This is not in production condition, I will be working on some Python scripting for efficient automatic startup and control.  This will not play ball with PulseAudio or Cadence for the known future, major work on them would be required.  I do use qjackctl to discover and prove jackd command lines and latency settings, but only for discovery and proving, because it too is explicitly not designed for this purpose, and if my experiences are representative, attempts to use it are likely to result in confusion and frustration :-)

### Script HARD

HARD starts the single JACK server connected to real audio hardware.  I have a USB stereo audio device in use, whose ALSA name (see /proc/asound/cards) is "Device", thus:

    #/bin/bash
    /usr/bin/jackd -nHARD -dalsa -r48000 -p512 -n3 -Xraw -D -Chw:Device -Phw:Device

In current testing, this is run within its own xterm.

It is unclear whether multiple hard servers could be supported; this might be possible by containerization, so that each hard server would be given its own multicast IP.  Food for thought anyhow.

### Script HARD-IP

HARD-IP needs to be run after HARD starts, after jackd -nHARD is running and confirmed good.  It starts zita-n2j, which connects to the hard server, and waits to receive audio data from soft servers, via multicast on port 54321:

    #/bin/bash
    zita-n2j --jserv HARD 127.0.0.1 54321

In current testing, this is run within its own xterm.

### Script SOFT

SOFT starts JACK servers not connected to audio hardware.  They run the dummy driver.  This script is invoked with a single command-line option, which is the soft server number.  zita-njbridge supports a theoretical maximum of 64 total servers, which means 1 hard and 63 soft.  

    #/bin/bash
    /usr/bin/jackd -nSOFT$1 -ddummy -r48000 -p512

For the current test example, after HARD and HARD-IP are running in their separate xterms, I start two more xterms, one running this command:

    bash SOFT 1

the other

    bash SOFT 2

### Script SOFT-IP

SOFT-IP needs to be started after all soft JACK servers are running.  It starts zita-j2n, which connects to each soft server, and then looks for zita-n2j over multicast and connects to it, completing the connections between each soft server and the hard server.

    #/bin/bash
    zita-j2n --jserv SOFT$1 --float 127.0.0.1 54321
    
For the current test example, after all of the above are running in their separate xterms, I start two more xterms, one running this command:

    bash SOFT-IP 1

the other

    bash SOFT-IP 2
  
