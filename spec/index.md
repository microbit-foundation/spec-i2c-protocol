---
layout: page
title: I2C Protocol Specification
heading: I2C Protocol Specification
description: Specification for the micro:bit I2C Protocol
permalink: /software/spec-i2c-protocol/
ref: spec-i2c-protocol
lang: en
---

# micro:bit I2C Protocol Specification

This is version 2.00 of the specification.

- [Glossary](#glossary)
- [Versioning](#versioning)
- [Introduction](#introduction)
- [I2C Secondary addresses](#i2c-secondary-addresses)
- [I2C nRF - KL27 config/comms interface](#i2c-nrf--kl27-configcomms-interface)
- [I2C Flash interface](#i2c-flash-interface)
- [I2C HID Interface](#i2c-hid-interface)
- [Doc Updates](#doc-updates)


## Glossary

| Term          | Definition |
| ------------- | ---------- |
| DAPLink       | [Interface firmware](https://github.com/ARMmbed/DAPLink) providing USB and programming capabilities |
| I2C           | [Inter-Integrated Circuit](https://en.wikipedia.org/wiki/I%C2%B2C) bus |
| I2C main      | I2C node in control of the clock and initiating transactions |
| I2C secondary | I2C peripheral that responds to the I2C main |
| Storage       | Flash available in the KL27 for micro:bit data and config storage |
| SWD           | [Serial Wire Debug](https://developer.arm.com/architectures/cpu-architecture/debug-visibility-and-trace/coresight-architecture/serial-wire-debug) |
| UART          | [Universal asynchronous receiver-transmitter](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) |


## Versioning

The version follows a `major.minor` configuration.

- `major`: Indicates a change or addition to the procotol, this is usually accompanied by a new DAPLink release.
- `minor`: Indicates an update, fix, or clarification to the documentation **only**. No changes required to the protocol implementations.

The `I2C protocol version` property returns only the major version.


## Introduction

The micro:bit contains two microcontrollers, the Interface MCU which provides the USB functionality, and the Target MCU where the user code runs.
More information can be found in the [Tech Site DAPLink page](https://tech.microbit.org/software/daplink-interface/).

In micro:bit V1 there are UART and SWD signals connecting the Interface MCU (KL26) and the Target MCU (nRF51). These are used to program the Target MCU (nRF51) and to provide serial communication between the Target (nRF51) and the computer.

The micro:bit V2 adds an internal I2C bus connected to the Interface MCU (KL27), the Target MCU (nRF52), and the motion sensors (in V1 the motions sensors are connected to the external I2C bus, connected only to the Target MCU (nRF51), more info in the [Tech Site I2C page](https://tech.microbit.org/hardware/i2c-shared/)).
This new I2C bus allows the Interface (KL27) to provide additional features to the  Target (nRF52), and to co-operate to set the board into different power modes (more info in the [Power Management Spec](https://github.com/microbit-foundation/spec-power-management/)).

![I2C Diagram](https://tech.microbit.org/docs/software/spec-i2c-protocol/spec/img/i2c-diagram.png)

The additional features provided by the Interface (KL27) via I2C are:
- Device Information
    - Board ID, DAPLink version, and more
- Power Management
    - As defined in the [Power Management Spec](https://github.com/microbit-foundation/spec-power-management/)
- I2C Flash Storage
    - The Interface (KL27) flash is 256 KBs, where 128KBs are reserved for non-volatile storage accessible to the Target (nRF52)


## I2C Secondary addresses

| I2C Secondary                                | 7-bit address |
| -------------------------------------------- | ------------- |
| KL27 (I2C nRF – KL27 config/comms interface) | 0x70          |
| KL27 (I2C USB/HID interface)                 | 0x71          |
| KL27 (I2C Flash interface)                   | 0x72          |
| FXOS8700CQ                                   | 0x1F          |
| LSM303AGR Accelerometer                      | 0x19          |
| LSM303AGR Magnetometer                       | 0x1E          |


## I2C nRF – KL27 config/comms interface

### Types of commands

| Command          | CMD ID | Used by          |
| ---------------- | ------ | ---------------- |
| `nop_cmd`        | 0x00   | main only        |
| `read_request`   | 0x10   | main only        |
| `read_response`  | 0x11   | secondary only   |
| `write_request`  | 0x12   | main & secondary |
| `write_response` | 0x13   | main & secondary |
| `error_response` | 0x20   | main & secondary |

### Packet format for each command

#### `nop_cmd`

|             |
| ----------- |
| CMD ID (1B) |

#### `read_request`

|             |                  |
| ----------- | ---------------- |
| CMD ID (1B) | Property ID (1B) |

#### `read_response`

|             |                  |                |           |
| ----------- | ---------------- | -------------- | --------- |
| CMD ID (1B) | Property ID (1B) | Data size (1B) | Data (NB) |

#### `write_request`

|             |                  |                |           |
| ----------- | ---------------- | -------------- | --------- |
| CMD ID (1B) | Property ID (1B) | Data size (1B) | Data (NB) |

#### `write_response`

|             |                  |
| ----------- | ---------------- |
| CMD ID (1B) | Property ID (1B) |

#### `error_response`

|             |                 |
| ----------- | --------------- |
| CMD ID (1B) | Error code (1B) |

### Properties

<table>
<thead>
<tr class="header">
<th>Property</th>
<th>Property ID</th>
<th>Data</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>DAPLink Board version (R)</td>
<td>0x01</td>
<td>Size: 2B e.g. 0x9904</td>
</tr>
<tr class="even">
<td>I2C protocol version (R)</td>
<td>0x02</td>
<td>
Size: 2B e.g. 0x0001<br />
Value only includes major version
</td>
</tr>
<tr class="odd">
<td>DAPLink version (R)</td>
<td>0x03</td>
<td>Size 2B e.g. 0x00FD</td>
</tr>
<tr class="even">
<td>Power state (R)</td>
<td>0x04</td>
<td>Size 1B e.g. 0x01<br />
<pre>typedef enum {
    PWR_SOURCE_NONE = 0,
    PWR_USB_ONLY = 0b01,
    PWR_BATT_ONLY = 0b10,
    PWR_USB_AND_BATT = 0b11  // (*)
} power_source_t;</pre>
* On USB power the battery reading (bit 1) might not be correct due to a hardware bug
</td>
</tr>
<tr class="odd">
<td>Power consumption (R)</td>
<td>0x05</td>
<td>Size 8B <br />
<pre>4B for bat_sense voltage
4B for vin voltage
(units in microvolts uint32)</pre></td>
</tr>
<tr class="even">
<td>USB enumeration state (R)</td>
<td>0x06</td>
<td>Size 1B e.g. 0x02<br />
<pre>typedef enum main_usb_connect {
    USB_DISCONNECTED,
    USB_CONNECTING,
    USB_CONNECTED,
    USB_CHECK_CONNECTED,
    USB_CONFIGURED,
    USB_DISCONNECTING
} main_usb_connect_t;</pre></td>
</tr>
<tr class="odd">
<td>KL27 Power mode (W)</td>
<td>0x07</td>
<td>Size 1B e.g. 0x08<br />
<pre>kAPP_PowerModeVlls0 = 0x08</pre></td>
</tr>
<tr class="even">
<td>Power LED Sleep state (W)</td>
<td>0x08</td>
<td>Size 1B e.g. 0x01<br />
<pre>0x00 -> OFF
0x01-0xFF -> ON</pre></td>
</tr>
<tr class="odd">
<td>User event (R asynch)</td>
<td>0x09</td>
<td>Size 1B e.g. 0x01<br />
<pre>0x01 -> Wakeup from reset button (not used)
0x02 -> Wakeup from WAKE_ON_EDGE
0x03 -> reset button long press</pre></td>
</tr>
<tr class="even">
<td>Automatic sleep (W)</td>
<td>0x0A</td>
<td>Size 1B e.g. 0x01<br />
<pre>0x00 -> OFF
0x01-0xFF -> ON</pre></td>
</tr>
</tbody>
</table>



#### Error codes

| Error                                               | Error Code |
| --------------------------------------------------- | ---------- |
| Success                                             | 0x30       |
| Incomplete command                                  | 0x31       |
| Unknown command                                     | 0x32       |
| Command disallowed                                  | 0x33       |
| Unknown property (for both read and write requests) | 0x34       |
| Wrong size for property (for write requests)        | 0x35       |
| Read property disallowed                            | 0x36       |
| Write property disallowed                           | 0x37       |
| Write fail                                          | 0x38       |

### Examples

- Read DAPLink Board version (nRF I2C main)
  1. `read_request` (cmd id + property) I2C Write: 0x10 0x01
  2. (KL27 processes cmd and asserts `COMBINED_SENSOR_INT` signal when response is ready)
  3. `read_response` (cmd id + property + size + data) I2C Read: 0x11 0x01 0x02 0x04 0x99
  4. (KL27 releases `COMBINED_SENSOR_INT` signal)

### Considerations

- `read_request` can only be sent by the I2C main (nrf)
    - The main (nRF) must wait for the `COMBINED_SENSOR_INT` signal to be asserted by the secondary (KL27)
- `write_request` can be sent by both secondary and main.
    - For the secondary to initiate this, it must assert the interrupt signal first and then the main must poll (i2c read) the device for data.


## I2C Flash interface

KL27 storage memory layout:

```
↓ KL27 flash address 0x20000                                ↓ KL27 flash address0x40000
↓         ↓ KL27 flash address 0x20400                      ↓
┌---------┬-------------------------------------------------┐
| KL27    | storage data                                    |
| config  |[ file.ext -------------------- ][ DAL’s config ]|
└---------┴-------------------------------------------------┘
          ↑ storage address 0x0000          ↑ address set by file size
                 ↑ base64 start address (anywhere inside file.ext range)
```

- config
    - Controlled by KL27
    - nRF reads/write data fields via I2C commands
- data
    - Controlled by nRF
    - nRF reads/write bytes via I2C commands
    - Storage address range: 0x00000-max_size

### Protocol

- Word (4 Bytes) align transactions
- All transactions start with:
    - 1 byte command ID
    - N byte data depending on command
- For KL27 responses, KL27 to hold interrupt until success/error messages have been served  

### Commands

- Read/Write config data (stored in RAM)
  - File name (includes extension)  (cmd id `0x01`)
    - 11B uppercase characters following 8.3 format (e.g. `"DATA    BIN"`)
    - default → DATA.BIN
  - File size (from start of data section)  (`0x02`)
    - 4B size in bytes (MSB First)
    - default → 129024 (126 KBs)
    - max size is 126 KBs (128KB - 1KB for flash interface config and 1KB for DAL's config)
  - Base64 encoding start address (from start of data section)  (`0x09`)
    - 4B size in bytes (MSB First)
    - default → 129024 (end of file address)
    - max value is the end of file adress
  - Enable file visible in MSD  (`0x03`)
    - 1B visibility
    - default → false
- Write config data to flash    (`0x04`)
    - Handles sector erase if RAM config data is different than Flash config data
- Erase all config  (`0x05`)
    - Config data will be in its default state
- Get available storage size    (`0x06`)
  - 1B size in KB
- Get flash sector size (`0x07`)
  - 2B sector size in bytes (MSB first)
- Re-mount MSD  (`0x08`)


### Flash Operations

#### Reading/writing config data
- For writing the config data (file name, file size and file visibility), the I2C main has to send the 1B command ID followed by the corresponding config data. If the data size is unexpected, an error will be returned in the I2C response.
- The I2C Response after a successful write operation will contain the corresponding command ID followed by the written config data which can be used as confirmation by the I2C main.
- For reading the config data, the I2C main has to send only the 1B command ID with no further data. The I2C response will then contain the config data.

#### Write storage data

- 1 byte write command  (cmd id `0x0B`)
- 3 bytes address  (MSB First)
- 4 bytes for length  (MSB First)
- data to write
- Address and size check
    - Size has to be a multiple of 4 bytes
    - Address has to be aligned to a 4 byte boundary
    - Bound check
- KL27 can clock stretch

#### Read storage data

- 1 bytes read command  (`0x0A`)
- 3 bytes address
- 4 bytes for length

#### Erase storage data

- 1 bytes erase command (`0x0C`)
- 3 bytes start address
- 1 unused
- 3 byte end address (inclusive)
    - For a single sector erase Start address == End address
- Address checks
    - Addresses have to be sector align
    - End address >= Start address
    - Bound check
- On errors KL27 to trigger interrupt and send error code
- On success KL27 to trigger interrupt and send success message
- KL to provide some kind of status read command

### KL27 behaviour

- Before enumeration it checks if it should show the file on the MSD
    - It sets the file name from the config
    - It sets the file size from the config
- KL27 I2C buffer size: 1KB + 4 bytes
- Storage writes should not trigger "hidden" sector erases, the nRF is
  responsible to write and erase
- When writing to the config data first check if the data to be written is
  different than present, avoid an erase-and-write operation if it's the same
- USB should have higher priority than any I2C transaction

### Examples

#### Storage Write
- Write 4 bytes "1234" to storage address 0x000010
```
  I2C Write:  [0x72,    0x0B,  0x00,0x00,0x10,  0x00,0x00,0x00,0x04,  0x31,0x32,0x33,0x34]
       I2C Addr ↑  cmd id ↑    address ↑        length ↑              data ↑
  I2C Read:   [0x72,    0x0B,  0x00,0x00,0x10,  0x00,0x00,0x00,0x04,  0x31,0x32,0x33,0x34]
       I2C Addr ↑  cmd id ↑    address ↑        length ↑              data ↑
```
  
#### Storage Read
- Read 4 bytes at storage address 0x000010
```
  I2C Write:  [0x72,    0x0A,  0x00,0x00,0x10,  0x00,0x00,0x00,0x04]
       I2C Addr ↑  cmd id ↑    address ↑        length ↑
  I2C Read:   [0x72,    0x0A,  0x00,0x00,0x10,  0x00,0x00,0x00,0x04,  0x31,0x32,0x33,0x34]
       I2C Addr ↑  cmd id ↑    address ↑        length ↑              data ↑
```

#### Config Update, enable file visibility and remount MSD drive
- Change filename to LOG.TXT
```
  I2C Write:  [0x72,    0x01,   0x4c,0x4f,0x47,0x20,0x20,0x20,0x20,0x20,0x54,0x58,0x54]
       I2C Addr ↑  cmd id ↑    filename "LOG     TXT" ↑
  I2C Read:   [0x72,    0x01,   0x4c,0x4f,0x47,0x20,0x20,0x20,0x20,0x20,0x54,0x58,0x54]
       I2C Addr ↑  cmd id ↑    filename "LOG     TXT" ↑
```
- Enable file visibility
```
  I2C Write:  [0x72,    0x03,           0x01]
       I2C Addr ↑  cmd id ↑    visibility ↑
  I2C Read:   [0x72,    0x03,           0x01]
       I2C Addr ↑  cmd id ↑    visibility ↑
```
- Remount MSD
```
  I2C Write:  [0x72,    0x08]
       I2C Addr ↑  cmd id ↑
  I2C Read:   [0x72,    0x08]
       I2C Addr ↑  cmd id ↑
```

### Universal Hex

KL27 storage area should be writeable via Universal Hex.

This is not yet implemented.


## I2C HID Interface

All I2C features should be available via HID interface, so that it can be accessed via WebUSB.

This is not yet implemented.


## Doc Updates

| Version | Changes |
|---------|---------|
| 1.00    | Initial release, as implemented in DAPLink 0255 |
| 1.01    | Add note to "Power state" property about the hardware issue detecting battery power when USB power is present. |
| 2.00    |         |
