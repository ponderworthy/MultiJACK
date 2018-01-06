# MultiJACK

The purpose of this project is to increase available audio DSP power available to JACK within a single multicore motherboard, using multiple JACK processes in concert, connected via IP transport of some sort.

## History and Theory

This project began shortly after I noticed that I was using 75% of JACK DSP capability, while pushing an octocore AMD X3 with 8G RAM to only about 25% CPU, fairly evenly spread out across all cores, including a large amount of audio synthesis as well as soundfont rendering.

There have been many failed attempts so far.  Noteworthy JACK devs have suggested publicly that it cannot work at all.

Based on both input and experience, it is suggested that NET and NETONE cannot do this at all, because the whole design of JACK is to be as absolutely timing-synchronized as possible: single JACK, NET, and NETONE all behave similarly, not leaving any wiggle-room (CPU and I/O cycle flexibility) left over.  NET and NETONE work nicely with multiple motherboards connected by Ethernet, again building effectively one big lockstepping JACK tarantula with legs in each motherboard, and this is a great way to combine the horsepower of multiple motherboards; and I have thought about buying four or ten Raspberry Pis; but my application needs major CPU for synthesis and rendering and requires portability.  So.  It does appear, that the wiggle room we need, may have to exist in the form of resampling.

Traditional JACK lore has it that resampling is terrible, awful, quality-destroying tarantula-poison.  However, the Zita project:

http://kokkinizita.linuxaudio.org/linuxaudio/index.html

has long 

zita-njbridge appears to be the closest method at hand.

From the page:
--
Command line Jack clients to transmit full quality multichannel audio over a local IP network, 
with adaptive resampling by the receiver(s). Zita-njbridge can be used for a one-to-one 
connection (using UDP) or in a one-to-many system (using multicast). Sender and receiver(s) 
can each have their own sample rate and period size, and no word clock sync between them is 
assumed. Up 64 channels can be transmitted, receivers can select any combination of these. 
On a lightly loaded or dedicated network zita-njbridge can provide low latency (same as 
for an analog connection). Additional buffering can be specified in case there is significant 
network delay jitter. IPv6 is fully supported. 
