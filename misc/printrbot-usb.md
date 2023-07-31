title: Quickie - Printrboard USB connection failure

# Quickie - Printrboard USB connection failure

This is a quick one. Some time ago I found a broken Printrbot Simple Maker's Kit at the FabLab. This
is a 3D-Printer Kit from the early days of RepRap. The structure is made of laser-cut wood sheets
kept together with screws and nuts. It has a small 10cm^3 print area, but it's a nice little
machine overall. It doesn't have an lcd screen, and prints must be sent via USB. It does a SD Card
slot though.

The reason the printer was abonded and deemed broken is that USB connections would always fail in an
unpredictable manner. Sometimes they would work just fine, sometimes the board would not show up on
the computer (different computers, different OSes) but it would show up if booted into the DFU
bootloader by shorting the appropriate headers on the board.

My first idea was to upgrade to a newer version of Marlin, ideally Marlin 2.x, since probably the
firmware on this board is still on some fork of Marlin 1.x. I came across this <a
href="https://github.com/stonehippo/printrbot_simple_maker_1405_firmware"> repo </a> which featured
Marlin 2.0.9.3 configuration files and even a precompiled firmware and a utility to flash it
named 'PrintBot FirmwareUpdatr', originally from Printrbot themselves, fixed by the repo author to
work on newer versions of macOS. Unfortunately yes, it does only work on macOS and I don't have
access such a machine at the moment. However, <a
href="https://reprap.org/wiki/Printrboard#Loading_Firmware_.28Linux.29"> reprap.org has instructions
for flashing the firmware</a> on Printrboards (yes, that's what Printrbot boards are called).

Even by flashing the new firmware, USB connections were still finnicky, so I decided to ditch the
Printrboard all together and upgrade to a spare ramps 1.4 I had laying around, complete with a LCD
screen and sdcard reader.<a
href="https://www.reddit.com/r/PrintrBot/comments/z5z5ya/first_benchy_on_my_printrbot_simple_makers_kit/"> I even posted about it on reddit</a>.
I later reused the ramps for my Anet A8, so the Printrbot remained unused (and unusable).

Today I decided to give the Printrboard another go. I soon discovered that it needs 12V power to turn
even on and connect to usb. Reading the firmware flashing guides again I discovered however that the DFU
bootloader, which seemed to be part of the problem, is writter into flash memory and is not bundled
with the microcontroller itself. This time, I decided to use plain and simple ICSP to flash the
firmware, and remove the DFU bootloader all together.

Below is a picture of the board and a closeup of the ICSP connector. The layout the connector is
standard. In this photo, the GND pin is the leftmost of the bottom row, circled in red.

<figure class="half">
<img class="half-img" src="/resources/misc/printrboard.jpg" style="width:40%" /> 
<img class="half-img" src="/resources/misc/printrboard-icsp.jpg" style="width:30%"/> 
</figure>

I used a USBASP programmer to flash the firmware from stonehippo's repo. Here's the command I used,
  readapated from <a href="https://github.com/Printrbot/printrboardmodernmarlin"> Printrbot Model
  Marlin repo</a>:

    avrdude -c usbasp -p at90usb1286 -U flash:w:simple_maker_1405_marlin_2.0.9.3.hex:i

Obviously change `usbasp` with the name of the programmer you are using and the name of the file if
needed.

After flashing the firmware via ICSP the board was correctly recognized as
    
    Bus 003 Device 081: ID 16c0:0483 Van Ooijen Technische Informatica Teensyduino Serial

by my computer. It also works fine after microcontroller resets, power cycles and usb reconnection.
It is correctly recognized by Cura, which means printing via usb works flawlessly.

I did some test prints and they turned out quite good considering the state I found this printer in.
There are some obvious artifacts caused by mechanical problems: a bit of deformation along the
Y-axis, the wood looks like it shrinked in some places and the Z motor rod is flexed. E-Steps also need to be recalibrated.


### Reference material

- <a href="https://github.com/stonehippo/printrbot_simple_maker_1405_firmware/">
https://github.com/stonehippo/printrbot_simple_maker_1405_firmware</a>
- <a
href="https://github.com/Printrbot/printrboardmodernmarlin">https://github.com/Printrbot/printrboardmodernmarlin</a>
- <a
href="https://reprap.org/wiki/Printrboard#Loading_Firmware_.28Linux.29">https://reprap.org/wiki/Printrboard#Loading_Firmware_.28Linux.29</a>

