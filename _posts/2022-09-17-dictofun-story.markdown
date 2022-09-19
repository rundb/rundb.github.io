---
layout: post
title:  "Dictofun - how it went"
date:   2022-09-17 11:46:24 +0200
permalink: /dictofun-story
---
Aboout a year ago from now, in August 2021, I was asked to come up with some ideas for an internal hackathon 
in the company I worked at. Usually for this hackathon offer various robots, games, for some people it's a chance 
to try another language (most often Rust). I almost always have a bunch of ideas of various electronic devices in my pocket. 
This time hasn't been an exception. 

# Concept

At the time of brainstorming this project has been called `Big Red Button` (let's name it `BRB` for now). Unlike the big brother, which is known to 
be able to cause a global apocalypsis, this device should've had much more modest abilities. Idea was - come up with a small device, most 
of which is a big red button that blinks when you press it. As soon as you press it (later we even have defined _soon_ - no later than 400
ms), it starts recording your voice and store it on SPI flash located next to the microcontroller. After releasing the button, the BRB should connect
to the phone and in the background transfer the file to the phone (let's keep term `phone` undefined for now). Phone then in the background should 
transcribe the received voice file to a text. This text then should be sent to your favorite note keeper (I was planning to use Telegram bot for that,
but this milestone eventually has not been reached).

# Challenges at the time

The pipeline is no rocket science, but there have been few challenges. First, HW projects are known to cause a huge amount of issues, which are hard
to resolve under time pressure and being outside of a lab with proper instruments. 
Second, it's a pipeline of data, and therefore in order to test it properly, the whole pipeline should be functional. 
In this case, it consisted of following steps for data: sensor to MCU, MCU to flash memory, flash memory back to MCU, MCU to phone,
phone to recognition server. Each stage could be implemented separately, but the final result is observable only when the whole pipeline operates. 

# First steps

First 2 revisions of the hardware has literally been looking like a big button. I found a big button somewhere on Digikey. For MCU I have chosen NRF52832,
as it had BLE on board, has been well-known, I did several PCBs with it already and had a will to experiment with it. Also, NRF52840 may have been a 
slightly better choice at the time, but since first revisions have been assembled by hand, NRF52840 couldn't be used in this case. First microphone 
has been analog, so we also utilized RC chain in the supply and in the data line of the microphone. I wanted to use I2C, but at this point I couldn't make any of available 
mics work with NRF52. AFAIU the problem was that Nordic's I2S supports only 24-bit frames, whereas the mics wanted 32 bits (I checked that with RPi, as it's know to
work with I2S microphones). Battery has been kicked out of scope for that time, we already had enough complexity. For phone we have chosen Google Pixel, 
as I had several of those and we didn't need Macbook for the development. 

By the time of the hackathon I have debugged the first revision, ordered the second revision and brought it up. It still used an analog microphone, but I stocked
some evaluation boards that could use I2S or PDM as well - just in case. I also have prepared some basic software elements: made sure that SPI flash is operational,
and checked that microphone produces signals that look like sound on the osci. We also stocked several NRF52-DK boards, as parts of the pipeline could be debugged 
without having the whole device. Everything has been ready for the hackathon.

# Hackathon 

We had a team of 4 people. One has been responsible for microphone and flash memory operation, one - for implementation of file transfer protocol over BLE to phone,
one person developed Android app and myself directing the process and bringing help where it was needed the most. Already at the hackathon location we found several
hardware issues, so I had to do soldering work in the field (btw, it's almost literally - hackathon took place in a village in Tiroler mountains). First issue - we 
suddenly have decided to move from analog microphone to PDM-based (to avoid all noise-related issues). Second issue - by doing so, we forgot that we had an RC chain on the data 
lane, and at one point we had saturation issues for the signal. 

Another issue that I had to suddenly address in the field was the filesystem. I wanted to use a MCU-agnostic file system from the internet, but eventually we haven't managed to
bring it up on Nordic. It was kind of working, but with so many issues that I had to make a decision to divert to self-developed simple storage. Spoiler: file system issues followed
me all the way until the end of the project. 

Hackathon took about 3 days. Eventually we have been able to perform a simple demo that basically has been just a straight line operation in the pipeline. Instead of file system we
used a raw write to the flash memory, file has been erased right after the transmission, transmission has been sometimes failing, and we were satisfied with just showing the transcribed
text on the screen of the phone. No restarts were possible, no timeouts were implemented, and many other aspects have been ignored. The only focus of the hackathon has been to bring 
up a PoC, which has basically succeeded.

# Pareto Principle

Soon after the hackathon was the time when I experienced the full scale of Pareto's law. PoC has been done, and I continued development of both hardware and firmware. I was considering
to reuse the SW developed on the hackathon, but eventually it has been easier to rewrite it completely. I made couple mistakes at this stage: the architecture hasn't been optimized for
quick experiments. In terms of the pipeline the main problem has been the file system, the rest of the pipeline has been mostly reused. In terms of hardware the main changes have been 
related to the battery charger subsystem and to testability of the hardware. There are still couple things missing from making it a close-to-perfect hardware: I would add an RTC chip
into it and add a digital latch circuit for powering the device off. 3 hardware revisions and software development took about a year of slow after-work and weekend work, with big 
delays related to a) development of iOS application and b) russian invasion into Ukraine. So it took roughly 8 working days to design, produce, bring up and develop the demo and about
20-30 working days to rework it to be able to store several records, transfer them to iPhone and get rid of all sporadic errors. Spoiler: sporadic errors are still in place. 

# Hardware design

The system is built around Nordic NRF52832. PDM microphone is the input device, so is the button. Data is stored to SPI NOR flash chip with 16 MBytes of memory. Also on board there is a RGB-LED,
LDO, latching circuit and a battery charger circuit. All buses have test points that are put on the bottom of the board, together with several power-related testpoints. This allowed to introduce
a bed of nails used for development purposes. First revisions have been using direct soldering on the PCB of wires related to console and JTAG, but I have decided to experiment and designed a 
bed of nails.

# Software design

Probably my biggest problem has been usage of Segger Embedded Studio as a build system. It may have been a good idea to utilize CMake/Bazel at the start in order to make the build process 
easier for other developers. 
Software for Dictofun is written using C++. No operating system is used, all operations are happening in the while-loop in the main function. There are three big SW modules that I called "tasks".
Main module is `task_state`, which is responsible for operation of the main state machine of the software. `task_audio` contains functions related to audio processing (technically there is no
processing, only repacking of audio frames to filesystem). `task_led` is responsible for blinking RGB LED, depending on the current state. Initial idea has been to split these tasks in order
to be able later to integrate FreeRTOS.

The most controversial aspect of the whole project is the filesystem. I have been attempting to use `littlefs` and few other filesystems designed for SPI flash memories. Eventually all of these 
systems have been triggering erase operation during the write of the record. I later came up with an idea on how to work it around without tweaking the FS, but at the time when I needed this FS
I have made a decision to design a "streaming" filesystem that would not be calling erase functions by design. I know the rule "don't write your own libs unless you really know what you do". 
My main idea has been that most operating systems out there are designed to be universal and not targeting at such streaming use-cases (even though the data stream has been not really huge,
just 24 kbytes/sec, SPI flash has been running at 1 MHz). I think this FS is the candidate #2 or #3 for refactoring (after introduction of build system and a console). Anyway, it has a weird API,
and to some extent it does work. 

File transfer protocol over BLE is also a big source of pain. It was inherited from a demo project of Nordic, where they use this protocol to stream photos from an evalboard to the phone. 
To be honest, it's just now is the time when I thought of it as of "protocol over BLE". It's just 3-4 characteristics, one for sending commands and two-three others to transfer the data of 
different types (notify function of BLE characteristic has been utilized). My biggest mistake here is probably that I only have been running this protocol only within the system "dictofun-phone". 
Right way to do it is probably to have a protocol that is easier to split to small pieces and use a media that is easy to analyze on the opposite side. It should be possible to debug the protocol
all the way to small radio packets sent over BLE, otherwise it's not possible to build an efficient protocol for 2-3 platforms. I have tried to use a BLE sniffer, but haven't manage to use it in
a convenient enough way. 

How would I've done it now? This functionality should be initially developed and tested on a separate bench with evalboard on one side and something like a Raspberry PI on the other side, with 
a full analysis of inbound and outbound BLE traffic, with timestamps and all possible flags that are used within BLE. Same setup should be used for development of this protocol between RPi and phone,
so other role is launched and tested on the RPi. This way one can assure that no surprises show up in an invisible way (fortunately there is enough help in the internet to debug BLE).

Another big thing where I have missed the RTOS is the communication between the modules. I think queues from FreeRTOS could've been very helpful. I had to pass the same file between several components
of the system to pass the signals. Basically all communication has consisted of direct calls of particular methods of particular objects. Probably queue-based system could've been more robust in this
case.


# Current problems

Main issue - battery lifetime. Power consumption is not optimized. The easiest way to reduce power consumption is by integrating an audio codec into the pipeline. Currently data is
recorded at rate of 24 kbytes/sec, being stored in raw WAV format into flash memory. Voice codec should be capable of reducing the amount of written data significantly, which should
result in shorter file transaction times and thus less active radio time and less power consumption.