https://github.com/Hello1024/brushless-encoder

Use a DC brushless motor as a rotary encoder to measure angles
==============================================================

This project measures unintended properties of a brushless motor which change as the motor angle is turned to determine it's angle.

[![Demo](https://img.youtube.com/vi/YSzVXCdpdm4/0.jpg)](https://www.youtube.com/watch?v=YSzVXCdpdm4)

The project is currently *beta*, and as such has limitations:

* It must be manually tuned for each type of motor/power supply/controller (see instructons below)
* It makes an annoying buzzing sound.
* It cannot produce accurate results when the motor is turning fast (1 rad/s max on my motor)
* It has 1500 counts per turn


All of these are removable limitations - patches welcome!

Getting Started
---------------

1.  Get a brushless DC motor and motor controller (ESC) compatible with `tgy.hex` from SimonK (list of controllers is [here](https://docs.google.com/spreadsheet/ccc?key=0AhR02IDNb7_MdEhfVjk3MkRHVzhKdjU1YzdBQkZZRlE))

2.  Make sure you can flash the firmware (directly, the bootloader is untested), and have access to the microprocessors Port D2 (TXD) pin to get outputs out in serial form.   I used an arduino (File > Examples > ArduinoISP) to do the flashing, but other programmers will work with modification.   Hook up the ESC to the motor and a 6-12 volt power supply *with a 10 ohm series resistor for safety*.

3.  You need a linux machine to build on (cygwin windows or mac may work, but untested).  Run `sudo apt-get install build-essential avrdude avra`.

4.  Clone this repro and run `make program_avrisp2_tgy`.   That will compile all the stuff and program the ESC.

5.  Listen to the data on the serial port.  (You can do that by programming the arduino with a blank project, then connecting it's TX pin to the ESC TX pin, and data from the ESC will make it to the computer.  Listen at 9600 baud).

6.  Record the output of the serial port while (very slowly) turning the motor one complete turn for calibration.   The data is hexadecimal CSV of four measured motor parameters and a final calculated angle.   Verify that when the motor is stationary all four parameters stay roughly unchanged (if not, some timing or voltage parameters are wrong in the code, and you will need a scope to help fix it, which is beyond this guide).

7.  Load up the csv data into Excel/Libreoffice (`=hex2dec()` is a useful function!).  Discard the last column.  Graph it and find the repeating period, then strip down the data to one complete cycle.  Thin out (remove) rows so you have 256 rows.   Save your new 4 column, 256 row file (now in decimal) overwriting `lookup2.csv`.

8. `make program_avrisp2_tgy` again.

9.  You're now done!   The last column of the serial output is now the angle the motor is at (mod 256).


How it works
------------

Motor coil inductance changes with the angle of the motor, since most motors are imperfect (for one thing, they usually use flat rather than curved neodynium magnets, which get closer and further from the stator as the motor rotates).

We measure the inductance of motor coils by putting a pulse through the coil, then letting the pulse dissipate through the mosfet body diode.   Since the current decays through the fixed motor winding resistance, when the inductance is lower, the pulse dissipates quicker.

We can easily measure if the current has reached zero yet by observing the voltage across the body diode (if it's -0.7V, current is still flowing, if it's 0, current has stopped.  It's surprisingly binary).

The ADC on these chips runs super slowly, so we can actually start the ADC sampling *before* sending the pulse.   We time the pulse so we sample where we *expect* the body diode will stop conducting, and then update our estimate with the result.

The first four columns on the output are these time measurements (in units of about 0.3us).

On the computer, we make a lookup table of which measurements match which angles, compile that into the binary, and then search for the closest matching row when we want to know the angle (the 5th column).



Improvements
------------

 * **Autogenerating the table**:  It should be possible for a startup routine to turn the motor slowly and take measurements to populate it's own calibration table.
 * **Quicker ADC sampling**:  It should be possible to run the ADC in continuous mode, so that it can run at ~20KHz rather than ~100Hz.   This will make it inaudible.
 * **Use an ISR**:  The current implementation is slowed dramatically via serial output delays.  It would probably be 10x faster if not blocked by outputting.
 * **Use binary search to find the timing**:  Currently we implement a simple increment/decrement logic for ever sample, but that severely limits how fast we can deal with the motor turning.  Binary search should make it 3x faster.
 * **Have a momentum model in the table lookup**:  The current position search assumes the motor position could be random at every sample.  A momentum model would make it significantly more accurate with ambiguous inputs (especially multi-pole motors where each pole is slightly, but not sufficiently different)
 * **Allow simultaniously applying torque to the motor**:  This is complex and would require a model which can quickly apply a PWM pulse to get the motor current down very low, then measure it dissipate, then another pulse to bring it back to operating current.    Alternatively, an alternate mode of operation is to measure variation in operating motor current through measuring voltage drop across the R_DSON of the MOSFETS, but the ADC likley doesn't have sufficient resolution for this.
 * **Combine this method with zero crossing detection for fast spinning motors**:  When a motor is moving fast, it's back-EMF is considerable and easy to detect, which allows accurate phase measurements.   My method is good for very slow movements/stationary where the back-EMF is undetectable.  A smart model would blend both.


Development
-----------

It is unlikley I will do further work on this, but I will accept pull requests.  Beware though, the whole thing is super hacked together, and you might be better reading "How it works" above and rewriting.

Credits
-------

This is a kludged together hack on top of SimonK (http://0x.ca/tgy/).  Used mostly for the pin definitions, setup logic and utility functions.  I had dreams of turning SimonK into a low speed controller too for fine grained 'move back by half an inch' control of RC cars, but it turns out I don't like hand-writing assembly enough!   Pull requests for cleanup welcome!

