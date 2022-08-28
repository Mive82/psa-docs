# Peugeot Project PCB

This is a brief overview of the PCB used for the Peugeot Infotainment project.

This is version 1 of the PCB, and some functionality is missing

## Form factor

The PCB was designed as a Raspberry Pi HAT.

## Function

The PCB contains all electronic components (except the power supply) necessary to read and write VAN packets.

## Components

The bom is located inside `bom.html`. It is a self-contained interactive BOM. Just load it in your favorite browser.

### Big ICs

- ESP32 header - A header for the ESP32 dev board. Meant to be elevated from the board
- Level shifter (U2) - A header for a simple 5 - 3.3 level shifter. Can be elevated or not.
- TSS463C - A VAN bus driver chip. Used for sending and receiving messages from the VAN bus
- MCP2551 (U4) - A CAN transciever
- 7805 (U5) - A 5V voltage regulator used to power the ESP. Connected to permanent battery power.

### SMD components

See BOM

### Unused components

- R4 - line termination resistor (DO NOT SOLDER, causes errors in the car)
- R5 - unneccessary pull-down resistor

### Headers

J1 - ESP power selector - Select how the ESP gets powered
J2 - Permanent 12 Volt power input pin
J3 - 5 volt input, ground and VAN data wires
J4 - LCD power output

### Power lines

The PCB has 2 power lines:

- A permanent 12 volt line - powers the ESP through a 7805 regulator
- A 5 volt switched(ish) power line - Regulated externally and connected to VAN+. This is powered when the car is not sleeping. It powers the Rasbperry Pi and the LCD.

Both power lines share a ground.

## Missing functionality

This board doesn't have a female header for connecting to the Raspberry's `RUN` header, and therefore can't turn the Pi off. A solution would be to jump pin `D13` from the esp to the `RUN` pin on the Pi.

Future boards will have an on board power supply, and will switch the Pi off that way.
