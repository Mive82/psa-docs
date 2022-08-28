# Peugeot infotainment Project ESP32 documentation

This a brief overview of what the ESP's firwmare does.

# Basic functions

The ESP's purpose is to function as a bridge between a computer (eg. a Raspberry Pi) and the VAN bus. It parses packets sniffed from the bus, responds to message requests, controls the Pi's power status and acts as a Real Time Clock (RTC) to keep the time.

# Firmware overview

The ESP32 firmware is built on top of FreeRTOS, so the basic program workflow is centered around tasks. There is a task for every major function in the firmware. The ESP32 also has 2 processor cores, and tasks can optionally be assigned to one or the other.

## Tasks

Core 1 runs the arduino `loop()` task, and no other tasks are pinned to it. It's used to run the MSP message receiver and sender, and to periodically send CD changer information. Since this is time critical, no delays are running on this core.

Core 0 runs other non time critical tasks:

- Receiving and parsing VAN messages
  - Reading radio button states
- Determining the Car's state
- Controlling the Raspberry Pi power state
- Wifi event handling
- Setting the time

## RTC

The ESP has an onboard Real Time Clock with its own SRAM. It's always powered, even in deep sleep so it can be used to store data over reboots.

Here it's used to keep the time, and remember radio preset information. In the future, it will be used for other settings.

## Power saving modes

The ESP supports a few power saving modes.

The mode used here is deep sleep, where everything except the RTC is powered off. In this mode, the current draw should be minimal (around 150 uA). The ESP goes into this mode when the car goes to sleep and turns off the VAN+ power line, and is woken up when the VAN+ power line turns on.

For more information on power modes used in this project, check out the Car state table in the project's manual.

## Van reading and parsing

The ESP constantly reads data from the bus, parses it and organizes the data in the respective MSP buffers. Those buffers contain messages which the ESP sends to the Raspberry when it requests data.

## CD Changer emulation

In order for the car to accept CD changer packets, they must be sent periodically (once every second is fine) with at least the header and footer changed. The ESP can also accept new data to send for music duration or track information. When such data is received it is immidiately sent to the car.

Since the CD changer frame is an in-frame reply frame, the display will be updated only when it sends a request for the information.
