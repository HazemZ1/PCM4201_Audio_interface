Progress: #Early
Related Notes: [[Convolutional Reverb]] [[Power Amplifier Design]] [[Audio Electronics]] [[Convolutional Reverb]] [[Hobby Projects]]

# Project Aims
This project aims to design an analogue front-end board to attach directly to the Raspberry Pi-5 GPIO pins. This board will act as an ADC interface to turn (mainly) guitar analogue signals into digital signals (positive and negative) which can then be clocked out by the Raspberry Pi over SPI/I2C/UART. The purpose of this board is to allow the use of the Raspberry PI as a reconfigurable digital effect pedal. The Pi can then use the ADC to read the analogue signal, then a DAC to play it back out to a pre-amplifier circuit.

# Basic Premise
This board is very simple. You have an analogue signal Input, and an analogue output. You want your processor to be digital and reprogrammable, so what do you do? Well, there are these nifty things called Analogue-to-Digital Converters (ADCs) and Digital-to-Analogue Converters (DACs). By using these two devices, we can convert an analogue input signal, say from a guitar, to a digital signal that can be processed on our digital signal processor. We also want to be able to play the processed signal back out, so we can use a DAC to convert the signal back to analogue that can then be driven through an amplifier or another pedal. 

Along with these two core circuits, we will use an input buffer/filter circuit, and an output pre-amplifier circuit. We may also either use a negative voltage rail or a differential amplifier to either create positive/negative ADC readings or subtract an applied offset to the signal (might be able to just use a decoupling capacitor)
# System Components
![[System Diagram.png]]
This is the basic system diagram for the PCB that we will use. We have our signal entering the PCB through a 1/4 inch guitar jack. The signal then goes through a Input Pre-Amp/Filter which will act to amplify the signal to improve the dynamic range. We will then pass the signal to an Anti-Aliasing filter which will ensure that the highest frequency present in the signal is not higher than half the sampling rate of the ADC. 

The signal is then passed to the ADC, where the analogue-to-digital conversion occurs. We can then clock this data out of the ADC into the Raspberry Pi, where it is processed to whatever our heart desires. We then pass this signal to a DAC, which will convert it back to analogue. We then need to filter this signal to remove any high frequency switching noise that may be present on the signal from the DAC, Raspberry Pi, or ADC. We then pass it through another Pre-Amplifier circuit to ensure high enough current drive and low output impedance. This will allow the circuit to drive an amplifier directly, or be passed to another pedal.
# Design Steps Suggestions
Now that we have decided roughly how the system will work, we need to decide what to work on. With all analogue electronics, one of the most important aspects is the power supply. When designing a power supply for analogue electronics, you want to ensure as low a noise as possible. This is because everything in your analogue circuit will fluctuate along with your power supply (There is power supply rejection ratio on most amplifiers so the supply doesn't have to be perfect, but it is best practice to not have a power supply that oscillates!), so it is better to minimise noise on the power supply.
## Power Supply Design

So how are we powering the system? Since this will essentially be a Raspberry Pi hat, we want to power it directly off the raspberry Pi. We can use the 5V rail of the Pi to power the board.

Because we are running off the 5V supply of the pi, we can't really use a buck or boost converter unless we want more headroom on the input/output rails. Also, unless we run off of a 4.5V supply, we cannot really use a linear regulator because you need a finite voltage drop for a linear regulator to work. Another possible filter circuit is the capacitance multiplier, but this also incurs a voltage drop so we wont be going for this.

So, let's result to using some form of LC filtering the supply rails. LC filtering will be used here over RC filtering because we want no voltage drop across the supply rail (otherwise we'd just use a linear regulator!). So, what are our LC filter topology options?

### LC Filter
There's quite a few topologies for LC filters. You can have standard LC lowpass and highpass circuits, T filters, Pi filters, and LC filters with integrated snubbers.

At its core, an LC filter is a filter that ideally produces no power loss as it is made of purely reactive elements. These elements only store and exchange power, and do not dissipate it.  In reality these circuits will dissipate some power as heat as they will have some series equivalent resistance in the inductors and capacitors.

So, which filter do we pick? We could go down the simple route and use a standard passive LC filter. The danger of this is that it could introduce a resonant peak (a pole) where we get some peakiness in the filter, causing an amplification of an unwanted signal. You can counteract this by using a snubber (resistor in series with the capacitor) which will act to dampen out the resonance. This snubber is typically small and is only 2-3 ohms. Lets keep this option in mind for now.

T-type filters form a series impedance and are typically used in Diplexers (where low frequencies go to one load and high frequencies to another).  This is because you don't want the T-type high pass to block any of the signal that reaches the low frequency load, so you tend to use the LPF and HPF in parallel with two different loads [1](https://www.youtube.com/watch?v=KECCpj1Qpn0)

Pi filters allow you to prevent any high frequencies noise from coupling from your supply to an IC. This is typically done by using shunt capacitors in a LPF arrangement to shunt the noise to ground. It is a symmetrical filter which also prevent noise from the IC coupling back to the supply or another IC. 

### Pi Filter or T filter?
The main difference between pi and T filters is how they appear to the source. A pi filter will appear as a low-impedance to the source, which will allow to have high attenuation at high frequencies from the source. T filters on the other hand will have a high impedance at the source, which is a low attenuation as the noise signal can more easily couple through. The main reason for this is due to the shunting component at the start of the filter in the pi filter. In a low-pass pi filter, there is a shunting capacitor to ground right at the source, which is a low impedance at high frequencies. In contrast with the T filter where the first element is a series inductor, which has a high impedance at high frequencies, resulting in high coupling. These things are more important to consider in RF circuits, but still worth knowing.

Since we are designing for a power supply to remove high frequency noise, we want to use a pi filter. Lets look at how to do this.

## Pi Filter Design
We will be using the [Altium Guide](https://resources.altium.com/p/pi-filter-circuit-design-formulas-and-calculator) to help us design our Pi filter for the power supply. 

One important thing to consider which I read in another Altium guide is that we will need our power supply to be able to respond to our circuit, so we cant just cut the power rail filter to 1Hz. If you want to read more about this look at this [power filter article](https://resources.altium.com/p/how-filter-noisy-power-rails) and this [power filter video](https://www.youtube.com/watch?v=VaHTEuqKB4Y) by Zach Peterson.

An important decision we need to make is: do we want a Chebyshev or a Butterworth filter? Well, since we are designing to eliminate high frequency noise, you'd think we want the strong roll-off of a Chebyshev. However, we don't want the passband ripple as we want as smooth a power supply as possible for our analogue electronics. We can always use a snubber to get rid of the resonance in the Butterworth filter!

All pi filters have a characteristic impedance value and a cutoff frequency value. When the shunt elements are equal to each other, the input and output impedances of the circuit match the selected impedance (doubly terminated). The frequency value is a resonant frequency, but the presence of a high-Q resonance in the transfer function depends on the value of the load, as we will see later. The design approach involves selecting a frequency (f) and impedance value (Z) and using these to determine the L and C values.

The above paragraph is quoted directly from the  [Altium Guide](https://resources.altium.com/p/pi-filter-circuit-design-formulas-and-calculator) I am following. Since we want a low input and output impedance, we can select Z to be something like 5 ohms.
![[Pasted image 20241208164813.png]]

This image shows the two typical topologies for LPF and HPF pi filters. Lets move onto some design equations.

The Inductance and total capcaitance for the circuit are determined by 
$$L=\frac{Z}{\pi f},        C = \frac{1}{\pi Z f}$$ The individual capacitances are just half the total capacitance.

Since we want our filter to work up to 20kHz, lets choose 23kHz as our -3dB point in our Butterworth along with our previously stated 50 ohms of output impedance. This would give an inductor value of:
$$L=\frac{5}{\pi \times 23k} = 69\mu H$$
Very conveniently, the closest standard value is 68$\mu H$, so we will use this in our simulation.  

Our capacitor total value is:
$$C=\frac{1}{\pi \times 5 \times 23k} = 2.8\mu F$$
So, we can use these two values to set-up our pi filter in LTSpice.

### LTSpice Results
![[Pasted image 20241208173844.png]]
 You can see in the figure that even though our filter works to filter out the frequencies we do not want, we have a lot of peakiness in the filter response due to resonance. This peak is around 13kHz so nothing to worry about in terms of  noise due to digital switching (They operate at 1Meg+), but still we don't really want this resonant peak as it can cause instability in analogue circuits. 
 
We can deal with this using a snubber circuit. A snubber circuit is where a resistor is put into an RLC network to dampen out the resonant peak due to the LC filter. Here is what it looks like:
![[Pasted image 20241209201652.png]]
The two 0.1Ohm resistors are part of the snubber (damping) circuit. The larger these resistors, the more damping you get. You must take care not to make these too big as it does increase the ripple on the supply. The current arrangement gives a ripple of 5mVpk-pk at 100kHz, which is pretty good!
![[Pasted image 20241209202926.png]]
You can see that the peaking has been massively reduced. At 100kHz, we only have a 3dB resonant peak which is much better than the 20dBs we were getting without the snubber. Higher resistance values will reduce this number, but will lead to more ripple on the supply rail, which we would like to avoid. We will go ahead with this for now.

## Pre-Amp & Filter Design
For our pre-amp and filter circuit, we want to design an amplifier that can amplify a 300mVpk-pk guitar signal and have a 20Hz-20kHz passband. We will do this by first creating the low-pass filter in the feedback network of our filter circuit, then adding in the high pass filter through the ADC driver circuit from the datasheet. 

To start off, we want to use an inverting amplifier so that we can attenuate any additional noise present in the circuit. To use an inverting amplifier with no negative rail, you need to offset your signal, then add this same offset to the non-inverting pin. This effectively creates a differential amplifier where the difference between the (offset + signal) and the offset is the amplified, so only the signal is amplified!

Here is a TINA-TI simulation of the inverting amplifier. It has an integrated active low pass filter with a corner frequency of 23kHz.
![[Pasted image 20241214152955.png]]

![[Pre-Amp LPF Response.jpg]]

You can see the response being having a -3dB point at 23kHz. We will use this as a first stage low-pass filter. Now, to design the high pass filter.

I want to use a 2nd order high pass filter to get rid of any low frequency hum out of the amplifiers, especially if we use this filter before the speaker. We can either use cascaded first order filters, Sallen-Key filters, or multiple feedback filters. 

It is important to note that if you are making wideband bandpass filters (which we are doing here), they are often best made using a cascade of a HPF and an LPF.  

I've used the [TI Filter Design Tool](https://webench.ti.com/filter-design-tool/design/1) to help me design this filter. Here is the produced design:
![[Pasted image 20241214164116.png]]

Now, I've slightly altered this to account for our input offset by adding in a reference voltage to the non-inverting signal. 
![[Pasted image 20241214164206.png]]
if you don't fully understand the circuit, do not worry. I've only recently stumbled upon it and am trying to understand it myself. From what I've gathered so far, it has a standard input high-pass filter made up of C4 and R6. At the node connecting these two components, we then have two options. Since we have multiple feedback paths, the signal can either go through C6, which means that the gain will be the ratio of C6 to the combined impedance of C4 and R6. The alternative is for the signal to go through C5 and R7, which makes the ratio for the gain to be R7/C5. The offsetting works in the exact same way as it does for other circuits. 
![[Pasted image 20241214164626.png]]
You can see that we have a nice high pass section with a cut-on at 23Hz, but we have introduced a resonant peak at 477kHz. This will be due to the Q-factor of the filter being too high. We will worry about this later when we come to design the anti-aliasing filter for the ADC input.

# ADC Drive Circuit
We will be using a PCM4201 audio ADC to convert the audio signal into a digital version. The drive circuit in the PCM4201 datasheet can be seen in the figure below:
![[Pasted image 20241214173755.png]]
Our input is single ended, so we will be using this version. There is a version for a differential input but this does not apply to guitars as they are single ended. 

Here is the completed schematic including filters and ADC drive circuit!
![[Pasted image 20241214175540.png]]
So we've been through what is happening in the primary filter section. In the bottom section, we are essentially taking our signal, filtering it and inverting it, then inverting it again. This is because the input to the PCM4201 must be differential, which we can easily achieve using this Op-Amp circuit. it is essentially an inverting low-pass followed by another inverter, putting the signal back to where it was!
![[ADC Driver Transient Response.jpg]]
This is the transient response of the circuit. When I first saw this, I was very confused. We want the drive circuit to produce signals that are equal and opposite, but we can see here that the positive peaks are different on both signals. This initially shocked me, but through a quick measurement you can see that the peak to peak of the signals is the same, so we do not need to worry about the input to the ADC as it will still be differential which will eliminate the offset and improve noise rejection. 

So what is causing the difference in the peaks? My leading idea is that it is the input offset voltage. This offset will push the signal one way or another, causing the level to shift on the output. Also, since we are amplifying this offset, its effect is more clear. We have been through 3 gain stages, each of which has cascaded the offset of the first amplifier and added its own. Since we have been using fairly small gain values, we are ok. Here is the frequency response for completeness sake.
![[Pasted image 20241214184458.png]]
The peakines we were seeing earlier has now been attenuated, so won't have any bad noise spikes that resonate. It is 25dB lower (over 10x lower) than the passband, so I am not too worried about it. 

Ok, we've designed the analogue front-end, what is left? Looking back at the system diagram, we need to integrate the ADC/DAC parts of the system along with the Raspberry Pi. There isn't much to design for that on this end, as it is mostly putting it on the PCB schematic/layout then doing the programming. What can we design now? We can design the output pre-amp and amplifier circuits, so then all we have to do is put things into the schematic and do the layout.

As an aside, while doing this design, I though that it would be good to add a Class-AB output that can drive a speaker in the range of 3-5W, an audio jack that can drive headphones at 1-2W, and the intended guitar jack output that can be plugged into a guitar amp. 

# Analogue Output Circuits
![[System Diagram_Updated 1.png]]
Here is an updated system diagram that includes the two new outputs. I did not take the output of the ADC driver circuit for these two amplifiers because I did not want to load the output that drives the ADC as this can have adverse results. We definitely need a pot on these outputs just to limit the power to the output, so we don't blow my headphones!

## 0.5W Amplifier Design
For this amplifier, I want to design a 0.5W class-AB amplifier to drive a headphone jack. For more information on what a class AB output stage is, look at [[Power Amplifier Design]]. 
![[Pasted image 20241217232909.png]]
Since we are running off a 5V single supply, trying to create a 1W amplifier itself is difficult, let alone 5W amplifier! (This is due to issues with headroom, biasing, and current draw). But anyway, The above figure shows the finished design for the Class AB 0.5W Peak Power Amplifier. This should be more than suitable for a headphone amp, and may be able to lightly drive a 3W speaker cone. 

It follows a classic Class AB amplifier topology, using diodes to bias the two output transistors. R10 and R11 are calculated based on signal swing. Due to headroom and signal swing issues, the signal is theoretically limited to ~4V peak. If we know what our peak output current will be at this voltage, we can determine the required base current into the transistors. 

My calculations were carried out for 1W output power (not realistic), but if our peak output voltage is 4V (assume no drop across diodes or transistors (hahahahahaahaha)), then to drive an 8 ohm load at 1W, we need 250mA of current. Assuming a current gain of 25 in the transistors, this means we need 10mA of base current into the transistors. Again, assuming no voltage difference between the diode and input voltage, the voltage at the base of T1 will be 4V. If we then say we want 10mA through R1, we can calculate R1 through ohms law to give 100 ohms.
![[Pasted image 20241217233857.png]]
The art of electronics is much better at explaining this than I am. But basically, the R10 and R11 act as the base resistors.

That's basically all there is to the calculation. Obviously there are a lot of factors that I haven't considered such as headroom issues, temperature instability, and transistor selection, but that is because this is a trial circuit. I aim to work on another project with much more headroom for a better design for a high power audio amplifier. ![[0.5W Amp Transient Response.jpg]]
Above is the simulation result for voltage and current into a 10ohm load under this amplifier. You can see that it can drive just over 1V on the positive (again input offset issues here). 

# ADC & DAC Selection
I will move to using an SPI controlled 16-bit DAC because the main interest here is to use the ADC section to perform digital effects. The digital effect will sufficiently alter the signal such that a 16-bit DAC will be sufficient for this application. (Also 24 bit DACs are expensive)

For the ADC, I will stick to the PCM4201. The DAC I ended up going with is the 16-bit SPI controllable DAC8551. This has an 8LSB relative accuracy with very low glitch energy. What I also liked about this DAC is the integrated output amplifier, so it essentially has an integrated Op-Amp!

# PCB Design
Now that we have done the circuit design and verified it via simulation, we now need to get this onto a printed circuit board (PCB). There is a variety of software you can do this with. Altium is very popular and is the industry standard, but can be quite cumbersome to deal with and to set up. Another option, Proteus, is...an option. I have to use it at work and lets just say, I don't have a good time. There are various other options like EasyEDA, Eagle, OrCAD (Cadence one), but Altium dominates the industry at the moment.

As this is not industry, and I couldn't be bothered to fight with Altium, I decided to use KiCAD. KiCAD is a free PCB design software that is actually really good! It has most of the features that you would ever need (annoyingly except via stitching), has a very nice user interface, and is quite easy to get going with.

There are 2 main aspects of PCB design - Schematic capture and PCB layout & routing. To me, although these are the only two real things, there are a lot of steps to circuit design! To me, these are the real steps in a design:
1. System specification - what will the system do? what is controllable? what is fixed? Are there mechanical constraints? How will you interface with it? Inputs? Outputs? Power? Control? Options/On-the-fly changes? - These are all important things to think about when you first start designing a circuit. 
2. IC Selection - Once you've figured out the spec, you then need to think, do you need specific ICs, or is it fully discrete/Op-Amp based? If the latter, then you can go to simulation. If the former, you will need to select an IC to meet the requirements!
3. Simulation - If you are simulating a circuit, there are a lot of tools you can use. LTSpice is good for quick and dirty simulations, but I never feel like I can trust it. I know it is correct... maybe its the user interface. Throughout this guide I have used TINA-TI which is a spice simulator developed by TI that contains a lot of TI parts. It is really good for analogue parts once you get used to it. The last option that I have used is PSpice for TI. This contains more complex models for ICs that you can simulate. I simulated an LED driver circuit using PSpice and it worked surprisingly well (it did take forever to run). 
4. Schematic Capture - This is the part where you put all your circuit components down and connect them up. You usually do this after simulation, or, if you are using an IC, while you're carrying out the design. Most ICs will be fairly straightforward, but some things will take longer and need more thought! 
5. PCB Layout - Now that you've set up your components, how they connect to each other, and how it all connects to form a system, its time to put it all in some physical space! This part can either be extremely satisfying or extremely frustrating, depending on your experience and choices in the schematic capture. Always remember, do your schematics so that layout you will thank you.
A lot of PCB design comes with experience. Knowing these steps, knowing what to do with layout, knowing how to make your life easier, all comes with experience. It wont be great the first time. My first PCB cost £300 in manufacturing, and a lot more in parts... It didn't really work either just to top it all off!

I wont talk through everything I have done for this design, but I will mention the highlights here.
## Schematic Capture
![[RaspberryPI_Audio_Interface_Schematics.pdf]]
Main points to note:
1. Add comments and images to your design. When your schematic is in progress, feel free to add comments, images, boxes, or anything you want to it! It is your drawing board and do whatever makes sense to you and helps your design. I tend to create To-Do lists, and paste in images that are relevant or important to remember. 
2. Use ports and labels - Creating ports really cleans up your schematic. If you look at page 4 above, I am using ports from the raspberry pi connector to my DAC and ADC. This means you don't have to put everything on the same schematic page or have traces everywhere. It makes sense sometimes to have it all connected, as on page 3, if it is all part of one circuit series. Again, you will learn with experience where you should use a port and where you can just connect things. If in doubt, use a port!
3. Make schematics easy to read. if you cant read your own schematic, you can't verify your design, and you can't debug it if something goes wrong. I like to stick to the standard circuit block configurations if possible. A lot of parts you can get from library loader will just come in an annoying box. If you look on page 6, you will see that I have just labelled each part of the dual transistor package and kept the same overall circuit shape as if the transistor symbol were there. This really helps clean up the schematic and makes it easier to verify!
## PCB Layout
![[Pasted image 20250113210856.png]]
There is a lot that goes into layout that I wont talk about here, as there is a lot of intuition that goes into layout once you have a basic understanding, but here are the main points:
1. Keep feedback loops short - If you use long traces, the trace will have an associated impedance which can change your gain and frequency response. 
2. Have inner layers! - It is cheaper to have a 2 layer board (top & bottom), but you will always have everything connected to power and ground. If your board has high current whilst also having signals, routing power isn't a great idea. It can also increase ground loops which you want to avoid, otherwise your ground for one part isnt the same as the other! Having inner layers allows you to drop vias through the board into an inner copper plane where everything is connected to the same potential, and is (in theory) at the same potential. Ground loops can still exist in this case, but it greatly helps minimise them.
3. Constrain yourself and layout the large things first based on specification.  - This board is designed to fit over a Raspberry Pi 5, so the footprint matches the [Raspberry Pi DAC Pro](https://www.raspberrypi.com/documentation/accessories/audio.html#raspberry-pi-dac-pro) Footprint. With this, the board is constrained in dimensions, as well as where I can put things on the edge. Then follow this by setting out your connectors. Finally, you can begin to section out your components.
4. Place everything that is in a signal path as close as possible - This makes things easier to route, prevents you from having to route on the other side of the board, and provides a clear and logical path for signal flow. 
I cant really sum up every decision I made on this board, but I am happy to have a chat if you have any questions.

And that's in on circuit design! Now to export the Gerber files and get it manufactured. I tend to use JLCPCB for custom PCBs as they are very quick and cheap. 

The next step on this board after ordering is assembly is testing, followed by programming.
# Programming 
There are a few things that go into the software initialisation of this audio interface. 
## Setting up the PCM4201
The PCM 4201 needs to be set up according to the datasheet [PCM4201 Datasheet](https://www.ti.com/lit/ds/symlink/pcm4201.pdf?ts=1735848902402&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FPCM4201). There are multiple set-up options for this including the clock frequency, sampling mode/rate, and digital high-pass filter.
### Master/Slave Configuration
![[Pasted image 20250318201652.png]]
Since we want the PCM4201 to work in Master mode, we will set the pin to 0
### Clock Frequency
In my specific case, the PCM4201 will be in master mode, as it is transmitting data downstream to the pi. I will need the GPClock running from the Pi at a specific frequency for the PCM4201 to work. 
To output sample at 44.1kHz, I will need a clock frequency of 22.5792MHz from the GPClock.
![[Pasted image 20250318200013.png]]
### Sampling Modes
Normal speed, low power mode has the lowest power dissipation and runs best off of a 1.8V VDD. I dont care too much about power dissipation as this is not handheld, so I will look at using Normal speed, high performance, which has the best THD+N with sampling rates up to 54kHz. For this mode, I can leave the pin floating but they say it is best to drive it with a high-impedance buffer. For now, I will use low power mode so that I can know the pin state
![[Pasted image 20250318200952.png]]
### Audio Serial Port
The PCM4201 audio serial port is a 3-wire synchronous serial interface comprised of the audio serial data output, DATA (pin 9); a frame synchronization clock, FSYNC (pin 10); and a bit or data clock, BCK (pin 11). The FSYNC and BCK clocks may be either inputs or outputs, supporting either Slave or Master mode interfaces, respectively. The audio data format is 24-bit linear PCM, represented as two’s complement binary data with the MSB being the first data bit in the frame. Figure 3 illustrates the audio frame format, while Figure 4 and the Electrical Characteristics table highlight the important timing parameters for the audio serial port interface.
![[Pasted image 20250318201543.png]]
### Digital High-Pass Filter
![[Pasted image 20250318202937.png]]
Lets pull this low to enable it.
### Reset Operation
On power up, the internal reset signal is forced low, forcing the PCM4201 into a reset state. The power-on reset circuit monitors the VDD (pin 13) and VCC (pin 4) power supplies. When the digital supply exceeds 0.6 × VDD nominal ±400mV, and the VCC supply exceeds +4.0V ±400mV, the internal reset signal is forced high. The PCM4201 will then wait for the system clock input (SCKI) to become active. Once the system clock has been detected, the initialization sequence begins. The initialization sequence requires 1024 system clock periods for completion. During the initialization sequence, the ADC output data pin will be forced low. Once the initialization sequence is completed, the PCM4201 outputs valid data. Figure 7 shows the power-on reset sequence timing
![[Pasted image 20250318203245.png]]
![[Pasted image 20250318203300.png]]
Now that we have mostly configured, or know what we are missing for the PCM4201, we can move on to the pi.
## Raspberry Pi Config
We want to configure the pi to be able to read the incoming I2S/PCM data. To do this, we will first need to start up the GPClock.

**Might have to abandon the Raspberry Pi as the I2S interface. The support isn't great generally and is even worse on the pi5. Currently looking at either an STM32F4 or an ESP32**






# References
[Introduction to LC Filters](https://www.youtube.com/watch?v=KECCpj1Qpn0)
[TI Filter Design Tool](https://webench.ti.com/filter-design-tool/design/1)
[Headphone Specs](https://www.youtube.com/watch?v=PqlcJnoIcGo)
[PCM4201 Datasheet](https://www.ti.com/lit/ds/symlink/pcm4201.pdf?ts=1735848902402&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FPCM4201)
[DAC8551 Datasheet](https://www.ti.com/lit/ds/symlink/dac8551.pdf?ts=1735838904345&ref_url=https%253A%252F%252Fwww.mouser.de%252F)
[Raspberry Pi 5 Pinout](https://pinout.xyz/pinout/pin19_gpio10/)
[OPA161x Datasheet](https://www.ti.com/lit/ds/symlink/opa1611.pdf?ts=1733937056128&ref_url=https%253A%252F%252Feu.mouser.com%252F)
[Raspberry Pi DAC Pro](https://www.raspberrypi.com/documentation/accessories/audio.html#raspberry-pi-dac-pro) 

 