# PWM and Timers on modern AVRs and their usage in this core
This document is divided into two sections. The first one simply describes the available timers, and what they are capable of (by "simply describes" I don't claim to have made a simple description, only that the purpose is simple. and that is to describe the timers). The second section describes how they are used by this core in particular. The first section is shared by DxCore and megaTinyCore. The second contains many sections specific to one core or another.

## Quick answer: Which PWM pins should I use?
TCA or TCD pins; these timers are much better for generation of PWM

## Background: The Timers on the AVR Dx-series and Ex-series parts
This applies to the tinyAVR 0/1/2-series, megaAVR 0-series, and AVR DA, DB and DD-series, and likely other future modern AVRs. There are very few differences between the implementations on the different families.

### TCA0 - Type A 16-bit Timer with 3/6 PWM channels
This timer is the crown jewel of the modern AVR devices, as far as timers go. It can be operated in two very different modes. The default mode on startup is "Normal" or `SINGLE` mode - it acts as a single 16-bit timer with 3 output compare channels. It can count in either direction, and can also be used as an event counter (ie, effectively "clocked" off the event), is capable of counting up or down, generating PWM in single and dual slope modes, and has 7-output prescaler. In terms of PWM generation, a TCA in SINGLE mode is on the same level as the classic avr 16-bit Timer1 (though most classic AVRs only had 2 outputs per timer - there the third output was a rare feature reserved for top-end parts like the 2560). The newly added features aren't ones that are particularly relevant for most Arduino users. TCA0 can generate events or interrupts on compare match for each channel (independently), as well as on an overflow.

The Type A timer can be also be set to split mode to get six 8-bit PWM channels (this is how it is configured by default by megaTinyCore and DxCore - since `analogWrite()` is only 8-bit, we might as well double the number of channels we get right? In split mode the high and low bytes of the timer count `TCA0.SINGLE.CNT` register become `TCA.SPLIT.LCNT` and `TCA.SPLIT.LCNT`; likewise the period and compare registers *in SINGLE mode, these are 16-bit registers; accessing them uses the temporary register. In SPLIT mode, they are 8-bit registers!*. Unlike the TCB in PWM mode, this works correctly. The frequency of the two "halves" of the timer is always the same (prescaler is shared between the two). However, the HPER and LPER registers can be used to adjust the period of the two "halves" independently. This means that you could change the number of counts per cycle, thus allowing for the two halves to output different frequencies of PWM - albeit at a cost in resolution. There is an example of this under the link below).

There are a few examples of using TCA0 to generate PWM at specific frequencies and duty cycles in the document on [Taking over TCA0](./TakingOverTCA0.md)

The pins that the type A timer is directed to can be mapped to a variety of different pins.
* On ATtiny parts, each Waveform Output (WO) pin can be controlled separately.
* On ATtiny parts with 14+ pins, WO0-2 can be mapped to either PB0-2 or PB3-5, and WO3-5 (the split mode only ones) can be mapped to PA3-5 or PC3-5 (if the part has those pins).
* On ATtiny parts with 8 pins, they are mapped to (in order) PA3-or-PA7, PA1, PA2, and PA3. Only the first 4 are available, and WO0 must be remapped in order to use WO3.
* On everything else, each TCA can be pointed at one group of pins from a list of options. TCA0 can (on all parts thus far released) be directed to pins 0-5 of any port, while TCA1's WO0-5 channels are only available on PB0-5 or (on 64-pin parts only - and not DA-series as of mid-2022 due to errata) PG0-5, and TCA1's WO0-2 can be output on pins PC4-6, or (on DB-series parts - DA's are again impacted by errata here) PE4-6. The Ex-series adds options for PD4-6 and PA4-6.

#### Events and CCL channels
On all parts, the TCA can be used to count events (counting either positive edges, or all edges), instead of system clocks (in this mode, the prescaler is ignored, and the events must be longer than 2 system clocks or generated synchronously to the system clock to guarantee the event input is seen) or to count only while the event channel is HIGH, or to reverse direction while the event input is high. On 2-series tinyAVR parts, AVR Dx and AVR Ex, there is a second event input dedicated to restarting the timer compare cycle on rising edge, any edge or have the timer only count or to reverse direction when an event is HIGH. **None of these event inputs are not available in SPLIT mode**. There is no restart-on-event input on the tinyAVR 0/1-series or megaAVR 0-series.

The TCA also generates 4 event outputs - compare match for the first three channels, and an overflow event (in split mode, the overflow event is replaced by two underflow events, for a total of five event generators). These are all pulse events 1 system clock long, synchronous to the system clock (among other things, this means they can't be used to remap the PWM outputs). However the first three WO output levels are ALSO available to the CCL LUTs, so they *can* be used for remapping WO0-2 to any LUT output pin.

Remember that the pin invert (INVEN) effects event inputs and outputs. So if you want to count negative edges, or count prescaled clock cycles while the pin is LOW, etc - just invert the input pin.

#### Interrupt note
When firing interrupts from a TCA, you must *ALWAYS* manually clear the intflags. It is not done for you by the hardware.

### TCBn - Type B 16-bit Timer
The type B timer is what I would describe as a "utility timer" - It is a very good utility timer. In this role, it can take one of no less than 7 modes, most of which require using the event system. They can also be used as a single channel 8-bit PWM timer. Unfortunately, they have limited selection of clock prescalers: only the system clock, the system clock divided by 2, or a clock that is already being generated for TCA0 or TCA1, making these very unappealing PWM timers. 2-series, Dx, and Ex-parts can also count rising event edges using the second event input, which is not available on the tinyAVR 0/1-series or megaAVR 0-series. In terms of input capture, the TCBs are able to do everything a classic AVR timer1 could do.

#### Periodic interrupt
This does exactly what you would expect - counts up to CCMP, and resets to 0 and generates an interrupt. This is how it's used for millis timekeeping. Even with the limited prescaling selections (1, 2, or the TCA prescaler), being able to count up to 64k makes this a lot less limiting than it would be on an 8-bit timer.

#### Input Capture on Event
Timer counts continually from 0 to 65535. This is very much like the old ICP mode on classic AVRs. Upon the specified edge (selectable) of the the event connected to the timer, it will copy the current count to CCMP and fire an interrupt. It is the responsibility of the user code to know when the last input capture occurred and subtract those values.

#### Input Capture Frequency Measurement
As above, but at the specified edge, the timer count is automatically reset (after copying count to CCMP of course), so it starts timing the next event. As the name implies, this is of great use for measuring the frequency of an input. Does not work with prescaled clocks on some tinyAVRs (see silicon errata)

#### Input Capture Pulse Width Measurement
One edge restarts the timer. The other edge copies the value and fires the interrupt, thus making it perfect for timing the length of pulses.

#### Input Capture Frequecy And Pulse Width Measurement
This one is the weird one. Does not work with prescaled clocks on some tinyAVRs (see silicon errata listings). On one edge, it starts the timer. On the other edge, it copies the count to CCMP *and continues to count* until the next edge. Only then will the interrupt fire (if enabled). Whether via polling or interrupt, when you see that this has happened, you need copy the count value someplace safely, then the CCMP value. As soon as CCMP has been read, when it next sees the "start count" edge, it will reset the timer and start counting. Thus, while great for measuring PWM of varying frequency and duty cycle where the duty cycle does not change drastically from cycle to cycle, it is not useful if you need to record every cycle (for that, you're forced to use Input Capture on Event mode, and have your interrupt very quickly switch the edge to which it is sensitive).

#### Single-shot
In addition to the input capture modes, they can be used for precise generation of a pulse with timing accuracty down to a single clock cycle. In this mode, the output pin will go high or low (selectable) in response to the specified edge of an event (though see below). The counter will count up to it's CCMP value, then reset both it's count and it's pin (these are the same output pins as are used for PWM). It is NOT NECESSARY to use the pin itself - you can use this instead to generate a event or interrupt after a delay has passed! Likewise, it could be triggered with an event channel connected to nothing, and manually kicked off with a software event. Without relying on TCA's prescaler, this can generate puloes of around 130ms in length. On the AVR DA datasheet and the DB datasheet clarification, it is stated that when EDGE is 1, it triggers on *any* edge, not just negative ones, - it's not clear if that applies to other parts or not; it may very well apply to all parts and simply not have been updated for the other parts. This may sound like a downgrade; if you think that, though, you've forgotten about the INVEN bit in PINnCTRL: it is applied *before* the signal heads to the event generator, so if you want to trigger on negative edges you just invert the pin and trigger on positive edges.

#### Timeout Check
In this mode, one type of edge resets the counter to 0 and starts it counting, while the other edge stops the counting. In this mode the interrupt fires when the count passes CCMP - that's the "time out" that it's checking for. While this may at first sound like an odd feature, it is in fact extremely useful in combination with the custom logic and event system. The magic is that because it is both reacting to and can output an event you can use it to do things in the background. Among other things, it is exceptionally useful for working with bespoke single wire protocols.

#### 8-bit PWM mode
They can be pressed into service as a rather poor PWM timer. TCB in PWM mode is 8-bit only. When used for PWM, they can only generate 8-bit PWM, despite being a 16-bit timer, because the 16-bit `TCBn.CCMP` register is used for both the period and the compare value in the low and high bytes respectively. They always operate in single-slope mode, counting upwards, the selection of clock options is limited in the extreme: system clock, half the system clock, or the frequency depends on that of the TCA (or one of the TCA's on parts with more than one).  In other words, **the type B timers are not very good at generating PWM**. Note also that `TCBn.CCMP` is effected by silicon errata on all available silicon as of mid 2022: It still acts like a 16-bit register, using the temp register for access, so you must read the low byte first, then high byte, and always write the high byte after the low one, lest it not be written or a bad value written over the low byte!

#### Extra features on 2-series and Dx/Ex-series
The 2-series and Dx parts add three upgrades, two useful, and the other less-so. The less useful one is a separate OVF event/interrupt source. I find this to be of dubious untility - likely the best use of the separate OVF bit is as a 17th bit in input capture mode, but this can generally be done without using it as an interrupt.

A much more interesting option is the clock-on-event option: The TCBs now have a second event user, which can be selected as the clock source! Combined with the CCL, this can, for example, be used to get a prescaled system clock into the TCB different from that of a TCA (see the Logic library documentation and examples for discussion of how the CCL filter and synchronizer can be used to generate s prescaled clock).

Finally, on these parts, you can combine two timers to make a single, monster, 32-bit input capture timer using the CASCADE feature. To do this, you configure the high byte timer such that it is clocked from the OVF event of the low timer, and the CASCADE bit enabled. When the event fires, both timers will get their CCMP values saved and/or be stopped, permitting incredible precision (or just really long things) to be measured. 2<sup>32</sup> system clocks, which is around 3 minutes at 24 MHz. Prescaling the the low timer by 2 would double that. Ever wanted to time approximately 6 minute long event to a resolution of tenths of a microsecond? Er, sorry, I meant tenths of some period of time that is within a percent or so of a microsecond, unless using an external clock or something. This could be more interesting if you were, instead of time, counting events on the first timer, and 16 bits just wasn't enough.

#### Intflag summary
Sometimes the intflags automatically clear, but usually they don't. Be sure to clear them manually in the cases where it isn't done automatically!

The OVF flag only exists on 2-series, DA, DB, DD, and EA parts

| Mode              | CAPT Reset by | OVF reset by | OVF set under normal use   |
|-------------------|---------------|--------------|----------------------------|
| Interrupts        | User code     | User code    | Only when CCMP set below CNT and allowed to roll over |
| Timeout Check     | User code     | User code    | Only when timeout goes on for a a long time.  |
| Input capt. Event | Read CCMP     | User code    | Every time the counter rolls over. Rarely needed |
| Input capt. (all) | Read CCMP     | User code    | When counter rolls over. Effectively 17th bit |
| Singleshot        | User code     | User code    | No.                        |
| PWM               | User code     | User code    | At end of every PWM cycle  |


### TCD0 - Type D 12-bit Async Timer
The Type D timer, is a very strange timer indeed. It can run from a totally separate clock supplied on EXTCLK, or from the unprescaled internal oscillator - or, on the Dx-series, from the on-chip PLL at 2, 3, or even 4 times the speed of the external clock or internal oscillator! (the 4x multiplication option is unofficial though - it was in the very earliest headers, but was neverin the datasheet andwas removed from headers shortly thereafter. It works under favorable conditions though). It was apparently designed with a particular eye towards motor control and SMPS control applications. This makes it very nice for those sorts of use cases, but in a variety of ways ,these get in the way of using it for the sort of things that people who would be using the Arduino IDE are likely to want to do with it. First, none of the control registers can be changed while it is running; it must be briefly stopped, the register changed, and the timer restarted. In addition, the transition between stopping and starting the timer is not instant due to the synchronization process. This is fast (it looks to me to be about 2 x the synchronizer prescaler 1-8x Synchronizer-prescaler, in clock cycless. The same thing applies to reading the value of the counter - you have to request a capture by writing the SCAPTUREx bit of TCD0.CTRLE, and wait a sync-delay for it. can *also* be clocked from the unprescaled 20 MHz (or 16 MHz) internal oscillator, even if the main CPU is running more slowly. - though it also has it's own prescaler - actually, two of them - a "synchronizer" clock that can then be further prescaled for the timer itself. It supports normal PWM (what they call one-ramp mode) and dual slope mode without that much weirdness, beyond the fact that `CMPBSET` is TOP, rather than it being set by a dedicated register. But the other modes are quite clearly made for driving motors and switching power supplies. Similar to Timer1 on the ATtiny x5 and x61 series parts in the classic AVR product line,  this timer can also create programmable dead-time between cycles.

It also has a 'dither' option to allow PWM at a frequency in between frequencies possible by normal division of the clock - a 4-bit value is supplied to the TCD0.DITHER register, and this is added to a 4-bit accumulator at the end of each cycle; when this rolls over, another clock cycle is inserted in the next TCD0 cycle.

The asynchronous nature of this timer, however, comes at a great cost: It is much harder to use than the other timers. Most changes to settings require it to be disabled as noted above - and you need to wait for that operation to complete (check for the `ENABLERDY` bit in `TCD0.STATUS`). Similarly, to tell it to apply changes made to the `CMPxSET` and `CMPxCLR` registers, you must use the `TCD.CTRLE` (the "command" register) to instruct it to synchronize the registers. Similarly, to capture the current count, you need to issue a SCAPTUREx command (x is A or B - there are two capture channels) - and then wait for the corresponding bit to be set in the `TCD0.STATUS` register. In the case of turning PWM channels on and off, not only must the timer be stopped, but a timed write sequence is needed ie, `_PROTECTED_WRITE(TCD0.FAULTCTRL,value)` to write to the register that controls whether PWM is enabled; this is apparenmtly because, in the intended use-cases of motor and switching power supply control, changing this accidentally (due to a wild pointer or other software bug) could have catastrophic consequences. Writes to any register when it is not "legal" to write to it will be ignored. Thus, making use of the type D timer for even simple tasks requires careful study of the datasheet - which is a frustratingly terse chapter at key points, yet is STILL the longest chapter at 50 pages (counting only chapters that are mostly text (so the electrical/typical charachteristics section doesn't count).

### RTC - 16-bit Real Time Clock and Programmable Interrupt Timer
Information on the RTC and PIT will be added in a future update.

#### RTC/PIT errata on 0/1-series
One very important thing to be aware of on the tinyAVR 0/1-series only (and not the 3216/3217, where this has been fixed) is that there is some NASTY errata here. First, you must never have the RTC running without the PIT, nor the pit without the RTC). The Microchip errata for impacted parts does not do justice to the extent of how strange the resulting behavior is. Second, the PIT timer will be glitch every time RTC.CTRLA is written to, because the prescaler count gets reset (since you can have up to 15 bits of prescaling, this can set the timer back up to 1 second (assuming a 32.768 kHz clock)). It is also easy to inadvertently rely on this to get a deterministic delay from the PIT, which is not supposed to be possible. The intended behavior is that if you set the PIT to generate an interrupt every X seconds, it will do that, but you don't get to control where in the cycle you start from.

## Timer Prescaler Availability
Prescaler    | TCAn  | TCBn  | TCD0  | TCD0 sync | TD0 counter |
------------ | ------|-------|-------|-----------|-------------|
CLK          |  YES  |  YES  |  YES  |  YES      |  YES        |
CLK2         |  YES  |  YES  |  YES* |  YES      |  NO         |
CLK/4        |  YES  |  TCA  |  YES  |  YES      |  YES        |
CLK/8        |  YES  |  TCA  |  YES* |  YES      |  NO         |
CLK/16       |  YES  |  TCA  |  YES* |  NO       |  NO         |
CLK/32       |  NO   |  NO   |  YES  |  NO       |  YES        |
CLK/64       |  YES  |  TCA  |  YES* |  NO       |  NO         |
CLK/128      |  NO   |  NO   |  YES* |  NO       |  NO         |
CLK/256      |  YES  |  TCA  |  YES* |  NO       |  NO         |
CLK/1024     |  YES  |  TCA  |  NO   |  NO       |  NO         |

* Requires using the synchronizer prescaler as well. My understanding is that this results in sync cycles taking longer.
`TCA` indicates that for this prescaler, a TCA must also use it, and the TCB set to use that TCA's clock.

## Resolution, Frequency and Period
When working with timers, I constantly found myself calculating periods, resolution, frequency and so on for timers at the common prescaler settings. While that is great for adhoc calculations, I felt it was worth some time to make a nice looking chart that showed those figures at a glance. The numbers shown are the resolution (when using it for timing), the frequency (at maximum range), and the period (at maximum range - ie, the most time you can measure without accounting for overflows).
### [In Google Sheets](https://docs.google.com/spreadsheets/d/10Id8DYLRtlp01KA7vvslC3cHaR4S2a1TrH7u6pHXMNY/edit?usp=sharing)

## Section Two: How the core uses these timers

| Timer function | Timer used | Under conditions                                    | Examples |
|----------------|------------|-----------------------------------------------------|----------|
| PWM            | TCA0       | Pin in question is linked to TCA0                   | All      |
| PWM            | TCD0       | TCD0 is present.                                    | 1-series |
| PWM            | TCBn       | Not supported on megaTinyCore - could only get 1 pin| None     |
| tone           | TCB1, TCB0 | Prefers TCB1 if not used for time                   | All      |
| servo          | TCB0, TCB1 | Prefers TCB0 if not used for time                   | All      |
| millis         | TCD0       | Default where TCD0 present                          | 1-series |
| millis         | TCB1       | Default where TCB1 present but TCD0 is not          | 2-series |
| millis         | TCA0       | Default where neither TCD0 nor TCB1 present         | 0-series |
| millis         | TCB0       | Used only when millis timer explicitly set to TCB0  |          |

millis() and PWM can coexist on TCA0 or TCD0, but not on a TCB.

### PWM via analogWrite()
#### TCAn
The core reconfigures the type A timers in split mode, so each can generate up to 6 PWM channels simultaneously. The `LPER` and `HPER` registers are set to 254, giving a period of 255 cycles (it starts from 0), thus allowing 255 levels of dimming (though 0, which would be a 0% duty cycle, is not used via analogWrite, since `analogWrite(pin,0)` calls `digitalWrite(pin,LOW)` to turn off PWM on that pin). This is used instead of a PER=255 because `analogWrite(255)` in the world of Arduino is 100% on, and sets that via `digitalWrite()`, so if it counted to 255, the arduino API would provide no way to set the 255/256th duty cycle). Additionally, modifications would be needed to make `millis()`/`micros()` timekeeping work without drift when TCA is selected for timekeeping. Preventing drift is easy with PER = 254, but hard at PER = 255 (.

Alternate pins are not supported for TCA PWM at this time. A tools submenu option may be added in the future

`analogWrite()` checks the `PORTMUX.TCAROUTEA` register. On parts with a working TCD portmux (ie, DD-series).

When there are multiple timers available for a pin, TCD and TCA take priority. TCBs are used only as a last resort.

`analogWrite()` on tinyAVR parts does NOT interpret the PORTMUX settings. The logic required is more complex than on DxCore while resources are more limited. On the tinyAVR parts, alternate pins are not supported. They are set individually, and much more chaotically, so an algorithm to support takes more clock cycles and more flash on parts that have less of both; The `PORTMUX.TCAROUTEA` or `PORTMUX.CTRLC` register should not be written without calling `takeOverTCA0()`.


### TCD0
TCD0, by default, is configured for generating PWM (unlike TCA's, that's about all it can do usefully). TCD0 is clocked from the CLK_PER when the system is using the internal clock without prescaling. On the prescaled clocks (5 and 10 MHz) it is run it off the unprescaled oscillator (just like on the 0/1-series parts that it inherits the frequencies from), keeping the PWM frequency near the center of the target range. When an external clock is used, we run it from the internal oscillator at 8 MHz, which is right on target.

It is always used in single-ramp mode, with `CMPBCLR` (hence TOP) set to either 254, 509, or 1019 (for 255 tick, 510 tick, or 1020 tick cycles), the sync prescaler set to 1 for fastest synchronization, and the count prescaler to 32 except at 1 MHz. `CMPACLR` is set to 0xFFF (the timer maximum, 4095). The `CMPxSET` registers are controlled by `analogWrite()` which subtracts the supplied dutycycle from 255, checks the current CMPBCLR high byte to see how many places to left-shift that result by before subtracting 1 and writing to the register. The `SYNCEOC` command is sent to synchronize the compare value registers at the end of the current PWM cycle if the channel is already outputting PWM. If it isn't, we have to briefly disable the timer, turn on the pin, and then re-enable it, producing a glitch on the other channel. To mitigate this issue we treat 0 and 255 duty cycles differently for the TCD pins - they instead set duty cycle to 0% without disconnecting the pin from the timer, for the 100% duty cycle case, we invert the pin (setting CMPxSET to 0 won't produce a constant output). This eliminates the glitches when the channels are enabled or disabled.

TCD0 has two output channels - however, each of them can go to either of two pins. PA4 and PA6 use WOB, and PA4 and PC0 use WOA.
```c++
analogWrite(PIN_PA4,64);  // outputting 25% on PA4
analogWrite(PIN_PA5,128); // 25% on PA4, 50% on PA5
analogWrite(PIN_PA5,0);   // 25% on PA4, PA5 constant LOW, but *still connected to timer*
digitalWrite(PIN_PA5,LOW);// NOW PA5 totally disconnected from timer. A glitch will show up briefly on PA4.
analogWrite(PIN_PA6,192); // This is on same channel as PA4. We connect channel to PA6 too (not in place of - we do the same thing on ATTinyCore for the 167 pwm output from Timer1 on the latest versions).
                          // so now, both PA4 and PA6 will be outputting a 75% duty cycle. Turn the first pin off with digitalWrite() to explicitly turn off that pin.
```
The output channels are only available through megaTinyCore on 20 and 24 pin parts (However, see below for planned options in 2.6.x)

#### TCBn
The type B timers are never used by megaTinyCore for PWM.

### PWM Frequencies
The frequency of PWM output using the settings supplied by the core is shown in the table below. The "target" is 1 kHz, never less than 490 Hz or morethan 1.5 kHz. As can be seen below, there are several frequencies where this has proven an unachievable goal. The upper end of that range is the point at which - if PWMing the gate of a MOSFET - you have to start giving thought to the gate charge and switching losses, and may not be able to directly drive the gate of a modern power MOSFET and expect to get acceptable results (ie, MOSFET turns on and off completely in each cycle, there is minimal distortion of the duty cycle, and it spends most of it's "on" time with the low resistance quoted in the datasheet, instead of something much higher that would cause it to overheat and fail). Not to say that it **definitely** will work with a given MOSFET under those conditions (see [the PWM section of my MOSFET guide](https://github.com/SpenceKonde/ProductInfo/blob/master/MOSFETs/Guide.md#pwm) ), but the intent was to try to keep the frequency low enough that that use case was viable (nobody wants to be forced into using a gate driver), without compromising the ability of the timers to be useful for timekeeping.

The frequency of PWM output using the settings supplied by the core is shown in the table below. The "target" is 1 kHz, never less than 490 Hz or morethan 1.5 kHz. As can be seen below, there are several frequencies where this has proven an unachievable goal. The upper end of that range is the point at which - if PWMing the gate of a MOSFET - you have to start giving thought to the gate charge and switching losses, and may not be able to directly drive the gate of a modern power MOSFET and expect to get acceptable results (ie, MOSFET turns on and off completely in each cycle, there is minimal distortion of the duty cycle, and it spends most of it's "on" time with the low resistance quoted in the datasheet, instead of something much higher that would cause it to overheat and fail). Not to say that it **definitely** will work with a given MOSFET under those conditions (see [the PWM section of my MOSFET guide](https://github.com/SpenceKonde/ProductInfo/blob/master/MOSFETs/Guide.md#pwm) for calculations and a shared spreadsheet that helps calculate  ), but the intent was to try to keep the frequency low enough that that use case was viable (nobody wants to be forced into using a gate driver), without compromising the ability of the timers to be useful for timekeeping.
#### TCA0

|   CLK_PER | Prescale |   fPWM  |
|-----------|----------|---------|
|    20 MHz |       64 | 1225 Hz |
|    16 MHz |       64 |  980 Hz |
|    10 MHz |       64 |  613 Hz |
|     8 MHz |       64 |  490 Hz |
|     5 MHz |       16 | 1225 Hz |
|     4 MHz |       16 |  980 Hz |
|     1 MHz |        8 |  490 Hz |

#### TCD0 (PC0, PC1 on 1-series only)

*warning* - When using the Tuned Internal oscillator clock option, the PWM frequency will scale up or down with the CPU speed, as shown below for the highest and lowest tuned frequencies available for each. However, when an external clock source is used, the internal oscillator will be left at it's default calibration (16 or 20 MHz). This will be used for TCD PWM, which we always generate from the internal oscillator (though it can be set to an external clock source,


| FREQSEL fuse  | TCD clock source | Sync Prescale | Count prescale | TOP |    fPWM |
|---------------|------------------|---------------|----------------|-----|---------|
| 0x02 (20 MHz) | OSCHF @ 32 MHz   |             2 |             32 | 509 |  980 Hz |
| 0x02 (20 MHz) | **OSCHF @ 20 MHz** |           1 |             32 | 509 | 1225 Hz |
| 0x02 (20 MHz) | OSCHF @ 12 MHz   |             1 |             32 | 509 |  735 Hz |
| 0x01 (16 MHz) | OSCHF @ 25 MHz   |             1 |             32 | 509 | 1531 Hz |
| 0x01 (16 MHz) | **OSCHF @ 16 MHz** |           1 |             32 | 509 |  980 Hz |
| 0x01 (16 MHz) | OSCHF @ 10 MHz   |             1 |             32 | 509 |  612 Hz |

This section is incomplete and will be expanded at a later date.

### Planned new PWM options for 2.6.x versions
Starting from 2.6.x, we are planning to add an additional tools submenu, PWM pin configuration. This functionality is not yet implemented

Unlike DxCore, where the overhead of identifying the timer and channel is low and flash abounds, the calculations are longer and the resources more limited here. Thus, we cannot offer automatic PWM moving by simply setting PORTMUX like we could there, however, starting in 2.6.0, there will be a tools menu to select from several PWM layouts.

Note: This cannot be made changeable at runtime; it *must* be a tools menu option, because the number of conditionals involved would cause unacceptable code bloat.

| Option description        | TCA0 PWM        | TCD0 PWM | 14pin | 20pin | 24pin | Notes |
|---------------------------|-----------------|----------|-------|-------|-------|-------|
| Default (6xTCA)           | PB0-2,PA3-5     | None     | Yes   | No    | No    | On 14-pin parts, this is default, Both of the TCD pins already have pwm from TCA which can't be remapped, TCD is not used for PWM on those   |
| No TCD (6xTCA)            | PB0-2,PA3-5     | None     | No    | Yes   | Yes   | On 20/24-pin parts, this will save a non-negligible amount of flash, and improve performance of digitalWrite and analogWrite. No TCD PWM.  |
| Default (6xTCA, 2xTCD)    | PB0-2,PA3-5     | PC0, PC1 | No    | Yes   | Yes   | On 14-pin 1-series, Both of the TCD pins already have pwm from TCA which can't be remapped, TCD is not used for PWM on those   |
| Avoid UART (6xTCA, 2xTCD) | PB0,1,5 PA3-5   | PC0, PC1 | No    | Yes   | Yes   | On 20/24-pin parts we can avoid wasting any PWM channels on the USART pins |
| Avoid Serial (6xTCA)      | PB1-3,PA3-5     | None     | Yes   | No    | No    | Minimizes conflict between PWM and SPI/I2C/USART as much as possible on 14-pin parts. No TCD0 PWM. |
| Avoid Serial (6xTCA 2xTCD)| PB3-5,PC3-5     | PC0, PC1 | No    | No    | Yes   | Minimizes conflict between PWM and SPI/I2C/USART. TCD0 PWM on PA4, PA5. |
| Avoid UART (3xbTCA, 2xTCD)| PB0-2           | PA4, PA5 | Yes   | No    | No    | Avoid UART pins when possible. 14-pin can't fully avoid USART pins. PWM output buffering enabled. TCD0 PWM on PA4, PA5. |
| Avoid UART (3xbTCA, 2xTCD)| PB0,1,5         | PA4, PA5 | No    | Yes   | Yes   | Avoid UART pins completely. PWM output buffering enabled. TCD0 PWM on PA4, PA5. |
| Avoid UART (3xbTCA, 2xTCD)| PB0,1,5         | PC0, PC1 | No    | Yes   | Yes   | Avoid UART pins completely. PWM output buffering enabled. TCD0 PWM on PC0, PC1 (note that these are also alternate pin mapping options USART1 on 2-series). |
| Avoid UART (3xbTCA)       | PB0-2           | None     | Yes   | No    | No    | 14-pin can't fully avoid USART. PWM output buffering enabled. No TCD0 PWM - saves flash. |
| Avoid UART (3xbTCA)       | PB3-5           | None     | No    | Yes   | Yes   | 14-pin can't fully avoid USART. PWM output buffering enabled. TCD0 PWM on PA4, PA5. |
| Avoid I2C (3xbTCA, 2xTCD) | PB1-3           | PA4, PA5 | Yes   | No    | No    | 14-pin can't fully avoid I2C. You lose one of the PWM pins if you want I2C (though the 1-series - and only the 1-series - can move I2C to PA1 and PA2). PWM output buffering enabled. TCD0 PWM on PA4, PA5. |
| Avoid I2C (3xbTCA, 2xTCD) | PB3-5           | PC0, PC1 | No    | Yes   | Yes   | Avoid I2C pins. PWM output buffering enabled. TCD0 PWM on PC0, PC1. |
| Avoid I2C (3xbTCA, 2xTCD) | PB3-5           | PA4, PA5 | No    | Yes   | Yes   | Avoid I2C pins. PWM output buffering enabled. TCD0 PWM on PA4, PA5. |
| Avoid I2C (3xbTCA)        | PB1-3           | None     | Yes   | No    | No    | Avoid I2C pins. PWM output buffering enabled. No TCD0 PWM - saves flash. |
| Avoid I2C (3xbTCA)        | PB3-5           | None     | No    | Yes   | Yes   | Avoid I2C pins. PWM output buffering enabled. No TCD0 PWM - saves flash. |
| Default 8-pin (4xTCA)     | PA1,2,3,7       | None     | No    | No    | No    | 8-pin parts only, the default
| 3 pins (3xbTCA)           | PA1-3           | None     | No    | No    | No    | 8-pin parts only, trade 4th pwm pin for buffering and a bit more flash.
| 5 pins (3xbTCA, 2xTCD)    | PA1-3           | PA6, PA7 | No    | No    | No    | 8-pin 1-series parts only. 5 PWM pins. A flash hog - these parts max out at just 4k of flash. The so-called "no glitch TCD" implementation is not used in order to save flash in this case.

#### TCD PWM pins
On the 14-pin parts, when TCA is used in split mode, the only pins available for TCD PWM are PA4 and PA5 - which are already the only outputs WO4 and WO5 of TCA0 on those parts. Hence, TCD PWM is not used in that configuration as it would just waste flash without giving you more PWM pins. When TCA is not used in split mode, that opens the door to TCD-generated PWM on the 14-pin 1-series parts. Also, alternate mappings of the TCA PWM pins make TCD PWM on PA4, PA5 reasonable to use on 24-pin parts depending on what alternate functions you need from the pins not used for PWM.
If TCD PWM pins are not needed, disabling that functionality saves a bit of flash.

#### Buffering/3-pin mode? What's that?
This means the following things will be done differently in the TCA configuration:
* TCA is not run in split mode. There are only 3 output channels.
* The PERBUF CMP0BUF, CMP1BUF, and CMP2BUF can be used instead of PER, CMP0, CMP1, or CMP2, and analogWrite will do this. When the buffer registers are used, the changes are applied at the end of the current duty cycle, preventing glitches when those registers are changed. This is occasionally a problem with PWM for brightness control, where there would be a 1/255 chance analogWrite(pin,1) called when the duty cycle was previously 2, to cause the LED to be on for an entire PWM cycle. This is usually difficult to generate visible effects from, but in certain lighting situations has for reasons unclear, been problematic, resulting in a distracting flash when transitioning between very low brightness levels.
* PER and prescaler are initialized the same was as above
* However, because it is not in split mode, they can be changed to 16-bit values.
  * In 2.6.x the megaTinyCore library will be expanded with functions to support this; it is conceptually surprisingly hard to do
* Buffered mode uses slightly less flash.
  * If TCD is added as a result of changes to this menu option, that will increase the flash usage significantly.
* The number of buffered channels is abbreviated bTCA in above table.

## Millis/Micros Timekeeping
megaTinyCore allows any of the timers to be selected as the clock source for timekeeping via the standard millis timekeeping functions. ThThe timer used and system clock speed will effect the resolution of `millis()` and `micros()`, the time spent in the millis ISR, and the time it takes for `micros()`  to return a value. The `micros()` function will typically take several times it's resolution to return, and the times returned corresponds to the time `micros()` was called, regardless of how long it takes to return.

A table is presented for each type of timer comparing the percentage of CPU time spent in the ISR, the resolution of the timekeeping functions, and the execution time of micros.

### Why this longass section matters
Both `micros()` and `millis()` take the timer value at almost the moment they are called, then do the math on it, so if you wait for something, call `micros()` wait until something else happens and call `micros()` again, you can take the difference and be confident that it's not grossly inaccurate. But since `micros()` takes several microseconds to return, you can't code as if it returns instantly. *(It has to multiply a 16 bit integer by a 32 bit one, perform division-by-successive-bitshift-and-add/subtraction on a 16-bit opperand and then add the two together, 8-bits at a time? Surely 120 clocks isn't too long to wait for a 32 bit multiply (`timer_overflow_count*1000`) and a 5-9 term approximated division (`ticks = (ticks >> n) - (ticks >> n+a) ... (ticks >> n+n+h)` on a 20 MHz 8-bit CPU?)*
Consider:
```c
while (!digitalReadFast(pina));  // this is an extremely tight loop - sbrs rjmp .-4 - 3 clock cycles, so 0-2 clock cycles, + 0-2 clock cycles sync delay/
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
//timetwo = micros(); // starttime - timetwo = ~6
while (digitalReadFast(pina));
endtime = micros(); //
```
Imagine if at t = 100 and around`*` 20 MHz, the pin goes high. Within 0.10 the pin change is sync'ed and then you wait 0.05-0.15us for the loop to catch it, within 0.4us it has recorded the micros timer value. So you enter the second loop at t = 106.25, and starttime = t at t = 100.45-100.55, returning 100.45-100.55 (that is, the value of micros at a true time of 100.45-100.55 us). If the pin is already low, we would call micros at that time, and expect that by 106.35 we would get the value of micros at true time 106.9, record 106 or 107, subtract the two and get 6 or 7 us for the length of the pulse, otherwise, suppose it changes at 200 us, we call micros, which has it's value around 200.4 and at 206.25 returns likely 200 or 201 (with 200 more likely if 100 was measured the first time and vice versa), and would conclude that the pulse was 100us long, at worst 99 or 101 us.

But if you check micros in a loop, it is not so harmless
```c
while (!digitalReadFast(pina));  // this is an extremely tight loop - sbrs rjmp .-4 - 3 clock cycles, so 0-2 clock cycles, + 0-2 clock cycles sync delay/
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
digitalWriteFast(pinb, HIGH); //
while (micros() - starttime < 100); // this loop only samples micros every ~6us,
digitalWriteFast(pinb, LOW); // so now our pulse is too short
```
We still enter at 106.30, but now we are waiting around 6us between calls to micros(), so with with orders to stop 100 us from the time it was about 6.5us ago, and we notice that it's time at t = 200-205, so our pulse was from 106 or 107 to 200 to 205, Hence, it's length would *actually* be 94-101 us.

This skews the other direction:
```c
while (!digitalReadFast(pina));  // this is an extremely tight loop - sbrs rjmp .-4 - 3 clock cycles, so 0-2 clock cycles, + 0-2 clock cycles sync delay/
digitalWriteFast(pinb, HIGH); // this time, write the pin before recording starttime.
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
while (micros() - starttime < 100); // this loop only samples micros every ~6us,
digitalWriteFast(pinb, LOW); // so now our pulse is too long
```
We enter as above at 106.35us, but this time we started the pulse at 100.3us, and hold about the same value (100, maybe 101) in starttime, but we wouldn't notice until T = 200-205, and our pulse would be 99-106 us long.
```c
while (!digitalReadFast(pina));  // this is an extremely tight loop - sbrs rjmp .-4 - 3 clock cycles, so 0-2 clock cycles, + 0-2 clock cycles sync delay/
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
digitalWriteFast(pinb, HIGH); //
while (micros() - starttime < (100 - (6/2))); // this loop only samples micros every ~6us,
digitalWriteFast(pinb, LOW); // so now our pulse on average is 100usm but might be as low as 96 or as high and 103
```

So - the message is you shouldn't busywait on micros(). micros is for time since something happened, it is not meant to be checked continually to see if a timeout is passed.  in the previous example, if linked by the event system to the pin, a TCB could be counting as soon as the new pin value was synchronized, giving a consistent length of 100us starting a fraction of a us after the pulse was seen. You can generate pulses up to (131070 / Clocks-per-us), about 6.5 seconds at 20 MHz, accurate to within 2 system clocks in that way. If you also wanted to delay the execution of later code, which you likely do if your first instinct was something like what is shown above, these can be done concurrently....

```c
setupTCB(); // call your function to configure TCB in single shot mode, outputting a pulse 2000 clocks long triggered by a rising edge.
setupEVSYS(); // anc connect pina to an event channel, and set the TCB to use it.
while (!digitalReadFast(pina));  // we'll see the pin at the same time as the TCB,
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
//timetwo = micros(); // starttime - timetwo = ~6
while (digitalReadFast(pina)); // and exit the loop at t=200 us basedon the ideal clock. Up until now, we've assumed the clock was at exactly 20 MHz, but if it's off 1%, the pulse length would be off in the same direction. So this will end after the original pulse ended, with our pulse on for another microsecond (if our clock is 1% slow) or have been off for a microsecond (if our clock is 1% fast).
endtime = micros(); //
```

```c
setupTCB(); // call your function to configure TCB in single shot mode, outputting a pulse 2000 clocks long triggered by a rising edge.
setupEVSYS(); // anc connect pina to an event channel, and set the TCB to use it.
while (!digitalReadFast(pina));  // we'll see the pin at the same time as the TCB,
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
//timetwo = micros(); // starttime - timetwo = ~6
while (digitalReadFast(pinb)); //wait for our own pulse to end;
endtime = micros(); //
```
You're probably hoping to time the input pulse and make sure the output pulse has ended before continuing:
```c
setupTCB(); // call your function to configure TCB in single shot mode, outputting a pulse 2000 clocks long triggered by a rising edge. If it is desired to not retrigger, enable the TCB interrupt and use it to disable the TCB and/or event channel.
setupEVSYS(); // anc connect pina to an event channel, and set the TCB to use it.
while (!digitalReadFast(pina));  // we'll see the pin at the same time as the TCB,
starttime = micros(); // this is 6 us @ 20 MHz. starttime returns the time measured in the first few clocks after calling it.
//timetwo = micros(); // starttime - timetwo = ~6
while (digitalReadFast(pina)); //wait for our input pulse to end;
endtime = micros(); // highly accurate as long as pulse len longer than micros()
while (digitalReadFast(pinb)); // make sure our pulse is over - can be omitted if we're not emitting a pulse plausibly longer by more than the micros execution time (in this example, if we were outputting a pulse that might be longer than 106 us, counting timer drift), though it only costs 4 bytes and 2 clocks to test for it.
```
`*` internal oscillator is usually within a percent at room temperature, and within half a percent is common. We're assuming that at the moment millis started, an ideal stopwatch was started, and when it indicated 100us had gone by, drove the pin low effectively instantaneously.

A table is presented for each type of timer comparing the percentage of CPU time spent in the ISR, the resolution of the timekeeping functions, and the execution time of micros. Typically `micros()`  can have one of three execution times, the shortest one being overwhelmingly more common, and the differences between them are small.


### TCAn for millis timekeeping
When TCA0 is used as the millis timekeeping source, it is set to run at the system clock prescaled by 8 when system clock is 1MHz, 16 when system clock is <=5 MHz, and 64 for faster clock speeds, with a period of 255 (as with PWM). This provides a `millis()`  resolution of 1-2ms, and is effecively not higher than 1ms between 16 and 30 MHz, while `micros()` resolution remains at 4 us or less. At 32 MHz or higher, to continue generating PWM output within the target range, we are forced to switch to a larger prescaler (by a factor of 4), so the resolution figures fall by a similar amount, and the ISR is called that much less often.

In 2.6.2, the same techniques used for TCB were applied to TCA, improving the ISR performance by an average of 29%, reducing time spent in the isr and increasing the time spent executing your code

#### TCA timekeeping resolution
|   CLK_PER | millis() | micros() | % in ISR | micros() time |
|-----------|----------|----------|----------|---------------|
|    32 MHz |  2.04 ms |   8.0 us |   0.19 % |          4 us |
|    30 MHz |  0.54 ms |   2.1 us |   0.51 % |   aprx   4 us |
|    28 MHz |  0.58 ms |   2.3 us |   0.51 % |          4 us |
|    25 MHz |  0.65 ms |   2.6 us |   0.51 % |   aprx   4 us |
|    24 MHz |  0.68 ms |   2.7 us |   0.51 % |          5 us |
|    20 MHz |  0.82 ms |   3.2 us |   0.51 % |          7 us |
|    16 MHz |  1.02 ms |   4.0 us |   0.51 % |          9 us |
| !  14 MHz |  1.14 ms |   4.6 us |   0.51 % |   aprx  10 us |
|    12 MHz |  1.36 ms |   5.3 us |   0.51 % |         10 us |
|    10 MHz |  1.63 ms |   6.4 us |   0.51 % |         14 us |
|     8 MHz |  2.04 ms |   8.0 us |   0.51 % |         17 us |
| !   7 MHz |  0.58 ms |   2.3 us |   2.13 % |   aprx  18 us |
| !   6 MHz |  0.68 ms |   2.7 us |   2.13 % |   aprx  19 us |
|     5 MHz |  0.82 ms |   3.2 us |   2.13 % |         27 us |
|     4 MHz |  1.02 ms |   4.0 us |   2.13 % |         33 us |
| !   3 MHz |  0.68 ms |   2.7 us |   3.55 % |   aprx  45 us |
| !   2 MHz |  1.02 ms |   4.0 us |   3.55 % |   aprx  60 us |
|     1 MHz |  2.04 ms |   8.0 us |   3.55 % |        112 us |

`!` - Theoretical, these speeds are not supported and have not been tested

In contrast to the type B timer where prescaler is held constant while the period changes, here period (in ticks) is constant but the prescaler is not. Hence each prescaler option is associated with a fixed % of time spent in the ISR (and yes, for reasons I don't understand, the generated ISR code is slightly faster for /64 prescaling compared to /256, /16, and /8 (which are equal to each other).

The micros execution time does not depend strongly on F_CPU, running from 112-145 clock cycles.

Except when the resolution is way down near the minimum, the device spends more time in the ISR on these parts. Notice that at these points that - barely - favor TCAn, the interrupt they're being compared to is firing twice as frequently! TCD0's interrupt is slower than TCA's, but it is the timer least likely to be repupurposed.


#### TCBn for millis timekeeping
When a TCB is used for `millis()` timekeeping, it is set to run at the system clock prescaled by 2 (except at 1 or 2 MHz system clock) and tick over every millisecond. This makes the millis ISR very fast, and provides 1ms resolution at all but the slowest speeds for millis. The `micros()` function also has 1 us or almost-1 us resolution at all clock speeds (though there are small deterministic distortions due to the performance shortcuts used for the microsecond calculations (see appendix below). The only reason these excellent timers are not used by default except on parts with two TCBs and no TCD (that is, on the 2-series) is that too many other useful things need a TCB.

### TCBn for millis timekeeping
When TCB2 (or other type B timer) is used for `millis()` timekeeping, it is set to run at the system clock prescaled by 2 (1 at 1 MHz system clock) and tick over every millisecond. This makes the millis ISR very fast, and provides 1ms resolution at all speeds for millis. The `micros()` function also has 1 us or almost-1 us resolution at all clock speeds (though there are small deterministic distortions due to the performance shortcuts used for the microsecond calculations. The type B timer is an ideal timer for millis, however, they';'re also good for a lot of other things too. It is anticipated that as libraries for IR, 433MHz OOK'ed remote control, and similar add support for the modern AVR parts, that these timers will see even more use.
ISR execution time was decreased by 25% from 2.6.1 of megaTinyCore through reimplementation of the ISR is assembly, meaninmg less time spent incrementing millis and more on your code.

|Note | CLK_PER | millis() | micros() | % in ISR | micros() time | Terms used             |
|-----|---------|----------|----------|----------|---------------|------------------------|
|  †  |  32 MHz |     1 ms |     1 us |   0.20 % |          3 us | 1                      |
|     |  30 MHz |     1 ms |  1.07 us |   0.22 % |        < 6 us | 5 (3,4,7,8)            |
| !   |  28 MHz |     1 ms |  1.14 us |   0.23 % |       <= 6 us | 5/7 (2,3,5,6 ~8,9~ )   |
| !   |  27 MHz |     1 ms |  1.18 us |   0.24 % |       <= 6 us | 4 (2,4,9)              |
|     |  25 MHz |     1 ms |  1.28 us |   0.26 % |       <= 6 us | 3/5 ( ~1,~ 2, ~4,~ 5)  |
|   * |  24 MHz |     1 ms |  1.33 us |   0.27 % |          5 us | 9 (0-7, 9)             |
|   * |  20 MHz |     1 ms |     1 us |   0.33 % |          6 us | 6 (2,4,6,8,10)         |
|  †  |  16 MHz |     1 ms |     1 us |   0.40 % |          6 us | 1                      |
| !   |  14 MHz |     1 ms |  1.14 us |   0.47 % | approx. 12 us | 5/7 (2,3,5,6 ~8,9~ )   |
|   * |  12 MHz |     1 ms |  1.33 us |   0.54 % |         10 us | 9 (0-7, 9)             |
|     |  10 MHz |     1 ms |     1 us |   0.65 % |         10 us | 6 (2,4,6,8,10)         |
|  †  |   8 MHz |     1 ms |     1 us |   0.80 % |         11 us | 1                      |
| !   |   7 MHz |     1 ms |  1.14 us |   0.94 % | approx. 25 us | 5/7 (2,3,5,6 ~8,9~ )   |
| ! * |   6 MHz |     1 ms |  1.33 us |   1.08 % |       > 20 us | 9 (0-7, 9)             |
|   * |   5 MHz |     1 ms |     1 us |   1.30 % |         23 us | 6 (2,4,6,8,10)         |
|  †  |   4 MHz |     1 ms |     1 us |   1.18 % |         21 us | 1                      |
| ! * |   3 MHz |     1 ms |  1.33 us |   1.61 % |       > 40 us | 9 (0-7, 9)             |
| !†  |   2 MHz |     2 ms |     1 us |   1.18 % |         39 us | 1                      |
|  †  |   1 MHz |     2 ms |     1 us |   2.36 % |         78 us | 1                      |

`!` - Theoretical, these speeds are not supported and have not been tested. % time in ISR and micros return times, where given are theoretically calculated, not experimentally verified.
`†` - Naturally ideal - no ersatz division is needed for powers of 2
`*` - indicates that the shift & sum ersatz-division is done in hand-optimized assembly because I just couldn't stand how stupid and stubborn the compiler was, and by the time I was done analyzing it, implementing it was mostly done. These use every term that would improve the accuracy of the approximation, shown in the final column. The algorithm divides the timer ticks (which is between zero and F_CPU/2000) by a power if 2 (by right shifting) to get term 0; a number of terms (generated by further rightshifting) are then added and subtracted to get the time since the last millis was counted, and that total is added to 1000x the millisecond count. The ideal number of terms varies from 1 (for powers of 2 - only term 0) to 9 (for the terrible twelves (12, 24, 48, as well as 3 and 6). In optimized cases, that's how many are used. Otherwise, some may be dropped, or different combinations used (adding 1 term, then subtracting the next, would seem equivalent to just adding the next - but because of integer math, it's not, and in fact can lead to several us of backwards timetravel, so if you'd have to do the same operation twice in a row, you get more accurate numbers if you avoid that by adding another term). In optimized cases, as well as the powers of 2, the results are as good as possible in light of the specified resolution - where resolution is coarser than 1us, (1/resolution) of possible values for the three lowest digits (in normal base 10 number systems) is skipped and never shows up. These skipped values are distributed evenly between 1 and 999. For example, for 12/24/48, that means 250 values never show up. Of the remaining 750, 500 will show for only 1 "actual" microsecond value, and 250 for 2 consecutive "actual" microsecond values. See the table at the end of this document for the list of skipped least-significant-digit combinations. None of the optimized options will never go back in time, even by a single microsecond (as long as the chance of backward time travel is limited to less than the time it takes micros to execute, it can't break `delay()`, cause timeouts to instantly expire or cause other catastrophic consequences) nor are more than 2 values in a row ever skipped. Those criteria are not guaranteed to be met for other speeds, though we do guarantee that negative time travel will never occur that causes two consecutive calls to micros to return a smaller value for the second call which is what triggers misbehavior of timing code.

The terms used, and - where different - the number that would be ideal, is listed above

Resolution is always exactly 1ms for millis with TCB millis except for 1 and 2 MHz speeds, where the millis resolution is 2ms, though micros resolution is still 1us, and whereas TCAn `micros()` is limited by the resolution of the timer, here it's instead limited only by the fact that the calculations drop the least significant bits first; this results in a value that may be as low as 750, yet is being mapped to 0-999, for 1.33 us resolution in the worst cases. The timer count and the running tally of overflows could get us microseconds limited only by F_CPU/2
The percentage of time spent in the ISR varies in inverse proportion to the clock speed - the ISR simply increments a counter and clears its flags. 65 clocks from interrupt bit set to interrupted code resuming.

The time that `micros()` takes to return a value varies significantly with F_CPU; where not exact, value is estimated. Specifically, powers of 2 are highly favorable, and almost all the calculations drop out of the 1 and 2 MHz cases (the are similar mathematically - at 1 MHz we don't prescale and count to 1999, at 2 we do prescale and count to 1999, but then we need only add the current timer count to 1000x the number of milliseconds. micros takes between 78 and 160 clocks to run. Everything else has to use some version of a shift-and-sum ersatz division algorithm, since we want to approximate division using `>>`, `+` and `-`. Each factor of 2 increase in clock speed results in 5 extra clocks being added to micros in most cases (bitshifts, while faster than division, are still slow when you need multiples of them on larger types, especially in compiler-generated code that doesn't reuse intermediate values.)

The terms used, and - where different - the number that would be ideal, is listed above
### TCD0 for millis timekeeping
This will be documented in a future release.
## Tone
The `tone()` function included with DxCore uses one Type B timer. It defaults to using TCB0; do not use that for millis timekeeping if using `tone()`. Tone is not compatible with any sketch that needs to take over TCB0. If possible, use a different timer for your other needs. When used with Tone, it will use CLK_PER or CLK_PER/2 as it's clock source - the TCA clock will never be used, so it does not care if you change the TCA0 prescaler (unlike the official megaAVR core). It does not support use of the hardware output compare to generate tones like tone on some parts does. (Note that if TCB0 is used for millis/micros timing - which is not a recommended configuration, we usually use the higher numbered TCB -  `tone()` will instead use TCB1).

**Recent Fixes** - Prior to the release of 1.3.3, there were a wide variety of bugs impacting the `tone()` function, particularly where the third argument, duration, was used; it could leave the pin `HIGH` after it finished, invalid pins and frequencies were handled with obvously unintended behavior, and integer math overflows could produce weird results with larger values of duration at high frequencies, and three-argumwent `tone()` ddn't work at all above 32767Hz due to an integer overflow bug (even though maximum supported frequency is 65535 Hz).

**Long tones specifying duration** The three argument `tone()` counts the toggles of the pin for the duration. This means some very large numbers can cause problems. Cases where this is an issue are hereafter known as "Long tones". This core supports long tones through the three argument tone. A long tone is any tone where `4.294 billion < (frequency * duration) < 2.149 trillion` - by rearranging some division, as long as 16k or more flash is present on the part we are compiling for (it uses more memory). Anything longer is a "very long" tone and is never supported via the three argument `tone()`. Your earsplitting high-pitched whine with a duration of more than a few hours (which is far outside the regime that tone is intended for) must be implemented with a 2-argument tone() and millis. If it needs to be implemented at all. Even normal "long tones" require a high frequency tone requested for minutes, which is an extreme use case.

Tone works the same was as the normal `tone()` function on official Arduino boards. Unlike the official megaAVR board package's tone function, it can be used to generate arbitrarily low frequency tones (as low as 1 Hz). If the period between required toggling's of the pin is greater than the maximum timer period possible, it will calculate how many cycles it has to wait through between switching the pins in order to achieve the desired frequency.

It can only generate a tone on one pin at a time.

All tone generation is done via interrupts. The hardware output compare functionality is not used for generating tones because in PWM mode, the type B timers kind-of suck. Hardware-tone would require using a type A timer, and few are willing to sacrifice 6 pwm channels for tone - though the library would be simple.

## Servo Library
The Servo library included with this core uses one Type B timer. It defaults to using TCB1 if available, unless that timer is selected for millis timekeeping. Otherwise, it will use TCB0. The Servo library is not compatible with any sketch that needs to take over these timers - if possible, use a different timer for your other needs. Servo and `tone()` can only be used together on when neither of those is used for millis timekeeping.

Regardless of which type B timer it uses, Servo configures that timer in Periodic Interrupt mode (`CNTMODE` = 0) mode with CLK_PER/2 or CLK_PER as the clock source, so there is no dependence on the TCA prescaler. The timer's interrupt vector is used, and it's period is constantly adjusted as needed to generate the requested pulse lengths. In 1.1.9 and later, CLK_PER is used if the system clock is below 10MHz to generate smoother output and improve performance at low clock speeds.

The above also applies to the Servo_megaTinyCore library; it is an exact copy except for the name. If you have installed a version of Servo via Library Manager or by manually placing it in your sketchbook/libraries folder, the IDE will use that in preference to the one supplied with this core. Unfortunately, that version is not compatible with the Dx-series parts. Include Servo_DxCore.h instead in this case. No changes to your code are needed other than the name of the library you include.


## Additional functions for advanced timer control
  We provide a few functions that allow the user to tell the core functions that they wish to take full control over a timer, and that the core should not try to use it for PWM. These are not part of a library because of the required integration with the core to control `analogWrite()` and digitalWrite behavior.

### takeOverTCA0()
  After this is called, `analogWrite()` will no longer control PWM on any pins attached to timer TCA0 (though it will attempt to use other timers that the pin may be controllable with to, if any), nor will `digitalWrite()` turn it off. TCA0 will be disabled and returned to it's power on reset state. All TCBs that are used for PWM on parts with only TCA0 use that as their prescaled clock source buy default. These will not function until TCA1 is re-enabled or they are set to use a different clock source. Available only on parts with TCA1 where a different timer is used for millis timekeeping.

### takeOverTCA1()
  After this is called, `analogWrite()` will no longer control PWM on any pins attached to timer TCA1 (though it will attempt to use other timers that the pin may be controllable with to, if any), nor will `digitalWrite()` turn it off. TCA1 will be disabled and returned to it's power on reset state. All TCBs that are used for PWM on parts with TCA1 use that as their prescaled clock source buy default. These will not function until TCA1 is re-enabled or they are set to use a different clock source. Available only on parts with TCA1, and only when a different timer is used for millis timekeeping.

### takeOverTCD0()
  After this is called, `analogWrite()` will no longer control PWM on any pins attached to timer TCD0 (though it will attempt to use other timers, if any), nor will `digitalWrite()` turn it off. There is no way to reset type D timers like Type A ones. Instead, if you are doing this at the start of your sketch, override init_TCD0. If TCD is ever supported as millis timing source, this will not be available.

### resumeTCA0()
  This can be called after takeOverTimerTCA0(). It resets TCA0 and sets it up the way the core normally does and re-enables TCA0 PWM via analogWrite.

### resumeTCA1()
  This can be called after takeOverTimerTCA1(). It resets TCA1 and sets it up the way the core normally does and re-enables TCA1 PWM via analogWrite.

## Appendix I: Names of timers
Whenever a function supplied by the core returns a representation of a timer, these constants will be used

| Timer Name   | Value | Peripheral |          Used for |
|--------------|-------|------------|-------------------|
| NOT_ON_TIMER |  0x00 |            |                   |
| TIMERA0      |  0x10 |      TCA0  | millis and/or PWM |
| TIMERB0      |  0x20 |      TCB0  | millis            |
| TIMERB1      |  0x21 |      TCB1  | millis            |
| TIMERD0      |  0x40 |      TCD0  | millis and/or PWM |
| TIMERRTC     |  0x90 |       RTC  | millis            |
| TIMERRTC_XTAL|  0x91 |RTC,32kXtal | millis            |
| TIMERRTC_EXT |  0x92 |RTC,extclk  | millis            |
| TIMERPIT     |  0x98 |       PIT  | Reserved for future |
| DACOUT **    |  0x80 |      DAC0  |       analogWrite |
0/NOT_ON_TIMER will be returned if the specified pin has no timer.
`*` Currently, the 3 low bits are don't-care bits. At some point in the future we may pass around the compare channel in the 3 lsb when using for PWM. No other timer will ever be numbered 0x10-0x17, nor 0x08-0x0F. 0x18-0x1F is reserved for hypothetical future parts with a third TCA. Hence to test for TCA type: `(timerType = MILLIS_TIMER & 0xF8; if (timerType==TIMERA0) { ... })`
`**` A hypothetical part with multiple DACs with output buffers will have extend up into 0x8x.

All other unforeseen timer-like peripherals will be assigned to values above 0x9F
## Appendix II: TCB Micros Artifacts
3, 6, 12, 24, and 48 MHz, with the new optimized micros code, running from a TCB, is known to skip these (and only these) 250 values when determining the least significant thousand micros. That is, if you repeatedly calculate `micros % 1000`, none of these will show up until it has been running long enough for micros to overflow.

```text
2, 7, 10, 13, 18, 23, 28, 31,  34,  39,  42,  45,  50,  53,  56,  61,  66,  71,  74,  77,  82,  87,  92,  95,  98, 103, 108, 113, 116, 119, 124, 127, 130,
135, 138, 141, 146, 151, 156, 159, 162, 167, 170, 173, 178, 181, 184, 189, 194, 199, 202, 205, 210, 213, 216, 221, 224, 227, 232, 237, 242, 245, 248, 253,
258, 263, 266, 269, 274, 279, 284, 287, 290, 295, 298, 301, 306, 309, 312, 317, 322, 327, 330, 333, 338, 341, 344, 349, 352, 355, 360, 365, 370, 373, 376,
381, 384, 387, 392, 395, 398, 403, 408, 413, 416, 419, 424, 429, 434, 437, 440, 445, 450, 455, 458, 461, 466, 469, 472, 477, 480, 483, 488, 493, 498, 501,
504, 509, 512, 515, 520, 523, 526, 531, 536, 541, 544, 547, 552, 555, 558, 563, 566, 569, 574, 579, 584, 587, 590, 595, 600, 605, 608, 611, 616, 621, 626,
629, 632, 637, 640, 643, 648, 651, 654, 659, 664, 669, 672, 675, 680, 685, 690, 693, 696, 701, 706, 711, 714, 717, 722, 725, 728, 733, 736, 739, 744, 749,
754, 757, 760, 765, 770, 775, 778, 781, 786, 791, 796, 799, 802, 807, 810, 813, 818, 821, 824, 829, 834, 839, 842, 845, 850, 853, 856, 861, 864, 867, 872,
877, 882, 885, 888, 893, 896, 899, 904, 907, 910, 915, 920, 925, 928, 931, 936, 941, 946, 949, 952, 957, 962, 967, 970, 973, 978, 981, 984, 989, 992, 995
```

Those 250 "missing" microseconds manifest as repeats. It is difficult to intuitively understand why these values are the values they are; these were calculated using a spreadsheet to simulate the results for every timer count value:
```text
0, 4, 8, 12, 16, 20, 24,  27,  32,  36,  40,  44,  48,  52,  57,  60,  64,  68,  72,  76,  80,  84,  88,  91,  96, 100, 104, 107, 111, 115, 120, 123, 128,
132, 136, 140, 144, 148, 152, 155, 160, 164, 168, 172, 176, 180, 185, 188, 192, 196, 200, 204, 208, 212, 217, 220, 225, 229, 233, 236, 240, 244, 249, 252,
256, 260, 264, 268, 272, 276, 280, 283, 288, 292, 296, 300, 304, 308, 313, 316, 320, 324, 328, 332, 336, 340, 345, 348, 353, 357, 361, 364, 368, 372, 377,
380, 385, 389, 393, 397, 401, 405, 409, 412, 417, 421, 425, 428, 432, 436, 441, 444, 448, 452, 456, 460, 464, 468, 473, 476, 481, 485, 489, 492, 496, 500,
505, 508, 513, 517, 521, 525, 529, 533, 537, 540, 545, 549, 553, 557, 561, 565, 570, 573, 577, 581, 585, 589, 593, 597, 601, 604, 609, 613, 617, 620, 624,
628, 633, 636, 641, 645, 649, 653, 657, 661, 665, 668, 673, 677, 681, 684, 688, 692, 697, 700, 704, 708, 712, 716, 720, 724, 729, 732, 737, 741, 745, 748,
752, 756, 761, 764, 768, 772, 776, 780, 784, 788, 792, 795, 800, 804, 808, 812, 816, 820, 825, 828, 832, 836, 840, 844, 848, 852, 857, 860, 865, 869, 873,
876, 880, 884, 889, 892, 897, 901, 905, 909, 913, 917, 921, 924, 929, 933, 937, 940, 944, 948, 953, 956, 960, 964, 968, 972, 976, 980, 985, 988, 993, 997,
```
Similar lists can be generated for any other clock speed where resolution is coarser than 1us and a TCB is used.

The number of values will, for ideal values, be `(1000) * (1 - (1 / resolution))` or something very close to that. Non-ideal series of terms or particularly adverse clock speeds may have more than that, as well as more cases of consecutive times that get the same micros value. If the terms were chosen poorly, it is possible for a time t give micros M where M(t-1) > M(t), violating the assumptions of the rest of the core and breaking EVERYTHING. 1.3.7 is believed to be free of them for all supported clock speeds. Likewise, it is possible for there to be larger skips or discontinuities, where M(t) - M(t-1) > 2, in contrast to skips listed above (where M(t) - M(t-1) = 2 ) or consecutive times return a repeated value (ie, M(t) = M(t-1))

Timer options which have resolution of 1us (internally, it is lower) may have repeats or skips if fewer than the optimal number of terms were used (as is the case with non optimized options) or where the clock speed is particularky hard to work with, like the 28 MHz one.
