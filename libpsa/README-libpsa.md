# libPSA - Communication API

Communication API between an ESP32 and a Raspberry Pi for the Peugeot infotainment Project.

Implementation example can be found in the [gui app](https://github.com/Mive82/psa-app).

# Overview

The libPSA API facilitates the communication between an ESP32 and a Raspberry Pi connected over UART.

The ESP is connected to the car's VAN bus, parses the VAN message frames and organizes them in a predefined message structure. It also acts as a Real Time Clock (RTC) and controls the power-on state of the Raspberry Pi.

The Raspberry Pi requests information from the ESP at regular intervals, and acts accordingly.

# API contents

This API contains two things:

- Message structure
- Functions for sending messages

## Message structure

The message structure format is inspired by the MSP, or MultiWii Serial Protocol used by many drone flight controller firmwares such as Betaflight for telemetry and configuration information.

The basic format is as follows:

| Header  | Data    | CRC-8 Checksum |
| ------- | ------- | -------------- |
| 5 bytes | N bytes | 1 byte         |

The maximum allowed size of a packet is 100 bytes $\rightarrow$ max data size is 94 bytes.

The header contains:

| Start magic      | Direction                 | Identifier                                 | Data size (N)          |
| ---------------- | ------------------------- | ------------------------------------------ | ---------------------- |
| `0x69` (`uint8`) | `'>'` or `'<'` ASCII char | Packet identifier (`uint16` little endian) | Size of data (`uint8`) |

Data is an array of N bytes.

The Checksum is an XOR checksum calculated by XOR-ing every byte in the packet apart from the start magic, direction, and the crc itself.

### CRC Example code

The following is an Example implementation of calculating the checksum.

The `packet` argument is the entire packet buffer (header + data + crc), and the `size` argument is the size of the entire packet (`data_size` + 6)

```c
uint8_t calculate_crc(uint8_t const *const packet, const uint8_t size)
{
    uint8_t crc = 0;

    for (int i = 2; i < size - 1; ++i)
    {
        crc ^= packet[i];
    }

    return crc;
}
```

## Message Types

There are two message types: GET and SET messages.

### GET messages

GET messages request data from the ESP, and only consist of a header. The header contains the ident of the data we would like to receive. The data size is set to 0, and the CRC is not present.

An example of a GET message:

| Start magic | Direction      | Identifier                                | Data size (N) |
| ----------- | -------------- | ----------------------------------------- | ------------- |
| `0x69`      | `0x60` (`'<'`) | `0x02` `0x40` (`0x4002` in little endian) | 0             |

The response would follow:

| Start magic | Direction      | Identifier    | Data size (N) | Data                                                           | CRC    |
| ----------- | -------------- | ------------- | ------------- | -------------------------------------------------------------- | ------ |
| `0x69`      | `0x62` (`'>'`) | `0x02` `0x40` | 9             | `0x00` `0x00` `0x00` `0x00` `0x55` `0x57` `0x00` `0x34` `0x12` | `0x6f` |

Not all message idents support all message formats.

### SET messages

SET messages send data to the ESP. Both the message and the response are full MSP packets.  
The response from the esp is a one byte message indicating the receive status.

The SET response can have one of the following statuses:

| Byte value | Meaning                                                |
| ---------- | ------------------------------------------------------ |
| `0x0E`     | Packet is OK                                           |
| `0xAA`     | Packet has invalid CRC                                 |
| `0xBB`     | Packet has an unknown Ident                            |
| `0xCC`     | The data size for the specified Ident is not valid     |
| `0xDD`     | The received data for the specified Ident is not valid |
| `0xFF`     | Generic (unknown) error                                |

An example of a SET message:

| Start magic | Direction      | Identifier    | Data size (N) | Data                                             | CRC    |
| ----------- | -------------- | ------------- | ------------- | ------------------------------------------------ | ------ |
| `0x69`      | `0x60` (`'<'`) | `0x01` `0x45` | 7             | `0x31` `0x15` `0x01` `0x03` `0x01` `0x21` `0x01` | `0x44` |

The example response would be:

| Start magic | Direction      | Identifier    | Data size (N) | Data   | CRC    |
| ----------- | -------------- | ------------- | ------------- | ------ | ------ |
| `0x69`      | `0x62` (`'>'`) | `0x01` `0x45` | 1             | `0x0E` | `0x4B` |

## Message Idents

The message idents currently supported by the Infotainment program are listed below. All numbers are little endian `uint16`.

Note: All size values are subject to change. If they are not listed, that means that they still aren't finalized. For up to date message structures see the header file.

| Ident value | Meaning                                         | enum                            | Type | Data Size in bytes |
| ----------- | ----------------------------------------------- | ------------------------------- | ---- | ------------------ |
| `0x4001`    | VIN number, not NULL terminated                 | `PSA_IDENT_VIN`                 | GET  | 17                 |
| `0x4002`    | Engine RPM, speed, temp, fuel level and reverse | `PSA_IDENT_ENGINE`              | GET  | 9                  |
| `0x4003`    | Radio station information                       | `PSA_IDENT_RADIO`               | GET  |                    |
| `0x4004`    | Known radio presets                             | `PSA_IDENT_RADIO_PRESETS`       | GET  |                    |
| `0x4005`    | Door information                                | `PSA_IDENT_DOORS`               | GET  |                    |
| `0x4006`    | Dashboard, backlight information                | `PSA_IDENT_DASHBOARD`           | GET  |                    |
| `0x4007`    | Trip computer information                       | `PSA_IDENT_TRIP`                | GET  |                    |
| `0x4008`    | Headunit settings, and audio information        | `PSA_IDENT_HEADUNIT`            | GET  |                    |
| `0x4009`    | Car status, CD changer / audio player command   | `PSA_IDENT_CAR_STATUS`          | GET  |                    |
| `0x4010`    | Radio CD player information                     | `PSA_IDENT_CD_PLAYER`           | GET  |                    |
| `0x4501`    | Send CD changer / audio player information      | `PSA_IDENT_SET_CD_CHANGER_DATA` | SET  |                    |
| `0x5000`    | RTC Time                                        | `PSA_IDENT_TIME`                | GET  | 7                  |
| `0x5100`    | ESP Temperature (Not accurate)                  | `PSA_IDENT_ESP32_TEMP`          | GET  |                    |

## Message formats

Because the packet structures change constantly, it is best to look at the header file from the ESP program's source.

## Message sending functions

Apart from the message format, this API supplies generic message sender functions. Those functions incorporate crc calculation and checking.

Only one instance of the API can be initialized at any one time.

Note: Only the Raspberry Pi side is supplied, as the ESP side is more integrated in the firmware.

### API enums

The following is a return value for all functions.

```c
enum psa_status
{
    PSA_OK = 0,

    PSA_INSTANCE_EXISTS, // An instance already exists.
                         // Only one instance can exist at any one time
    PSA_NO_INSTANCE,     // No instance exists

    PSA_NULL_ARG,    // One of the supplied arguments is NULL
    PSA_INVALID_ARG, // One of the supplied arguments is invalid

    PSA_READ_ERROR,  // An error occured while reading from the bus
    PSA_WRITE_ERROR, // An error occured while writing to the bus

    PSA_WRONG_PACKET, // Wrong packet has been received
    PSA_CRC_ERROR,    // The CRC checksum doesn't match

    PSA_GENERIC_ERROR, // An unknown error.
                       // This indicates that the API
                       // has most likely failed internally

    PSA_RESPONSE_CRC_ERROR,    // A CRC error from a SET message response
    PSA_RESPONSE_WRONG_PACKET, // A reponse for the wrong SET message
                               // has been received
    PSA_RESPONSE_PACKET_SIZE,  // The packet size for a SET message
                               // response is invalid
};
```

### Generic API functions

```c
/**
    * @brief Initializes the API. Returns `PSA_OK` on success
    *
    * @param[in] port The Serial port on which to communicate
    * @return enum psa_status
    */
enum psa_status psa_init(
    PSA_IN char const *const port);
```

```c
/**
    * @brief Closes the Serial port and cleans up after itself
    *
    * @return enum psa_status
    */
enum psa_status psa_close(PSA_NOARGS);
```

### Data API functions

These functions abstract the process of getting the data for some messages.

```c
/**
    * @brief Get the engine data
    *
    * @param[out] engine_data Buffer to store the data
    * @return enum psa_status
    */
enum psa_status psa_get_engine_data(
    PSA_OUT struct psa_engine_data *const engine_data);
```

```c
/**
    * @brief Get the radio data
    *
    * @param[out] radio_data Buffer to store the data
    * @return enum psa_status
    */
enum psa_status psa_get_radio_data(
    PSA_OUT struct psa_radio_data *const radio_data);
```

```c
/**
    * @brief Get the headunit data
    *
    * @param[out] headunit_data Buffer to store the data
    * @return enum psa_status
    */
enum psa_status psa_get_headunit_data(
    PSA_OUT struct psa_headunit_data *const headunit_data);
```

There are more in `psa_receive.h`. For the actual sender functions, check out `psa_receive.c`

In addition, there are functions for sending data to the ESP:

```c
/**
    * @brief Send CD Changer track information to the ESP
    * for display on the car's MFD
    *
    * @param[in] cdc_data Data to send
    * @return enum psa_status
    */
enum psa_status psa_set_cd_changer_data(
    PSA_IN struct psa_cd_changer_data const *const cdc_data);
```

### Minimal example

The following is a minimal example of querying a VIN number:

```c
#include <stdio.h>
#include <psa/psa_receive.h>

int main()
{
    enum psa_status status = PSA_OK;

    // API initialization

    status = psa_init("/dev/ttyUSB0");
    if (status != PSA_OK)
    {
        exit(EXIT_FAILURE); // An error occured
    }

    // Creating the struct to store the VIN
    psa_vin_data_t vin_data;

    status = psa_get_vin(&vin_data);

    if(status == PSA_OK)
    {
        // The VIN contains 17 ASCII characters,
        // and is not NULL-terminated
        printf("%17c\n", vin_data.vin);
    }

    status = psa_close();

    exit(EXIT_SUCCESS);
}
```
