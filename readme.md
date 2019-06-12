# FPGA Generator of phase-shifted digital signals

This folder contains a Quartus project implementing a generator of phase-shifted signals. The generator has multiple output channels. The duty cycle of each channel, as well as the relative phase shift between channels can be set. In addition, the frequency (which is shared by all channels) can be changed. 


## Features

* 64 digital output channels
* Configurable output frequency shared among channels [TODO: range]
* Configurable phase shift of individual channels (resolution 1/360 of signal period)
* Configurable duty cycle of individual channels (resolution 1/360 of signal period)
* Communication via UART

## Hardware

This version of the generator runs on Altera DE0-Nano developement board (Cyclone IV).

## Software

Developed using Quartus II 64-bit Version 13.0.1.


## Communication protocol

At the moment of writing, the communication is one-way. Each command sent via UART starts with a 3 byte opencode, followed by a number of data bytes and ends with a 3 byte closecode.

### Reconfiguring channel phases and duty cycles

||Phase & Duty comm codes|
--- | ---
Opencode | 255 255 240
Closecode | 255 255 241

The phase offset and duty cycle of each channel is represented by two 9-bit integers, whose values range from 0 to 360.

Each unit (or degree) corresponds to 1/360 of the signal period. Therefore, a channel with duty set to 180 will have a 50% duty cycle. A channel with phase set to 90 will lag behind those with duty set to 0 by 1/4 of the signal period.

As there are 64 channels, a total of 64x2x9 = 1152 bits (144 bytes) of data need to be transferred to reconfigure the channels.

After receiving the opencode, the device shifts following bytes into a settings shift register. Upon receiving the closecode, all channels are simultaneously reconfigured using the settings in said register.

Note that the output channels keep their original configuration until the closecode is received.

The 9-bit integers within the register are arranged in alternating order of phases and duty cycles, starting with the phase of 63rd channel and ending with the duty of 0th channel. See table below:

|Shift in =>|63rd phase|63rd duty|62nd phase|62nd duty|...|1st phase|1st duty|0th phase|0th duty|Shift out=>|
|-|-|-|-|-|-|-|-|-|-|-|

The device accepts bytes, not 9-bit integers via UART. Therefore, the first data byte sent must correspond to the first 8 bits of of the 0th duty integer, the second byte sent will correspond to the highest bit of 0th duty integer followed by the first 7 bits of 0th phase, et cetera (see example below).

When reconfiguring the channels, always send all 144 data bytes before sending the closecode. The device does not verify the number of bytes received; sending the closecode prematurely may lead to unexpected results due to the use of a shift register.

### Reconfiguring the output frequency

||Frequency comm codes|
--- | ---
Opencode | 255 255 242
Closecode | 255 255 243

The change of output signal frequency is accomplished by reconfiguring the [ALTPLL (phase-locked loop)](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/ug/ug_altpll.pdf) entity used in the design.

The PLL uses the onboard 50MHz clock to derive the clock used for driving the logic of channels. Let's call the frequency of this clock F_Logic. To enable the resolution of 1/360 of signal period, F_Logic is always 360 times higher than the frequency of the output signal (F_Out). 

By reconfiguring the PLL, we will be changing it's internal counter variables M, N and C. The output frequency of the PLL, which is the aforementioned F_Logic, can be calculated as:

F_Logic = (50*M)/(N*C) [MHz]

At the time of writing, the PLL is reconfigured by means of a 144-bit long scanchain. The description of the scanchain format will follow. I also reccomend that you see [this generated sample scanchain](scanchainExample/36div125.mif). The settings in that generated scanchain correspond to values M = 36, N = 5 and C = 25, which in turn corresponds to F_Logic of 14.4 MHz (and F_Out of 40kHz).

The order of bits follows (see below table for details).

|Bits|todo|
--- | ---
0 to 1 | Reserved 
2 to 9 | Loop Filter
10 to 14| Reserved
15 to 17| Charge Pump
18| N counter Bypass
19 to 26| N counter High Count
27| N counter odd Division
28 to 35| N counter Low Count
36| M couner Bypass
37 to 44| M counter High Count
45| M counter Odd division
46 to 53|M counter Low Count
54| C counter Bypass
55 to 62| C counter High Count
63| C counter odd Division
64 to 71| C counter Low Count
72 to 89|   Same as 54 to 71
90 to 107|  Same as 54 to 71
108 to 125| Same as 54 to 71
126 to 143| Same as 54 to 71





### Example of communication: Channels reconfiguration

Let's assume that we wish to configure the output signals according to the following table:

Variable | Value [°]
---     | ---
Channel 0 duty | 180
Channel 0 phase | 90
Channel 1 duty | 180
Channel 1 phase | 0
Channel 2 duty | 270
Channel 2 phase | 45
Other channels phase | 0
Other channels duty | 0

Therefore, we wish for the shift register to contain following 9-bit integers in this order:

|63rd phase|63rd duty|...|2nd phase|2nd duty|1st phase|1st duty|0th phase|0th duty|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|0|0|0...0|45|270|0|180|90|180|

When viewed bit by bit, we get the following table: (only the last 7 bytes are shown)

<table>
  <tr>
    <th colspan="56">9-bit integers</th>
  </tr>
  <tr>
    <th colspan ="2"></th>
    <th colspan="9">45</th>
    <th colspan="9">270</th>
    <th colspan="9">0</th>
    <th colspan="9">180</th>
    <th colspan="9">90</th>
    <th colspan="9">180</th>
  </tr>
  <tr>
    <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>1</td> <td>0</td> <td>1</td> <td>1</td> <td>0</td> <td>1</td> <td>1</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>1</td> <td>1</td> <td>1</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>1</td> <td>0</td> <td>1</td> <td>1</td> <td>0</td> <td>1</td> <td>0</td> <td>0</td> <td>0</td> <td>0</td> <td>1</td> <td>0</td> <td>1</td> <td>1</td> <td>0</td> <td>1</td> <td>0</td> <td>0</td> <td>1</td> <td>0</td> <td>1</td> <td>1</td> <td>0</td> <td>1</td> <td>0</td> <td>0</td>
  <tr>
    <th colspan="8">5</th>
    <th colspan="8">176</th>
    <th colspan="8">224</th>
    <th colspan="8">2</th>
    <th colspan="8">208</th>
    <th colspan="8">180</th>
    <th colspan="8">180</th>
  </tr>
  <tr>
    <th colspan="56">Bytes to send</th>
  </tr>
</table>

Therefore, to configure the channels to the desired settings, we need to send following bytes over UART:

255 255 240 180 180 208 2 224 176 5 XXX 255 255 241

(replace XXX with 137 bytes of value 0)





