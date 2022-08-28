# Peugeot project development notes

Various notes written down during development.

# Pinout notes

White - DATA
Gray - DATAB

## ESP8266

### VAN

D1 i D2 koristiti za VAN bus - najsigurniji pinovi

### SPI

```
D5 - SCLK
D6 - MISO
D7 - MOSI
D8 - CS
```

## ESP32

### VAN

```
D21 - VAN_RX (Level shifter)
```

### TSS463C (SPI)

Connected to `VSPI` on ESP32

```
D5 - CS
D18 - CLK
D19 - MISO (Level shifter)
D23 - MOSI
```

# Hardware notes

## ESP32

### Sleep modes

The ESP is running from persistent battery power. When VAN+ is gone, put the ESP into deep sleep mode.

When the clock has been syncronized with NTP, put the radios to sleep.

### RTC

Add option to power only the ESP with persistent 12 volts, and implement sleep.  
VAN+ would be used to power everything else.  
Add a jumper to power the ESP from VAN+ if persistent 12 volts are not available.

<https://lastminuteengineers.com/esp32-ntp-server-date-time-tutorial/>

<https://esp32.com/viewtopic.php?t=13699>

### Voltmeter (Later)

Plan is to measure battery voltage (up to 25V).  
Use a voltage divider and an ADC to measure voltage and interpolate from the resistor values.

<https://duino4projects.com/arduino-digital-voltmeter-0v-30v/>

Use pin 32.

### References

<https://lastminuteengineers.com/esp32-sleep-modes-power-consumption/>  
<https://lastminuteengineers.com/esp32-deep-sleep-wakeup-sources/>  
<https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/system_time.html>

## Hazard switch

~~Potentially implement the hazard switch in software as it will have to be removed.~~
Nope, will have to reposition it, as software won't be running all the time.

### Pinout

| Pin | Function | Notes              |
|-----|----------|--------------------|
| 3   | GND      |                    |
| 4   | LED      | 5V I guess         |
| 6   | Button   | Acts as a pulldown |

# Messages to monitor

## Iden 0x824

RPM, speed and meters?? information

Len 7

## Iden 0x564

Trip computer and door information

Len 27

## Iden 0x8A4

Ignition, engine temp, external temp, total mileage in km

Len 7

## Iden 0xE24

VIN

Len 17

## Iden 0x554

Radio tuner / CD / Cd changer info

Tuner Len 22  
Tape Len 5  
Radio preset Len 12  
CD player Len 10 or 19

## Iden 0x4D4

Headunit settings and power status

Len 11

## Iden 0x8EC

Cd Changer commands

Len 2

## Iden 0x8C4

Radio button reports

Varius lenghts, the one I'm looking at is Len 3

## Iden 0x4FC

Fuel level, Oil level, service indicator

Len 11 (<=2002)

Len 14 (>2002)

## Iden 0x524

MFD messages. Used to determine door lock status.

Len 14 (<=2002)

Len 16 (>2002)

## Iden 0x554

Radio tuner, presets, and CD / tape information

The length and first byte after the header determine the type of info

Len 22, first byte is 0xD1 - Radio tuner information

Len 19, first byte is 0xD6 - CD player information

Len 12, first byte is 0xD3 - Radio presets information

# Messages to send

## Iden 0x4EC

Cd changer status for CDC emulation.

# General notes

## Packet reading

- Receving a lot of data with the TSS463C does not really work, or I'm not doing it correctly.
- Will have to use a software reader (ESP32) to read packets
- The TSS463C will still send cd changer data (and potentially other data)

## ESP32 multithreading:

- <https://randomnerdtutorials.com/esp32-dual-core-arduino-ide/>
- <https://www.instructables.com/ESP32-With-Arduino-IDE-Multi-Core-Programming/>

## Gear ratios

This is something that is hardcoded to my car.

Formula: $\frac{\text{SPEED}}{\text{RPM}} \cdot 100$

- `1st` - $[0.7 - 0.8]$
- `2nd` - $[1.42 - 1.47]$
- `3rd` - $[2.02 - 2.07]$
- `4th` - $[2.65 - 2.73]$
- `5th` - $[3.38 - 3.44]$

## Car state

Idea is to collect all relevant information and generalize it into one 'car state`.

| State                    | Description                                                         | Raspberry behaviour                                             |
|--------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------|
| `PSA_STATE_CAR_SLEEP`    | Car in sleep mode (VAN+ is off)                                     | Turned off (no power to VAN+).                                  |
| `PSA_STATE_ECONOMY_MODE` | Car is in Eco mode (battery saving). Reset when engine is turned on | Turned off.                                                     |
| `PSA_STATE_CAR_LOCKED`   | Car off and locked, but not in sleep mode yet                       | Turned off.                                                     |
| `PSA_STATE_CAR_OFF`      | Car turned off, but not locked and not in sleep mode                | If coming from states above, turned off, otherwise display off. |
| `PSA_STATE_RADIO_ON`     | Car turned off, but radio is on. By button or by accessory switch   | Turned on, display on.                                          |
| `PSA_STATE_ACCESSORY`    | Car turned off, key in accessory position                           | Turned on, display on.                                          |
| `PSA_STATE_IGNITION`     | Car turned off, key in ignition position                            | Turned on, display on.                                          |
| `PSA_STATE_CRANKING`     | Car is cranking the engine                                          | Turned on, display off.                                         |
| `PSA_STATE_ENGINE_ON`    | Engine is running                                                   | Turned on, display on.                                          |

## Audio settings

Not all audio settings broadcast when they are being changed.
The order for my car is: Bass -> Treble -> Loudness -> Fader -> Balance -> Auto volume

The settings menu for my car isn't broadcast properly, so I'm doing it manually.

### Audio button message

The following message appeares when the audio settings button is "let go"

`8A 22 56` with iden `0x8C4`

# Other notes

## CD changer information

The CD changer expects values in BCD: 0x16 instead of 0x10 to get 16.

https://www.eevblog.com/forum/projects/van-bus-interfacing-(to-read-car-speed-and-engine-rpm-peugeot-206-1-4-hdi-2002)/75/

```c
// #include "VanMessageSender.h"
#include <itss46x.h>
#include <tss46x_van.h>
#include <tss463.h>

const int VAN_PIN = 7;

TSS46X_VAN *VANInterface;
ITss46x *vanSender;
unsigned long currentTime = millis();
unsigned long previousCdcTime = millis();
volatile uint8_t seconds = 0;
volatile uint8_t headerByte = 0x80;

void IncrementHeader()
{
    if (headerByte == 0x87)
    {
        headerByte = 0x80;
    }
    else
    {
        headerByte++;
    }
}

void IncrementSeconds()
{
    seconds++;
    if (seconds == 60)
    {
        seconds = 0;
    }
}

uint8_t DecToBcd(uint8_t input)
{
    return ((input / 10 * 16) + (input % 10));
}

void AnswerToCDC()
{
    uint8_t status = 0xC3; // playing
    uint8_t cartridge = 0x16;
    uint8_t minutes = 0x01;
    uint8_t trackNo = 0x17;
    uint8_t cdNo = 0x02;
    uint8_t trackCount = 0x21;

    uint8_t packet[12] = {headerByte, 0x00, status, cartridge, minutes, DecToBcd(seconds), trackNo, cdNo, trackCount, 0x3f, 0x01, headerByte};
    VANInterface->set_channel_for_immediate_reply_message(0, 0x4E, 0xC, packet, 12);
}

void setup()
{
    VANInterface = new TSS46X_VAN(vanSender, VAN_125KBPS);
    VANInterface->begin();
}

void loop()
{
    currentTime = millis();
    if (currentTime - previousCdcTime >= 1000)
    {
        previousCdcTime = currentTime;

        IncrementSeconds();
        IncrementHeader();

        AnswerToCDC();
    }
}
```

# Yocto

## Audio

<https://forum.qt.io/topic/101103/yocto-qt5-alsa-raspberrypi-usb-audio/6>

# Qt

## Qt text scrolling

<https://stackoverflow.com/questions/48640888/make-qlabel-width-independent-of-text>

## Qt replay gain

```
dbVolume = 20*log10(volume/100);
correctedVolume = pow(10, (dbVolume+replay_gain/2)/20)*100;
```

## Music library/playlist

Odvojeno folder i playlist view.

### Folder view

- Tipka lijevo dolje
- Izlistati foldere bez file-ova
- Kada se klikne na folder bez djece, shuffle-aj sve u folderu

#### Behaviour

- Počni u `root_music_directory` folderu.
- Mijenjaj `music_directory` direktorij.
- Stavi `<--` strelicu za povratak u zadnji folder.
- Nema strelice u `root_music_directory` folderu.

### Playlist view

- Tipka desno dolje
- Izlistaj playlistu u stanju kojem je
- Popuni tagove kako treba

## To Do:

- Objedini sve relevantne pakete u jedan veliki paket da smanjim delay izmedu informacija.
- ~~Popravi fontove za radio presete~~
- Dodaj memoriju za radio presete
- ~~Makni prozirnost sa popup-ova~~
- ~~Volume za sve neka bude na 100%~~
- ~~Adaptive brightness~~ (time of day za sada)
- ~~Skontaj kako backprobe-ati pin sa stalnih 12 Volti~~ Nema potrebe, jer na cd changer portu postoji stalnih 12 volti.
- ~~Signal display se ne pojavljuje (pojavljuje se, al to nije to)~~
