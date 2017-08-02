---
title: "Hacking Voting Machines at DEF CON 25"
layout: post
author: TJ Horner
summary: "Let's rig some elections."
permalink: /post/hacking-voting-machines-def-con-25
cover: assets/voting-village/cover.jpg
---

**UPDATE JULY 31 2017:** Hard-coded username/password vuln added.

**UPDATE AUGUST 2 2017:** Photos have been added to the bottom of the page. And a note: I just wanted to clarify to anyone reading this: I am **not commenting** on the likeliness that any recent election was "hacked" or "rigged" by anyone. The truth is we simply don't know. Anything could have happened during the election, so without hard evidence that the exploits listed below were used, there is **absolutely no way of knowing for sure.** This post is simply to provide the research into what goes on in the election systems we trust so much to bring us accurate results, and to give people _facts_ so they can form their _own_ opinion. If you're wondering about my opinion, I think we should stick to paper ballots. But that's just me. Go ahead, read this post and form your own.

![](/assets/voting-village/cover.jpg)

# Introduction

DEF CON 25 was my first DEF CON, and it was totally amazing. There was a first-time village called the "Voting Village". It was a village where you can join in the fun of breaking into various voting registration and ballot machines. My friend Sean and I saw this village in the con book and first decided we would walk around to see everyone's progress for around 30 minutes before attending a talk. Little did we know, it would turn into an ~8 hour hacking session on voting machines.

# The Machine

We decided to work on the [ExpressPoll 5000](http://www.essvote.com/products/5/11/electronic-poll-books/expresspoll-5000%C2%AE/) voter registration machine since nobody had made any real progress on it.

![An image of an ExpressPoll 5000 device.](/assets/voting-village/expoll-5000.jpg)

It uses an ancient PCMCIA/CF slot to transfer data and hold various database files on it. Luckily, the village provided a ton of PCMCIA cards to play around with and a PCMCIA to USB converter so we could use it as a storage device on our laptops.

# General Discoveries

Here is a list of discoveries that we found useful in exploiting the device:

- It runs Windows CE 5.0
- ExpressPoll software is a giant .NET app with the assembly name `ExPoll`
- ExpressPoll software uses WinForms for UI
- Bootloader is proprietary
- Processor's architecture is ARM
- Database file is stored on memory card (more details later)

# The Exploits

Several exploits were tested against this, such as the usual suspects: buffer overflow, Windows key shortcuts, network exploits, etc. Here is what worked and what didn't.

## Arbitrary Firmware Injection

When the memory card is inserted into the machine and it boots up, the bootloader first checks for a file called `NK.BIN`. If this file is found, it will attempt to load it into RAM and boot into it with _absolutely no signature verification_. The firmware is then flashed and is permanent upon next boot.

The `NK.BIN` file is generally a Windows CE 5.0 update for the machine bundled with the ExpressPoll software, but an attacker is able to upload and run any custom NT kernel, and possibly a Linux image (untested thus far).

We tested this on our ExpressPoll 5000, and sure enough, it loaded our Windows CE without complaining:

![An image of an ExpressPoll 5000 with the text "Download Image from CF Card to RAM, please wait..."](/assets/voting-village/firmware-upgrade.jpg)

## Arbitrary Bootloader Injection

Very similar to the firmware injection exploit, the bootloader also checks for an update to itself while booting, with the file name `EBOOT.BIN`. It is also loaded without signature verification, but we have not been able to get a bootloader to successfully work on a machine. However, the bootloader injection itself is confirmed to work since the bootloader we put onto the `EBOOT.BIN` file is indeed flashed and is present upon reboot without the memory card. This also means we basically bricked the thing (we had to get another, heh).

## .NET `.resources` File Override

When the ExpressPoll software is "launched" from the "Launch Express Poll" button, it reads the memory card for a file named `ExPoll.resources`. This is a resources file defining any resources that the app will use, such as translation strings, images, buttons, layouts, etc. It would generally be used for overriding and customizing the registration process (for example, adding your own logo instead of having the Diebold logo.)

The weak point in this system is that buttons (and potentially other UI elements) can be overriden to take different actions, such as run an executable stored on the memory card (which is mounted in something like `/Storage Card`) or run a command.

If an `ExPoll.resources` file is found, it will first load it into memory, use it for that session, and also copy it to the device's internal memory so it can be used again.

This vulnerability was confirmed by creating a random .NET `.resources` file, naming it `ExPoll.resources` and copying to the memory card. We popped it into the machine, pressed the "Launch Express Poll" button, and (as expected), it errored. Woohoo! It loaded the file. Sadly, since it also copied to memory, it is essentially bricked if you give it a bad file (you can't get to the "Launch Express Poll" button). Sadly, this was the case with our device so I don't have any images of it in this state.

# The Vulnerabilities

Along with the exploits described above, we have found many security vulnerabilities that can enable several types of attacks.

## Hard-coded Username/Password

When you launch the ExpressPoll software, you are required to enter a consolidation number, plus a username/password.

There is one hard-coded into the software that always works:

- Username: `1`
- Password: `1111`

Brilliant.

## Unencrypted SQLite3 Database

When starting the ExpressPoll software, it looks for a file called `PollData.db3` on the memory card. This is an standard unencrypted SQLite3 database, including _literally all the information_. Polls, parties, voters, all of it. (Link coming soon with sample database.) Using this information, an attacker could launch several types of attacks:

- Data exfiltration
  - For example, the attacker could easily come with a card that has an empty database, and silently swap out the card when they register to vote, thus obtaining the entire voter database (which includes: name, address, last 4 of SSN, signature, and many others) for the county.
- Falsifying voter information
  - For example, the attacker could forge their own database with fake voter information (e.g. change their parties/signatures, etc.) and place it into the machine when they register to vote.
- Other attacks. Your imagination is the limit when you have access to the entire database.

![](/assets/voting-village/voter-data.jpg)

## 2 Open USB Ports

These pose less of a threat, but it is still threatening. An attacker could plug any USB device they wanted (for example, a USB Rubber Ducky or LAN Turtle) to launch a creative attack. I have tested this with my Bash Bunny by spamming the letter "a" infinitely into one of the text boxes in an attempt to trigger a buffer overflow and potentially crash the .NET app (it didn't work).

![](/assets/voting-village/usbports.jpg)

## Default WinCE Web Server

The default Windows CE web server is open on port 80, which poses a possible security risk (among the others lol). Nothing has been confirmed using this server, so further research is required.

# Other Attempted Attacks

We attempted some other attacks, but they didn't work.

## Memory Overflow

Using a Bash Bunny, I wrote a payload to spam the letter "a" infinitely into a text box, and plugged it into the machine for around an hour. Time stopped and everything got super slow, but nothing crashed.

## Memory Overflow (again)

Using the `ConsolidationMaps` table on the aforementioned SQLite3 database, I changed a map image to something outrageous in size (>1GB) and loaded it up. It didn't exit the .NET app like I expected. It errored (with a `malloc()` out-of-memory error) but did not crash the .NET app. It simply restarted.

## Installing Grub (almost worked)

We attempted to create a Grub image for the ARM architecture, but we couldn't finish building it before the village closed. If you have access to an ARM Grub image and an ExpressPoll 5000, try to install it using the `EBOOT.BIN` trick described above. Ping me if it works :)

# Conclusion

DEF CON 25 was super fun, and this was definitely my favorite part. If you have any questions about this, er, research, let me know on Twitter: [@tjhorner](https://twitter.com/tjhorner).

# Credits

I couldn't have done this without the help of others that joined our team later on, they were super helpful:

- Ian Smith
- Morgan Jones
- Thomas Fulmer
- Sean Roach ([@TheEdgyDev](https://twitter.com/TheEdgyDev))
- Some others (lmk if you belong here, I'm not good with names)

# Various Images

Here are all of the images I took while hacking these machines: [Google Photos Album](https://goo.gl/photos/baSp4QDVYGABfMxV8)

You are allowed to use these photos freely, so long as you credit me with my Twitter username (@tjhorner) and a link back to this blog post.