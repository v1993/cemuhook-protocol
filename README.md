# Introduction

Cemuhook is modification for Cemu WiiU emulator which allow to do all sorts of cool stuff, including custom button/motion sources.  
Purpose of this document is to shed light on previously undocumented protocol it uses.  
**Only one known and existing version of protocol at the moment of creation and last modification of this document is `1001` which is described below.**

## Common information

* Cemuhook use UDP protocol, which may make things tricky but very efficient
* Cemuhook use server at `localhost:26760` (number is a port)
* Your application should serve all sources it can, not few different instances of it.
* Each (valid) packet either from cemuhook or your server contain:
* * Header (16 bytes)
* * Message type (4 bytes)
* * Other data

## Numbers encoding
You can refer to C# implementations of BitConverter for details on this.  
Numbers are encoded as little endian, which is also native endian on most platforms in the world (but not all).

# Header stucture

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0  | 4  | String | Magic string — `DSUS` if it's message by server (you), `DSUC` if by client (cemuhook). |
| 4  | 2  | Unsigned 16-bit | Protocol version used in message. Currently `1001`. |
| 6  | 2  | Unsigned 16-bit | Length of packet without header. Drop packet if it's too short, truncate if it's too long. |
| 8  | 4 | Unsigned 32-bit | CRC32 of whole packet while this field was zeroed out. Be careful with endianness here! |
| 12 | 4 | Unsigned 32-bit | Client or server ID who sent this packet. Should stay the same on one run. Can be randomly generated on startup. |
| 16 | 4 | Unsigned 32-bit | **Not actually part of header so it counts as length.** Event type. Read below to learn possible ones. |

Below I'll refer to all positions **after cutting out first 20 bytes and impying that structure above is included in your response at the beginning**.

# Message types

| Value | Meaning |
| ----- | ------- |
| `0x100000` | Protocol version information (doesn't seem to be ever requested) |
| `0x100001` | Information about connected controllers |
| `0x100002` | Actual controllers data |

Same constants are used both for incoming and outgoing messages. So if you got message with some type, response(s) to it will have same type value.

## Protocol version information

This message type carry no extra incoming payload and require you to reply with maximal protocol version supported.

### Incoming packet structure

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| - | - | - | No additional payload. |

### Outgoing packet structure:

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0 | 2 | Unsigned 16-bit | Maximal protocol version supported by your application. |

## Shared response beginning for message types below

Both "Information about connected controllers" and "Actual controllers data" responses begin the same way that is described below:

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0  | 1  | Unsigned 8-bit | Slot you're reporting about. Must be the same as byte value you read. |
| 1  | 1  | Unsigned 8-bit | Slot state: `0` if not connected, `1` if reserved (?), `2` if connected. |
| 2  | 1  | Unsigned 8-bit | Device model: `0` if not applicable, `1` if no or partial gyro `2` for full gyro. Value `3` exist but should not be used (go with VR, guys). |
| 3  | 1  | Unsigned 8-bit | Connection type: `0` if not applicable, `1` for USB, `2` for bluetooth. |
| 4  | 6 | **Unsigned 48-bit** | MAC address of device. It's used to detect same device between launches. Zero out if not applicable. |
| 10 | 1 | Unsigned 8-bit | Battery status. See below for possible values. |

This structure is 11 bytes long.

### Battery values

| Value  | Meaning |
| ------ | ------- |
| `0x00` | Not applicable |
| `0x01` | Dying |
| `0x02` | Low |
| `0x03` | Medium |
| `0x04` | High |
| `0x05` | Full (or almost) |
| `0xEE` | Charging |
| `0xEF` | Charged |

## Information about connected controllers

This request type is a bit more complicated. When you recieve it, you should report all controllers you serve (up to four).  
This message have length of 12 bytes (32 total with structure from above).  
If controller for some port is not connected, you can respond with 12 zero bytes (plus header).

### Incoming packet structure

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0 | 4 | **Signed 32-bit** | Amount of ports you should report about. Always less than 5. |
| 4 | 1 to 4 | Unsigned 8-bit (array) | Each byte represent number of slot you should report about. Count of bytes here is determined by value above. Each value is less than 4. |

For every requested controller slot you should send one packet structured like described below.

### Outgoing packet structure:

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0  | 11  | Complex | Beginning described above |
| 11 | 1 | N/A | Zero byte (`\0`). |

# Actual controllers data

When you get this type of message, no instant response is required. Instead, you should begin or continue sending data from requested controller to this server. There is no way to know when client is gone, so you should implement some kind of timeout to stop sending data when client didn't requested data for some time.

If requested controller is not connected, message should be ignored.

Total message length (*with packet header*) is 100 bytes.

### Incoming packet structure

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0 | 1 | Unsigned 8-bit | Bitmask of actions you should take. Valid flags are `1` for slot-based registration, `2` for MAC-based registration, no bits (all set to `0`) to subscribe to all controllers. |
| 1 | 1 | Unsigned 8-bit | If slot-based registration is requested, slot to report about. |
| 2 | 6 | **Unsigned 48-bit** | If MAC-based registration is requested, MAC of device to report about. |

### Outgoing packet structure:

Bitmasks are described in descending order:, as bits in number go: `128`, `64`, `32` and so on.  
All sticks use full 8-bit range (0-255). Set them to `128` for neutral value.  
Analog buttons use full range as well. If button is not analog, report `0` for released and `255` for pressed states.  
Acceleration values are in g's (1 g ≈ 9.8 m/s²), gyroscope ones are in deg/s. 
There is no standard range for touch values aside from requirement to have x axis point rightward and y to point downward. Clients interested in using touch input should implement some sort of calibration method to determine touch range.

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0  | 11 | Complex | Beginning described above |
| 11 | 1 | Unsigned 8-bit | Is controller connected (`1` if connected, `0` if not) |
| 12 | 4 | Unsigned 32-bit | Packet number (for this client) |
| 16 | 1 | Bitmask | D-Pad Left, D-Pad Down, D-Pad Right, D-Pad Up, Options (?), R3, L3, Share (?) |
| 17 | 1 | Bitmask | Y, B, A, X, R1, L1, R2, L2 |
| 18 | 1 | Unsigned 8-bit | PS Button (unused) |
| 19 | 1 | Unsigned 8-bit | Touch Button (unused) |
| 20 | 1 | Unsigned 8-bit | Left stick X (plus rightward) |
| 21 | 1 | Unsigned 8-bit | Left stick Y **(plus upward)** |
| 22 | 1 | Unsigned 8-bit | Right stick X (plus rightward) |
| 23 | 1 | Unsigned 8-bit | Right stick Y **(plus upward)** |
| 24 | 1 | Unsigned 8-bit | Analog D-Pad Left |
| 25 | 1 | Unsigned 8-bit | Analog D-Pad Down |
| 26 | 1 | Unsigned 8-bit | Analog D-Pad Right |
| 27 | 1 | Unsigned 8-bit | Analog D-Pad Up |
| 28 | 1 | Unsigned 8-bit | Analog Y |
| 29 | 1 | Unsigned 8-bit | Analog B |
| 30 | 1 | Unsigned 8-bit | Analog A |
| 31 | 1 | Unsigned 8-bit | Analog X |
| 32 | 1 | Unsigned 8-bit | Analog R1 |
| 33 | 1 | Unsigned 8-bit | Analog L1 |
| 34 | 1 | Unsigned 8-bit | Analog R2 |
| 35 | 1 | Unsigned 8-bit | Analog L2 |
| 36 | 6 | Complex | First touch (see below) |
| 42 | 6 | Complex | Second touch (see below) |
| 48 | 8 | Unsigned 64-bit | Motion data timestamp in microseconds, update only with accelerometer (but not gyro only) changes |
| 56 | 4 | Float | Accelerometer X axis |
| 60 | 4 | Float | Accelerometer Y axis |
| 64 | 4 | Float | Accelerometer Z axis |
| 68 | 4 | Float | Gyroscope pitch |
| 72 | 4 | Float | Gyroscope yaw |
| 76 | 4 | Float | Gyroscope roll |

#### Touch data structure

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0 | 1 | Unsigned 8-bit | Is touch active (`1` if active, else `0`) |
| 1 | 1 | Unsigned 8-bit | Touch id (should be the same for one continious touch) |
| 2 | 2 | Unsigned 16-bit | Touch X position |
| 4 | 2 | Unsigned 16-bit | Touch Y position |
