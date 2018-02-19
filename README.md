# MultiJACK

This project contains a fully operational demo of a framework which increases available audio DSP power available to a [JACK audio](http://www.jackaudio.org/) setup within a single multicore motherboard, using multiple JACK processes in concert, connected via IP transport.  It also forms a beginning for very seamless cooperation of multiple motherboards.  It has been implemented for production use in the [Box of No Return](https://ponderworthy.github.io/the-box-of-no-return/).

## Vocabulary

In MultiJACK, we have JACK servers connected to real audio hardware, called "hard servers", and JACK servers not connected to real audio hardware, called "soft servers".  

## Practical Advantages

First and last and most of all, we can get a lot more flexibility, a lot more simultaneous complex patch setup, on one motherboard, provided we have enough CPU; the current working example uses a 4 GHz octocore AMD with 8 gigabytes RAM.  

Second of all we get a much more stable system, which stays running even through momentary overloads.  In a single-JACK system, a major overload will provoke static and serious unpleasantry through the sound system, and may crash it all. When we do MultiJACK, if the major processing is done with clients of the 'soft' servers (see below) and not the 'hard' server, overload means about a quarter-second of silence (no static through the PA) from the one soft server only, cutout of just only those directly connected processes; and it all comes back quickly, no train-wrecks, because zita-njbridge auto-reconnects.

## History and Theory

This project began shortly after I noticed that I was using 75% of JACK DSP capability, while pushing an octocore AMD X3 with 8G RAM to only about 25% CPU, fairly evenly spread out across all cores, including a large amount of audio synthesis as well as soundfont rendering, using only about 10% RAM.  I wanted to add more capability to the box, but had capped out on JACK DSP.

There have been many failed attempts.  Noteworthy JACK devs have suggested publicly that it cannot work at all, or cannot work well.  The literal code below is working for three servers simultaneously, as of this writing.  

Based on both input and experience, it is suggested that NET and NETONE cannot do this, though to this writer it is not clear why.  NET and NETONE work nicely with multiple motherboards connected by Ethernet, again building effectively one big lockstepping JACK tarantula with legs in each motherboard.  This is a great way to combine the horsepower of multiple motherboards; and I have thought about buying four or ten Raspberry Pis; but my application needs major CPU for synthesis and rendering and requires easy physical transportability.  So.  

One thought has been that we need a bit more wiggle-room than NET or NETONE can give us.  JACK does tend towards the extreme in lockstepping, in time-synchronization -- and this is absolutely essential for a great many of its applications! -- but for our synth we don't need to worry up to about 5 milliseconds between keyhit and tone roar.  And there is one known source of wiggle room: resampling.  Traditional JACK lore has it that resampling is terrible, awful, quality-destroying tarantula-poison.  However, Fons Adriaensen  and his Zita project:

http://kokkinizita.linuxaudio.org/linuxaudio/index.html

have long been bucking the traditional, and have been increasing our options for maximum quality for quite a while now, in part with excellent-quality resampling.  'zita-njbridge' appears to be a method quite close at hand, to give us (a) IP transport between JACK servers along with (b) synchronization-independence, with Zita-class resampling for our wiggle room.

I wish to thank Fons explicitly for a MultiJACK design contribution: for each soft server, there is a *pair* of zita-n2j and zita-j2n:  

* One n2j for each soft server, runs on the hard server as the receiver; 
* One j2n runs on each soft server, sending to its respective n2j on the hard server.
* Each pair, runs on its own IP port.


## Confirmed Test Setup

The current test runs, have three soft servers, and one hard server.

### Standard [Cadence/KXStudio](http://kxstudio.linuxaudio.org/Applications:Cadence) JACK setup, for the hard server

A whole lot of problems with JACK setup, control, and cooperation with other components, have been [solved by the JACK and KXStudio people](https://github.com/jackaudio/jackaudio.github.com/wiki/WalkThrough_User_PulseOnJack), in the newer versions of Cadence and associated tools.  We therefore start here.  

### Script HARD-IP

HARD-IP needs to be run as the first MultiJACK component, after the standard hard server runs.  

    #/bin/bash
    echo ""
    echo "Starting zita-n2j #$1 ..."
    echo ""
    zita-n2j --jname zita-n2j-mj$1 127.0.0.$1 5555$1

Three of these are run, one at a time, each in its own xterm, a la:

    bash HARD-IP 1
    bash HARD-IP 2
    bash HARD-IP 3
    
These are the IP receivers, for each transmitter running on every soft server.  

You may notice that each has its own localhost IP, e.g., 127.0.0.1, 127.0.0.2, etc.  This was found to be necessary to avoid errors, it is not clear why.  

### Script SOFT

SOFT starts JACK servers not connected to audio hardware.  They run the dummy driver.  This script is invoked with a single command-line option, which is the soft server number.  zita-njbridge supports a theoretical maximum of 64 total JACK servers in concert.

    #/bin/bash
    echo ""
    echo "Starting soft JACK server #"$1" ..."
    echo ""
    /usr/bin/jackd -nSOFT$1 -ddummy -r48000 -p128

Three of these are run, one at a time, each in its own xterm:

    bash SOFT 1
    bash SOFT 2
    bash SOFT 3

### Script SOFT-IP

SOFT-IP needs to be started after all soft JACK servers are running.  It starts zita-j2n, which connects to a soft server, and then looks for zita-n2j on its appropriately numbered port and connects to it, completing the connections between each soft server and the hard server.

    #/bin/bash
    echo ""
    echo "Starting zita-j2n #"$1" ..."
    echo ""
    zita-j2n --jname zita-j2n-mj$1 --jserv SOFT$1 127.0.0.$1 5555$1
    
Three of these are run, one at a time, each in its own xterm:

    bash SOFT-IP 1
    bash SOFT-IP 2
    bash SOFT-IP 3

### Real Testing and Use

So after Cadence has JACK running well, we have the following, each started in its own xterm, in order:

    bash HARD-IP 1
    bash HARD-IP 2
    bash HARD-IP 2
    bash SOFT 1
    bash SOFT 2
    bash SOFT 3
    bash SOFT-IP 1
    bash SOFT-IP 2
    bash SOFT-IP 3

And we have all of these processes running, and the xterms with HARD-IP report the IP links connected.  How do we do actual use testing?  Items of note:

* To run multiple JACK servers on one motherboard, each JACK server has to have its own name.  This has been a standard ability of JACK for ages, but often unused.  When a name is not specified, there is a default which is universal.
* Cadence and its corrolary tools is designed to use the default, so in the above setup, we use the default as the single hard server, to keep things as simple as possible for now :-)  
* The soft servers are all named SOFT1, SOFT2, et cetera.
* There are two ways to use a named JACK server.  One is via command-line option; for zita-j2n above, the option is "--jserv", and many (but definitely not all, and probably not most) JACK client applications will let you specify JACK server name by a command line option.  
* The other is an environment variable: the JACK client library looks for $JACK_DEFAULT_SERVER, to give the name of the JACK server, and if the variable is present will use it unless told otherwise by code inside the client application.

So if we want to run, say, [Yoshimi](http://yoshimi.sourceforge.net/), and attach it to soft server #1, we do this:

    JACK_DEFAULT_SERVER=SOFT1 bash -c 'yoshimi'
    
which runs yoshimi, setting that variable for its run alone.  

That gives us a signal source on soft server #1.  And all of our binaries are now running, and the IP links are functional. But we also need to connect the zita-n2j instances on the hard server, to the real audio hardware outputs, and also Yoshimi, to zita-j2n, on soft server SOFT1.  To do this, we run [Patchage](http://drobilla.net/software/patchage), twice:

    patchage
    JACK_DEFAULT_SERVER=SOFT1 bash -c 'patchage'

and make the connections.  And then we try Yoshimi using its on-screenkeyboard.  Voila!  We can then add other JACK clients to any of the JACK servers we desire using additional Patchage runs, and watch things behave in wonderful ways.
    
### Addenda

1. When last testing the literal above, I had Cadence set up to run JACK with period 512 and number of periods 3, for 10.7ms latency on this relatively slow testing box and its inexpensive USB audio, but you may notice that the period in SOFT is 128.  According to the zita-njbridge docs, the total latency is thus not much more than the hard server's, and CPU load of all of the above running on this aged Intel E7300 is less than 10% with lots of other things running including Firefox.  When rebuilding the [Box of No Return](https://ponderworthy.github.io/the-box-of-no-return/) I found that my new Mackie Onyx Artist was running just fine with JACK latency of just 1.7ms, so this has not been needed, the general word is that human beings cannot detect latencies of less than 5ms.

1.  One of the challenges of implementing MultiJACK is automating the startup.  Clearly we do not want to manually start a pile of xterms whenever we play!  The Python library jpctrl, created for the Box of No Return, handles automation of JACK startup, handles this very well.

1.  All glory to God!

1.  And don't forget to have fun!

Jonathan E. Brickman
jeb@ponderworthy.com
