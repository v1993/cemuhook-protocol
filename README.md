# Introduction

Cemuhook is modification for Cemu WiiU emulator which allow to do all sorts of cool stuff, including custom button/motion sources.  
Purpose of this document is to shed light on previously undocumented protocol it uses.  
**Only one known and existing version of protocol at the moment of creation of this document is `1001` which is described below.**

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

## Information about connected controllers

This request type is a bit more complicated. When you recieve it, you should report all controllers you serve (up to four).  
This message have length of 12 bytes (32 total with structure from above).  
If controller for some port is not connected, you can respond with 12 zero bytes.

### Incoming packet structure

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0 | 4 | **Signed 32-bit** | Amount of ports you should report about. Always less than 5. |
| 4 | 1 to 4 | Unsigned 8-bit (array) | Each byte represent number of slot you should report about. Count of bytes here is determined by value above. Each value is less than 4. |

For every requested controller slot you should send one packet structured like described below.

### Outgoing packet structure:

| Offset | Length | Type | Meaning |
| ------ | ------ | ---- | ------- |
| 0  | 1  | Unsigned 8-bit | Slot you're reporting about. Must be the same as byte value you read. |
| 1  | 1  | Unsigned 8-bit | Slot state: `0` if not connected, `1` if reserved (?), `2` if connected. |
| 2  | 1  | Unsigned 8-bit | Device model: `0` if not applicable, `1` if no or partial gyro `2` for full gyro. Value `3` exist but should not be used (RIP Half-Life series). |
| 3  | 1  | Unsigned 8-bit | Connection type: `0` if not applicable, `1` for USB, `2` for bluetooth. |
| 4  | 6 | Unsigned 48-bit | MAC address of device. It's used to detect same device between launches. Zero out if not applicable. |
| 10 | 1 | Unsigned 8-bit | Battery status. See below for possible values. |
| 11 | 1 | Unsigned 8-bit | One zero byte. |

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

# Actual controllers data

To be documented.
