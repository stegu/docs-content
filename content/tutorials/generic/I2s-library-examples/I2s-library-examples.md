---
title: I2S Library Examples
description: 'Enables to use the I2S protocol on SAMD21 board like Arduino Zero, Arduino MKRZero and Arduino MKR1000.'
tags: [I2S, Sound]
author: Arduino
---

The first example will show you how to read and visualize audio data coming from an I2S microphone. The second example shows you how to generate a simple tone using a SAMD21 based board and an I2S DAC.

## Hardware Required

- An I2S compatible Arduino board, such as a [Zero](/hardware/zero), [MKR1000](/hardware/mkr-1000-wifi), [MKRZero](/hardware/mkr-zero), [Nano 33 IoT](/hardware/nano-33-iot) or [Nano 33 BLE](/hardware/nano-33-iot).
- For the first example, an I2S microphone (i.e [ICS43432](http://www.invensense.com/wp-content/uploads/2015/02/ICS-43432_DS.pdf)), or a board with a built-in microphone such as the [Nano 33 BLE Sense](/hardware/nano-33-ble-sense).
- For the second and third examples, an I2S stereo output device, e.g. a MAX98357A amplifier.
- A suitable speaker. Note that even a small amplifier can pack quite a punch. The 98375A can push up to 3W into a 4 Ohm speaker, or 1W into 8 Ohms.

## Circuit

To run the first example, you simply have to connect the board and the I2S microphone using the I2S bus as shown in the image. The image is for MKRZero; you find the proper pins for Zero, MKR1000 and Nano 33 at the beginning of the sketch, in the comments.

![Circuit for Input Serial Plotter.](assets/I2SMIC.png)

To run the second and third examples you simply have to connect the board and the I2S DAC using the I2S bus as shown in the image. The image is for MKRZero; you find the proper pins for Zero, MKR1000 and Nano 33 at the beginning of the sketch, in the comments.

![Circuit for I2S Simple Tone.](assets/I2SDAC_PINOK.png)


### I2S Input Serial Plotter

This example shows you how to read and visualize on the serial plotter I2S audio data coming from an I2S microphone.

```arduino
/*

 This example reads audio data from an Invensense's ICS43432 I2S microphone

 breakout board, and prints out the samples to the Serial console. The

 Serial Plotter built into the Arduino IDE can be used to plot the audio

 data (Tools -> Serial Plotter)

 Circuit:

 * Arduino Zero, MKR family and Nano 33 IoT

 * ICS43432:

   * GND connected GND

   * 3.3V connected to 3.3V (Zero, Nano) or VCC (MKR)

   * WS connected to pin 0 (Zero) or 3 (MKR) or A2 (Nano)

   * CLK connected to pin 1 (Zero) or 2 (MKR) or A3 (Nano)

   * SD connected to pin 9 (Zero) or A6 (MKR) or 4 (Nano)

 created 17 November 2016

 by Sandeep Mistry

 */

#include <I2S.h>

void setup() {

  // Open serial communications and wait for port to open:

  // A baud rate of 115200 is used instead of 9600 for a faster data rate

  // on non-native USB ports

  Serial.begin(115200);

  while (!Serial) {

    ; // wait for serial port to connect. Needed for native USB port only

  }

  // start I2S at 8 kHz with 32-bits per sample

  if (!I2S.begin(I2S_PHILIPS_MODE, 8000, 32)) {

    Serial.println("Failed to initialize I2S!");

    while (1); // do nothing

  }
}

void loop() {

  // read a sample

  int sample = I2S.read();

  if (sample) {

    // if it's non-zero print value to serial

    Serial.println(sample);

  }
}
```

### I2S Simple Tone

This example shows you how to generate a simple tone using a SAMD21 based board (MKRZero, MKR1000 or Zero) and an I2S DAC like the adafruit MAX98357A.

```
/*

 This example generates a square wave based tone at a specified frequency

 and sample rate. Then outputs the data using the I2S interface to a

 MAX98357 I2S Amp Breakout board.

 Circuit:

 * Arduino Zero, MKR family and Nano 33 IoT

 * MAX98357:

   * GND connected GND

   * VIN connected 5V

   * LRC connected to pin 0 (Zero) or 3 (MKR) or A2 (Nano)

   * BCLK connected to pin 1 (Zero) or 2 (MKR) or A3 (Nano)

   * DIN connected to pin 9 (Zero) or A6 (MKR) or 4 (Nano)

 created 17 November 2016

 by Sandeep Mistry

 */

#include <I2S.h>

const int frequency = 440; // frequency of square wave in Hz

const int amplitude = 500; // amplitude of square wave

const int sampleRate = 8000; // sample rate in Hz

const int halfWavelength = (sampleRate / frequency); // half wavelength of square wave

short sample = amplitude; // current sample value
int count = 0;

void setup() {

  Serial.begin(9600);

  Serial.println("I2S simple tone");

  // start I2S at the sample rate with 16-bits per sample

  if (!I2S.begin(I2S_PHILIPS_MODE, sampleRate, 16)) {

    Serial.println("Failed to initialize I2S!");

    while (1); // do nothing

  }
}

void loop() {

  if (count % halfWavelength == 0) {

    // invert the sample every half wavelength count multiple to generate square wave

    sample = -1 * sample;

  }

  // write the same sample twice, once for left and once for the right channel

  I2S.write(sample);

  I2S.write(sample);

  // increment the counter for the next sample

  count++;
}
```

### I2S Buffered Output

When you run the I2S library, the loop() function executes once for every sample, which makes it convenient to program simple demos. You just handle one sample per loop execution, and that's it. However, for real world sound applications, this is usually not enough. If your loop ever takes longer to execute than the time between two audio samples, things will go wrong. If you are handling any other input and output or doing some computations in your code, you can easily end up taking more time than, say, the 45 microseconds you have available between each sample at a sample rate of 22 kHz.

If you are missing samples while handling sound input, you will read less data than the device is sending, which means that data will accumulate in the input buffer, which is typically a few hundred samples in size. Your input will lag behind more for every sample you miss, and after a while the buffer will fill up. This has an unpredictable result which will definitely sound bad.

If you are generating sound output, a failure to generate a sample in time will force the output device to instead use a "panic sample" with a default value, usually 0, which will cause nasty pops and crackle in the sound output if you miss a few samples here and there, and sound stutter if you miss a longer sequence of samples.

The solution to the input problem is to read all samples that are available in the input buffer and make the loop handle each of them in turn. For output, you should generate a small batch of samples and place them in a buffer, and keep sending new batches of samples *before* the buffer becomes empty. While you still need to process buffered samples at an *average* speed to match the sample rate, you do not have to worry about some iterations of the loop possibly taking more than one sample interval to finish. This could happen because of other things going on with the hardware and software. This could be interrupts that are often beyond your control, or it could be your own code to handle non-I2S input and output in response to user interaction.

The I2S library has a simple mechanism to handle buffered data, and it comes in three parts: functions that transfer multiple samples, functions that tell you how much data is already in the buffers, and finally functions to register a *callback function* to signal when the previous batch of data has been transferred. It's perfectly fine to use the buffered API without the callback function. Handling buffered output goes like this:

* For each iteration of the main loop, check how many samples remain in the output buffer.
* If there are fewer samples than some safe level threshold, generate and send a new batch of samples.

A tutorial for this would be welcome if you have the time to write it.

* (Optional: check if any samples are available in the input buffer)
* Either read all samples that are available in the input buffer, or ask for a smaller number of samples if you need the loop to finish quickly
* If any samples were read, process each of them in a loop

*Note that we do not present any code for handling buffered input. A tutorial section on this would be welcome if you have the time to write it.*

If you register a callback function for writing, it will be executed when a batch of data has been sent and your local array variable can be safely overwritten with new data. A callback function for reading will be executed as soon as there is data available for your program to read from the input buffer. You don't have to use the callbacks, but they come in handy in many situations. Note that the callbacks are executed when data has been transferred to the input and output *buffers* for the I2S device. For sound input, this happens more or less immediately after new data is sampled, but for output it happens when the batch of samples you sent most recently has been placed in the output buffer. This typically happens a few milliseconds before the sound is actually played, depending on your threshold for sending new samples,  the size of the output buffer, and the sample rate.

The example below shows how to use buffered I2S output to use a somewhat more interesting waveform to changes the harmonics of a tone over time.

```
/*

 This example generates a tone at a specified frequency and sample rate,
 using I2S buffered output and changing the harmonics of the tone over time.

 Circuit:

 * Arduino Zero, MKR family, or Nano 33 IoT/BLE/BLE Sense

 * MAX98357:
   * GND connected to GND
   * VIN connected to 5V
   * LRC connected to pin 0 (Zero), pin 3 (MKR) or pin A2 (Nano)
   * BCLK connected to pin 1 (Zero), pin 2 (MKR) or pin A3 (Nano)
   * DIN connected to pin 9 (Zero), pin A6 (MKR) or pin 4 (Nano)

 created 30 June 2022
 by Stefan Gustavson
 */

#include <I2S.h>

/* Still to be written, need to clean up the code and isolate the I2S stuff */

```
