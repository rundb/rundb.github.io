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

Another issue that I had to suddenly address in the field was the filesystem. I wanted to use a media-agnostic file system from the internet, but eventually we haven't managed to
bring it up on Nordic. It was kind of working, but with so many issues that I had to make a decision to divert to self-developed simple storage. Spoiler: file system issues followed
me all the way until the end of the project. 

Hackathon took about 3 days. Eventually we have been able to perform a simple demo that basically has been just a straight line operation in the pipeline. Instead of file system we
used a raw write to the flash memory, file has been erased right after the transmission, transmission has been sometimes failing, and we were satisfied with just showing the transcribed
text on the screen of the phone. No restarts were possible, no timeouts were implemented, and many other aspects have been ignored.

# Pareto Principle