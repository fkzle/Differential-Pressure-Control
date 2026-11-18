# Differential-Pressure-Control
Hacking the communication of LEDA LUC2 


The LEDA LUC2 is a differential pressure watch safety device (according to its manual). 
Its purpose is to switch of the central ventilation system or the fume hood, if the chimney does not generate enough negative pressure to bring the fumes of the wood stove out of the building. 
In germany, it is under certain circumstances mandatory to install such a device.

It consists of 2 (maybe 3) main parts: 
 * a DIN-Rail part that has a relais to cut off power to the ventilation system and acts as power supply for the whole system
 * a display unit that is installed near the stove, consisting of 2 parts
   * the display itself
   * the measurement device 
 
 The units are connected via RJ12 plugs, in the manual you can find the pinout:
 
 1. GND
 2. CAN H (Wrongly labeled "CAN RX")
 3. CAN L (Wrongly labeled "CAN TX")
 4. GND
 5. +12V (+-3V)
 6. +12V (+-3V)

# Breakout PCB
First step was to build a breakout PCB that has 2 RJ sockets. I had RJ45 sockets in stock, so i used these. RJ12 plugs can be plugged into RJ45, so this was fine. The other side of the board were screw-terminals 

# CAN datarate and pinning
With the breakout PCB working, it was time to lug my old oscilloscope to the breakout board. Sadly i forgot to take pictures. 
I found out that the datarate is 125kBaud and which one is CAN-H and which is CAN-L.

# CAN-communication 
The CAN communication consists of 4 different messages that use the same CAN-ID. 
These are distinguished by their first byte: 

| Message | Byte 0 | Byte 1 | Byte 2 | Byte 3 | Byte 4 | byte 5 | byte 6 | byte 7 |
| --------|--------|--------|--------|--------|--------|--------|--------|--------|
| 0 | 0x55 | message counter LSB | message counter MSB | 0x04 | 0x05 | 0x06 | 0x07 | 0x0D |
| 1 | 0x00 | pressure LSB | pressure MSB | temperature LSB | temperature MSB | reserve | reserve |0x0D |
| 2 | 0x01 | offsetpressure LSB | offsetpressure MSB | zero calibration pressure | 400degree threshold | ventilation active | inactivity time remaining |0x0D |
| 3 | 0x09 | message counter LSB | message counter MSB | 0x00 | 0x00 | 0x00 | 0x00 | 0x0A |



**construction:** Byte 1 is the Identifier (Multiplexor).
Bytes 2–7 contain 6 databytes.
Byte 8 is for flow control and marks the end of the sequence. 


## CAN-message 1 – Multiplexor `0x55`

| Byte | Signal | description |
|---:|---|---|
| 1 | Multiplexor | `0x55` |
| 2 | Message counter LSB | |
| 3 | Message counter MSB | |
| 4 | databyte 3 | always 0x03 |
| 5 | databyte 4 | always 0x04 |
| 6 | databyte 5 | always 0x05 |
| 7 | databyte 6 | always 0x06 |
| 8 | flow control byte | 0x0D for "message following" |

## CAN-message 2 – Multiplexor `0x00`

| Byte | Signal | description |
|---:|---|---|
| 1 | Multiplexor | `0x00` |
| 2 | Pressure LSB | Low-Byte des Druckwertes |
| 3 | Pressure MSB / sign | High-Byte / Vorzeichen des Druckwertes |
| 4 | Temperature LSB | Low-Byte der Temperatur |
| 5 | Temperature MSB | High-Byte der Temperatur |
| 6 | Reserve | always 0x00 |
| 7 | Reserve | always 0x00 |
| 8 | flow control byte | 0x0D for "message following" |

## CAN-Botschaft 3 – Multiplexor `0x01`

| Byte | Signal | description |
|---:|---|---|
| 1 | Multiplexor | `0x01` |
| 2 | offset pressure LSB | offset pressure from the menu (Offsetdruck) |
| 3 | offset pressure MSB | offset pressure from the menu (Offsetdruck) |
| 4 | zero calibration pressure | (Nullabgleichdruck) |
| 5 | 400degree threshold | "400-Grad-Grenzwert" from the menu |
| 6 | ventilation active | 0 = inaktive, 1 = aktive |
| 7 | inactive time | if ventilation is switched off from the menu, the remaining time (in min) until it is activated again is sent here |
| 8 | flow control byte | 0x0D for "message following" |

## CAN-Botschaft 4 – Multiplexor `0x09`

| Byte | Signal | description |
|---:|---|---|
| 1 | Multiplexor | `0x09` |
| 2 | Message counter LSB | |
| 3 | Message counter MSB | |
| 4 | databyte 3 | always 0x00 |
| 5 | databyte 4 | always 0x00 |
| 6 | databyte 5 | always 0x00 |
| 7 | databyte 6 | always 0x00 |
| 8 | flow control byte | 0x0A for last message |

## Zusammenfassung

Message 1 is only sent if the ventilation system is active. This is missing if the ventilation is set to "inactive". 

| Multiplexor | CAN-Botschaft | Inhalt |
|---|---|---|
| `0x55` | 1 | ventilation active message |
| `0x00` | 2 | pressure and temperature - fast changing values |
| `0x01` | 3 | diagnosis values - slow changing and only visible inside the menu of the LUC2 |
| `0x09` | 4 | end message |


## Switching the ventilation on and off

### off
Using the same CAN-ID (28A) and sending a Message with the first 2 bytes only, containing [0x82, 0x00] switches the ventilation of for the configured time. 

### on
sending a can message on CAN-ID (28A) with the first byte only set to [0x08] (length = 1) reactivates the ventilation. 


# esphome code

The esphome code contains a switch for switching the ventilation on and off. 
The implementation has a template switch that sends the CAN-messages on switching. But since this only lasts for the configured amount of time (in my case 6 minutes), the remaining time sensor is monitored and if it nears the 0, the message for switching off is sent again. 
