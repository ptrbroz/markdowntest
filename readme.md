# FPGA Generator of phase-shifted digital signals

This folder contains a Quartus project implementing a generator of phase-shifted signals. The generator has multiple output channels. The duty cycle of each channel, as well as the relative phase shift between channels can be set. In addition, the frequency (which is shared by all channels) can be changed. 

## Hardware

This version of the generator runs on Altera DE0-Nano developement board (Cyclone IV).

## Features

* 64 digital output channels
* Configurable output frequency shared among channels [TODO: range]
* Configurable phase shift of individual channels (resolution 1/360 of signal period)
* Configurable duty cycle of individual channels (resolution 1/360 of signal period)
* Communication via UART

## Communication protocol

At the moment of writing, the communication is one-way. Each command sent via UART starts with a 3 byte opencode, followed by a number of data bytes and ends with a 3 byte closecode.

### Reconfiguring channel phases and duty cycles

|Phase & Duty comm codes|
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

The device accepts bytes, not 9-bit integers via UART. Therefore, the first data byte sent must correspond to the first 8 bits of of the 0th duty integer, the second byte sent will correspond to the highest bit of 0th duty integer followed by the first 7 bits of 0th duty phase, et cetera (see example below).

When reconfiguring the channels, always send all 144 data bytes before sending the closecode. The device does not verify the number of bytes received; sending the closecode prematurely may lead to unexpected results due to the use of a shift register.

### Example of communication: Channels reconfiguration

Let's assume that we wish to configure the output signals according to following table:

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

When viewed bit by bit, we get

<table>
  <tr>
    <th colspan="3">a</th>
    <th>d</th>
    <th></th>
  </tr>
  <tr>
    <td>b</td>
    <td>c</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>









