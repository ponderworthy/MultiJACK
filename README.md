# MultiJACK

The primary purpose of this project is to increase available audio DSP power available to a [JACK audio](http://www.jackaudio.org/) setup within a single multicore motherboard, using multiple JACK processes in concert, connected via IP transport of some sort.  Another purpose is to set up for much more seamless cooperation of multiple motherboards.

## History and Theory

This project began shortly after I noticed that I was using 75% of JACK DSP capability, while pushing an octocore AMD X3 with 8G RAM to only about 25% CPU, fairly evenly spread out across all cores, including a large amount of audio synthesis as well as soundfont rendering, using only about 10% RAM.  I wanted to add more capability to the box, but had capped out on JACK DSP.

There have been many failed attempts so far.  Noteworthy JACK devs have suggested publicly that it cannot work at all.  The below is working for tthree servers simultaneously, as of this writing.  

Based on both input and experience, it is suggested that NET and NETONE cannot do this at all, because the whole design of JACK is to be as absolutely timing-synchronized as possible: single JACK, NET, and NETONE all behave similarly, not leaving any wiggle-room (CPU and I/O cycle flexibility) left over.  NET and NETONE work nicely with multiple motherboards connected by Ethernet, again building effectively one big lockstepping JACK tarantula with legs in each motherboard.  This is a great way to combine the horsepower of multiple motherboards; and I have thought about buying four or ten Raspberry Pis; but my application needs major CPU for synthesis and rendering and requires physical transportability.  So.  

One source of the wiggle room we need, may exist in the form of resampling.  Traditional JACK lore has it that resampling is terrible, awful, quality-destroying tarantula-poison.  However, Fons Adriaensen  and his Zita project:

http://kokkinizita.linuxaudio.org/linuxaudio/index.html

have long been bucking the traditional, and have been increasing our options for maximum quality for quite a while now.  And 
zita-njbridge appears to be a method quite close at hand, to give us (a) IP transport between JACK servers along with (b) synchronization-independence, with Zita-class resampling for our wiggle room.

I wish to thank Fons explicitly for a MultiJACK design contribution: for each soft server, there is a *pair* of zita-n2j and zita-j2n:  

* One n2j for each soft server, runs on the hard server as the receiver; 
* One j2n runs on each soft server, sending to its respective n2j on the hard server.
* Each pair, runs on its own IP port.

## Vocabulary

In MultiJACK, we have JACK servers connected to real audio hardware, called "hard servers", and JACK servers not connected to real audio hardware, called "soft servers".  

## Current Working Setup

The current test runs, have three soft servers, and one hard server.

### Standard [Cadence/KXStudio](http://kxstudio.linuxaudio.org/Applications:Cadence) JACK setup, for the hard server

A whole lot of problems with JACK setup, control, and cooperation with other components, have been [solved by the JACK and KXStudio people](https://github.com/jackaudio/jackaudio.github.com/wiki/WalkThrough_User_PulseOnJack), in the newer versions of Cadence and associated tools.  We therefore start here.  It is theoretically possible to use more than one hard server per motherboard, but to do so would probably require containerization, to give each hard server its own effectively independent NIC and IP.  Unless of course we start looking at multiple motherboards, in which case all we need to do is vary IPs.

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

SOFT-IP needs to be started after all soft JACK servers are running.  It starts zita-j2n, which connects to each soft server, and then looks for zita-n2j over multicast and connects to it, completing the connections between each soft server and the hard server.

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

* To run multiple JACK servers on one motherboard, each JACK server has to have its own name.  This has been a standard ability of JACK for ages, but rarely used.  When a name is not specified, there is a default which is universal.
* Cadence and its corrolary tools is designed to use the default, so in the above setup, we use the default as the single hard server, to keep things as simple as possible for now :-)  
* The soft servers are all named SOFT1, SOFT2, et cetera.
* There are two ways to use a named JACK server.  One is via command-line option; for zita-j2n above, the option is "--jserv", and many (but definitely not all, and probably not most) JACK client applications will let you specify JACK server name by a command line option.  
* The other is an environment variable: the JACK client library looks for $JACK_DEFAULT_SERVER, to give the name of the JACK server, and if the variable is present will use it unless told otherwise by code inside the client application.

So if we want to run, say, Yoshimi, and attach it to soft server #1, we do this:

    JACK_DEFAULT_SERVER=SOFT1 bash -c 'yoshimi'
    
which runs yoshimi, setting that variable for its run alone.  

That gives us a signal source on soft server #1.  And all of our binaries are now running, and the IP links are functional. But we also need to connect the zita-n2j instances on the hard server, to the real audio hardware outputs, and also Yoshimi, to zita-j2n, on soft server SOFT1.  To do this, we run Patchage, twice:

    patchage
    JACK_DEFAULT_SERVER=SOFT1 bash -c 'patchage'

and make the connections.  And then we try Yoshimi using its on-screenkeyboard.  Voila!  We can then add other JACK clients to any of the JACK servers we desire, and watch things behave in wonderful ways.
    
### Addenda

1. At this moment, I have Cadence set up to run JACK with period 512 and number of periods 3, for 10.7ms latency on this relatively slow testing box and its inexpensive USB audio, but you may notice that the period in SOFT is 128.  According to the zita-njbridge docs, the total latency is thus not much more than the hard server's, and CPU load of all of the above running on this aged Intel E7300 is less than 10% with lots of other things running including Firefox.  When testing moves to a performance box, the hard server will be set up with a significantly smaller period, and the soft server(s) will be set with periods even smaller yet.  This is expected to produce either a negligible increase in latency over single-jack, or quite possibly a net reduction, if the hard server can be kept extremely simple, and fed by multiple asynchronous soft servers.

2. In a [linux-audio-user](https://lists.linuxaudio.org/listinfo/linux-audio-user) post on 2016-04-06, Stéphane Letz suggested:

> jackd2 has a « loopback » driver implemented since day 1 (OK maybe day 2….)
>
> - the code is in common/JackLoopbackDriver.cpp, h
>
> - it can be activated using  the -L parameter like : jackd -L 4 -d also xxxxxx  to add 4 loopback ports.

At the time, I did not understand it, but I do now.  This does not give multiple-motherboard capability, but it may well permit making much more use of a single motherboard, and there is a whole lot less overhead involved.  I have tried this a few different ways, and have not been able to get it to work at all; if you have a way, please do get with me!

3.  And don't forget to have fun!

Jonathan E. Brickman
jeb@ponderworthy.com
