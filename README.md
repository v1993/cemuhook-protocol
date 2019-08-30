# Introduction

Cemuhook is modification for Cemu WiiU emulator which allow to do all sorts of cool stuff, including custom button/motion sources.
Purpose of this document is to shed light on previously undocumented protocol it uses.

## Common information

* Cemuhook use UDP protocol, which may make things tricky but very efficient
* Cemuhook use server at localhost:26760 (number is a port)
* Your application should serve all sources it can, not few different instances.
* Each (valid) packet either from cemuhook or your server contain:
* * Header (16 bytes)
* * Message type (4 bytes)
* * Other data

## Numbers encoding
You can refer to C# implementations of BitConverter for details on this.  
Also, you can take a look at this Lua 5.3 code which should give you overall impression:

```Lua
local function num2bytestring(num, len)
	local arr = {}
	for i=1,len do
		arr[i] = (num >> ((i-1) * 8)) & 0xFF
	end
	return string.char(table.unpack(arr))
end

local function bytestring2num(str)
	local arr = {str:byte(1, -1)}
	local num = arr[1]
	for i=2,#str do
		num = (arr[i] << ((i-1) * 8)) | num
	end
	return num
end
```

# Header stucture

| First byte position | Last byte position | Meaning |
| ------------------- | ------------------ | ------- |
| 1  | 4  | Magic string — `DSUS` if it's message by server (you), `DSUC` if by client (cemuhook). |
| 5  | 6  | Protocol version used in message (unsigned). Currently `1001`. |
| 7  | 8  | Length of packet without header (unsigned). Drop packet if it's too short, truncate if it's too long. |
| 9  | 12 | CRC32 of whole packet while this field was zeroed. You may need to reverse one depending on your language implementation. |
| 13 | 16 | Client or server ID (unsigned). Should stay the same on one run. Can be randomly generated on startup. |
| 17 | 20 | **Not actually part of header so it counts as length.** Event type (unsigned). Read below for possible ones. |

Below I'll refer to all positions **after cutting out first 20 bytes and impying that structure above is included in your response**.

# Message types

| Value | Meaning |
| ----- | ------- |
| `0x100000` | Protocol version information (doesn't seem to be ever requested) |
| `0x100001` | Information about connected controllers |
| `0x100002` | Actual controllers data |

Same constants are used both for incoming and outgoing messages. So if you got message with type, response(s) to it will contain same constant.

## Protocol version information

This message type carry no extra payload and require you to reply with maximal protocol version supported.

### Incoming packet structure

| First byte position | Last byte position | Meaning |
| ------------------- | ------------------ | ------- |
| - | - | No additional payload. |

### Outgoing packet structure:

| First byte position | Last byte position | Meaning |
| ------------------- | ------------------ | ------- |
| 1 | 2 | Maximal protocol version supported by your application. |

## Information about connected controllers

This request type is more complicated. When you recieve it, you should report all controllers you serve (up to four).  
This message have length of 12 bytes (32 total with structure from above).  
If controller for some port is not connected, you can respond with 12 zero bytes.

### Incoming packet structure

| First byte position | Last byte position | Meaning |
| ------------------- | ------------------ | ------- |
| 1 | 4 | Amount of ports you should report about. Always less than 5. |
| 5 | up to 8 | Each byte represent number of slot you should report about. Count of bytes here is determined by value above. Each value is less than 4. |

For every requested controller slot you should send one packet structured like described below.

### Outgoing packet structure:

| First byte position | Last byte position | Meaning |
| ------------------- | ------------------ | ------- |
| 1  | 1  | Slot you're reporting about. Must be the same as byte value you read. |
| 2  | 2  | Slot state: `0` if not connected, `1` if reserved (?), `2` if connected. |
| 3  | 3  | Device model. No idea about values. |
| 4  | 4  | Connection type: `0` if not applicable, `1` for USB, `2` for bluetooth. |
| 5  | 10 | MAC address of device. It's used to detect same device between launches. Zero out if not applicable. |
| 11 | 11 | Battery status. See below for possible values. |
| 12 | 12 | One zero byte. |

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
