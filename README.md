# Lytro Unlock - Making a bad camera slightly better
![alt text](https://github.com/ea/lytro_unlock/blob/main/lytro_images/00_main.jpg?raw=true)

![_DSC6236](https://user-images.githubusercontent.com/65802/213890019-91dc7961-75c2-4b6c-b12e-40b2f86a5454.jpg)

## TL;DR; summary 
I’ve recently spent some time playing with and reverse engineering this curious piece of tech that was a first consumer oriented,though odd looking, lightfield camera called Lytro. Killer feature of this new technology was the ability to refocus the image after it was taken! 
The bad side was that the software was pretty bad, the camera was trying to solve a problem that didn’t exist and the whole endeavor mostly failed. 

This project documents and implements a python library that unlocks and makes available a few interesting features that weren’t ever made available through official software. Among those are:
- full remote camera parameters control
- Live View streaming
- Debug console
- Custom code execution

Although it failed as a commercial product, lytro camera has some pretty nifty tech (only one of which is very high optical zoom). It is my hope that somebody out there has a cool idea that would benefit from having full software control over the camera. 
If you are interested in the details, read on. Otherwise, feel free to play with supporting software under [https://github.com/ea/lytroctrl].

## The Why - Start

Anybody remember these weird rectangles from the photo above? They were all over tech news websites almost a decade ago. They were supposed to usher in the new age of photography. 
Some of you might have them tucked away in some drawer as a reminder of wasted money.
The product promised so much, yet what it delivered in reality was oh so disappointing. 
Why did I get interested in this? Few months ago , an interesting twitter thread caught my eye. It was written by (https://web.archive.org/web/20221014234930/http://twitter.com/warren_craddock) Warren Craddock and was talking about tech projects can have so much momentum that people can ignore the obvious flaws and showstoppers for a long time before they are abandoned. One of the projects he talked about was a Lytro camera. Flaws being that , due to laws of physics and optics, most of the benefits of the technology (refocus, paralax) would be of no practical use to general public. 
Long story short, after developing a second generation of the LightField camera called Illum (this time in a more familiar DSLR-like package) company soon went bust. 
Naturally, the first question that popped into my mind was: “I wonder how cheap those are on eBay right now?”
Sure enough, if you are patient, you can easily pick them up for anywhere from $20 to $40 dollars. A great discount from the original retail price. 
I decided to procure one, if for no other reason than simply to have it as an interesting relic. But… The curiosity got the better of me and I soon found myself digging into the firmware. 

## Goals

Lets review the current state of affairs with regards to Lytro and set out some goals. 
While the company is gone, most of the online materials, firmware and software has been kindly archived in different places online. 
The camera itself has very little in terms of controls, yet you are supposed to use it as a standalone device. It’s not permanently tethered to a phone (although there is/was an app for it at some point). One would expect that, when connected to a computer, the camera would show up as a regular mass storage device, but that is not the case. Rather, you have to use Lytro software to download and process the images. Windows and OS X versions are available, but look like they were already outdated 10 years ago and leave a lot to be desired. 
Newer firmware versions support WiFi access point mode, but the official software doesn’t make much use of it. Maybe the app did once, but it doesn’t matter. No official API or documentation was ever provided.

There were a couple of community and 3rd party projects that aimed to replicate the official software and did a very good job on documenting how things work. Highlights of those are :
- lytro meldown (insert web page) by <INSERT NAME> which was of immense help as it documents publicly known parts of the communication protocol, has an archive of previous firmware versions and still runs a firmware update service. 
- Lyli (inser webpage) - linux based implementation of USB communication protocol  

While both of these provide good starting points, they don’t offer much benefit over the official software, only a few extra features. 
Just looking at a product , without considering existing software, I thought it would be a lot more useful if it had features that a normal webcam would have. With that in mind, I set out to see if implementing the following was possible:
- control zoom level through software 
- Control focus point through software
- Take a picture on demand 
Initially, as stretch goals I was considering:
- firmware modification
- Arbitrary code execution
- Live View / video streaming

Ok, with that, lets have a specific use case we want to make possible: Aim the camera through the window, zoom to certain level and feed the live view images to computer-vision code. Whenever a squirrel  is detected in view , take a photo. Then refocus on demand later. 


## Take it apart

First things first, lets take a look at how this thing is made.

Due to its unusual shape , the device  has an unusual number of boards that are interconnected. This makes it a bit of a pain to disassemble, solder wires to test points and then reassemble but just by looking at the following photos we can identify interesting things. 
Topmost board contains the sensor with the microlens array and connects further to lens controls. Lens has tiny motors and actuators that control zoom, focus, shutter and built in neutral density filter.

Next is a power/battery control board which is also where usb connection is located. Beneath the battery is the main SoC board which is based around a MIPS MCU called Coach (Camera On A Chip) from Zoran corporation. This SoC seems to be very common with dashcam manufacturers from the same era. On the flipside, we can see Samsung’s flash and other ICs. Certain labeled test points would suggest a JTAG interface to be present. 

Finally, the last board connects to the display as well as the capacitive touch sensor that controls zoom. It is on this board that we find a peculiar looking  unpopulated connector pad. Basic testing quickly reveals UART pins like shown, but all we get is a short boot message and no input or echo…
Lets make a nice breakout of those pins for later use just in case. 

## The firmware

So we have a serial port, but it doesn’t give us anything useful (yet!).  Let’s check out the firmware to see if we should even expect anything on the serial output. 
Luckily for us, <INSERT NAME> has archived all the public firmware versions at : <insert website>

Firmware update seems to be complete and not incremental, which is a good thing. It starts with a simple JSON text that describes the actual data that follows. It appears that the firmware is just flat memory contents whithout any disernable file system. Binwalk reveals the following:

Remember, ELF files don’t imply that we are dealing with Linux. And indeed, in this case, ELF format is used for memory mapping and binary loading purposes. All the binaries seem to be mapped to their respective addresses and cross-reference each other. The whole firmware seems to be based around an unknown real-time operating system. 

Even though we don’t know the exact size of the binaries, we can assume they are contiguous in memory and just extract the maximum possible size. That gives us the following ELF files. Examining the first one reveals what appears to be a bootloader, or at least a stage of it. We can see the same strings we observed in UART output. Binaries that follow are bigger and appear to contain a lot more functionality.

Randomly stumbling through the strings reveal the following:

This definitely appears to be some sort of command palette with command names, descriptions and associated target functions. This is great news! Already I can see some that look very interesting. The question is, how do we get to them… No response over UART. 

## Interfaces

With UART unresponsive, but encouraged by what we saw in the firmware, we turn to other was of interacting with the camera. That is, USB and WiFi. 

Again, thanks to <LyTRO Meltdown> docs, we have a pretty good start when it comes to wifi communication. Similarly, from Lyli project, we can see that WiFi communication is actually based on USB protocol and mostly follows the same protocol. Only that USB is actually based around SCSI commands, while WiFi is direct over TCP/IP. 
Lytro meltdown doccuments the following set of commands:
<TODO>

Some of these look incomplete and I am willing to bet there are more so our next step is to go back to the firmware to try to find where these are being processed. 

## Command interpreter

Now we are faced with a conundrum. We have this giant piece of code, all of these binaries from the firmware, and we want to find the small chunk of code that’s parsing incoming Wifi or USB data in order to locate the code that corresponds to commands observed from official software. 

This is a common problem in reverse engineering that I like to call localization. It’s much harder to figure out what the code does without knowing the context it’s being executed in. This can often be simply solved with a debugger, but in this case we don’t have such luxury. 

Unusual or rare constants are a reverser’s friend. Those can be unique cryptographic IVs, but also something much less exotic. Consider that there are 3600 seconds in an hour. That seems like a round number in decimal, but is a somewhat unusual looking constant in hex. As such, there’s a chance it will be somewhat rare in the binary you are looking at. 

On the other hand, GetTime command from above returns camera time in a very specific format. We can guess that at some point during its execution, it would need to convert seconds to hours or vice versa. Searching for seconds-in-hour constant in the binary reveals only a handful of locations which we can manually investigate. And sure enough, one of them definitely look like time/date formatting code. Backtracking through references quickly leads us to code that looks awfully like what we’d expect a command interpreter to look like. A series of switch statements with command handlers. 

Success, now we have our bearing in the firmware and can start exploring and make guesses about functionality with more confidence. 

## Command classes, commands and payloads

All in all, by studying the command interpreter code we can deduce the following complete set of commands. All of them match with what was previously documented and observed, and there’s quite a few hidden an unknown ones.
<INSERT table of all commands>


## Secret Unlock command

You’ll notice in the above that I’ve listed an UNLOCK command. It first caught my attention because it’s clearly doing something with SHA1. That has to be something interesting. 
After cleaning up the decompiled code, the functionality becomes obvious.
The function takes some string from the memory, concatenates “ please” to it and calculates a SHA1 hash of it. Then it compares that hash to whatever was received as payload. 
After a bit more  digging, it becomes obvious that the string in memory is actually the serial number of the camera. We can easily get the serial number, so let’s calculate the expected hash, send it and see what happens:

Well, looks like that takes care of the locked serial console. In addition to unlocking the serial output, this enables command execution a lot of other functionality.


## Commands over Wifi

Following is the full list of command classes and commands. Not all of them have been fully implemented in the accompanying software yet. If you find it useful, feel free to make a pull request. 

- Full list 

Some of the commands are more complex than others and some even have subcommands of their own. 
    - Wifi Settings
        - Looks like two modes are supported, the AP mode that is the default, and “sync” mode where the camera connects to the specified access point. The second mode would probably be a lot more convenient to use, but I haven’t yet found a way to persist these particular settings across reboots. 
    - Live View
        - Parameters are enable/disable, compression ratio and FPS specification. When Live View is enabled, a new thread is started on the camera that simply sends JPEG frames over multicast UDP <insert address and port>. These can easily be captured by ffmpeg for example: <insert ffmpeg cmdline> It’s a bit unstable, and care should be taken when choosing the parameters. Check out the acompanying LiveView.py script
    - Manual Controls
        - Live View above is actually implemented as a subcommand to manual control command. Subcommands are as follows : <insert list of subcommands>
    - Command Exec and Serial Sync
        - As shown previously, the firmware has a sort of command shell which can be accessed thought serial console.  It turns out that we can send commands to it directly through CmdExec command. Similarly, we can read the serial output buffer via SerialSync command. This enables implementing a simple shell over wifi. Check out the `shell.py` script to see how it works. 
Built-in shell

With ability to unlock the camera over wifi, and with combined ExecCmd and SerialSync commands there’s no need to open the camera or do any physical modifications in order to explore the built in shell. The following is the complete list of commands with their associated descriptions:


- List of commands with help
- Highlights 
    - Rfitamron 
    - Rfisettings
    - Rficapture
Tools and utils
- Shell
- Demo
- LiveView

