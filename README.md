# Loxone Nano Dimmer Air - MOSFET Failure Problem

## The Problem

Recently, I bought a few Loxone Nano Dimmer Air modules for dimming halogen lights (around 150W-200W). However, almost all of them failed after some time - I was not able to dim the lights, which remained at about 50% power and flickering. After some investigation, I found that one of the internal MOSFETs (STD16N65M5) was blown.

![nano-dimmer-air](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/d8bd8566-92d5-4a31-8aa7-ddf67e9a7056)

These dimmers do not use triacs; instead, they use a couple of MOSFETs, functioning like a solid-state relay. These dimmers can work in leading or trailing edge mode (cutting off the beginning or the end of the sinus wave) - more info can be found here: [Leading and Trailing Edge LED Dimmers](https://www.lamps-on-line.com/leading-trailing-edge-led-dimmers).

Setting to leading edge lowers the probability of failure. Interesting, isn't it?

I did some basic investigation.

There is no standard 12V zener diode protecting the gate of the MOSFET. However, there are pads on the board for the zener diode, so it can be added. Maybe it is not there due to zener diode capacitance - these MOSFETs have very low input capacitance on the gate, making them easy to drive. Perhaps they did not want to add a zener diode to increase the capacitance. Who knows. Adding the zener diode did not solve the problem.

The MOSFETs are protected by transils/varistors. Adding a 400V transil over each MOSFET also did not help.

There is an overcurrent circuit. I was suspicious that, as halogen bulbs have much lower cold resistance, the overcurrent circuit oscillates when the dimmer tries to light up cold bulbs, but that was also a false lead.

I was getting desperate. It was hard to catch the situation where the dimmer fails. It was random and always came exactly when I was not prepared.

I had to do reverse engineering:

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/46fb7f7e-0c83-4178-abdd-602211bab7b0)

## Dimmer Description

The dimmer has three boards. The main board with MOSFETs, relay, supplies, etc. The second board with the MCU and radio, and the third one with MOSFET drivers (the one on top, soldered into the main board with long pins).

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/113a0c3d-996a-4467-a0b1-09ea7dfe437e)

This is a schematic of the MOSFET driver board (the one on the top) and the MOSFETs.

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/3e91e25a-0268-44e9-92a0-054556754c98)

V2 emulates the drive from the MCU. V1 is the 230V supply. R26 emulates the load. I did not draw transils and X2 protection capacitors.

There are:
- A simple power supply with a zener diode.
- I guess undervoltage protection (gates A1, A2).
- Overcurrent protection: LM324 as an amplifier and comparator (measurement resistor is 0.22 Ohm, op amp gain is 3.2, reference voltage is about 8.6V -> it limits current at 12A).
- Q1 and Q2 keep gates low when overcurrent is triggered.

Design issues found:
- The dimmer does not measure MOSFET temperature - there is a temperature protection, but it measures MCU temperature. When the dimmer fails, it can melt its case before the temperature protection engages.
- Old-school driver circuit with relay, zener-based power supply (which is not effective). It would be better to use some drivers like [Si8751-2](https://sound-au.com/pdf/Si8751-2.pdf). Theoretically, no relay would be needed to switch off the driver board. It is also capable of providing 50mW on the high side of the MOSFETs to supply e.g. the overcurrent circuit.
- Weak drive of MOSFETs.
- Long paths from the driver board to the MOSFETs.
- No snubber circuit?

There are three main flaws:

1. Weak MOSFET drive. The gate is held off by a 4k7 resistor (which is driven directly from the HEF40106BT Schmitt).
2. Miller capacitance (capacitance between drain and gate).
3. Fast dV/dt on the drain of MOSFET M1 and no snubber circuit.

## The Solution(s)

### MOSFET Drive

MOSFETs used in this dimmer are known for easy drive (no extra driver needed). However, in combination with Miller capacitance, it can be a problem. The first thing I tried was to add a simple solution that fixes the problem: an old-school approach with a PNP transistor half-bridge/discharge helper. Nice description [here](https://electronics.stackexchange.com/questions/617572/how-does-this-pnp-transistor-improve-the-switching-performance-of-mosfet).

Modified schema:

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/a8d03b37-00a5-4faf-abbd-a36a35360be7)

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/411e2575-bb43-468e-ad33-86d5b116e515)

R14 and D5 were interchanged and a PNP transistor Q4 was added. Type BC327 was used. This will help the weak MOSFET drive. However, it is still not the right solution. Even adding a MOSFET driver like [TC4426](https://img.gme.cz/files/eshop_data/eshop_data/8/329-035/dsh.329-035.1.PDF) does not help.

### Optocoupler

I got my hands on an updated HW with a different optocoupler with an external base [IL211A](https://www.vishay.com/docs/83615/il211at.pdf). There is a 47k resistor between the base and emitter - this should make the optocoupler faster. The dimmer seems more stable with this solution, but the main problem is still fast dV/dt.

### Snubber

Simply adding a 100n capacitor over M2 solves the problem. The problem with high dV/dt on M2 is that GND is actually floating, and when M2 closes, GND goes up by 300V at high speed, causing interference and oscillations/ringing. The proper solution is to add a small RCD snubber circuit similar to the one used in switching power supplies.

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/402d41aa-29bc-448e-af73-d82d269d94e5)

Dimmer with added 100n capacitor:

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/046a5eee-3b0f-40bc-9d9f-4b9fb3c561a1)

MOSFET gate signal when oscillating (blue):

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/9ad82eef-e889-4623-b8d8-061f02f953fa)

MOSFET gate with snubber(blue):

![image](https://github.com/ivantichy/loxone-nano-dimmer-air/assets/11446854/5b0852cf-f283-421d-b78b-c78b328bdf74)

# It took 5 months of my life!
