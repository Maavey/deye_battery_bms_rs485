# Deye / LVESS Battery RS485 Protocol (Reverse Engineered)

Reverse engineered RS485 binary protocol for Deye / LVESS lithium batteries using ESP32 + Tasmota Berry.

---

# Serial Parameters

| Parameter        | Value   |
| ---------------- | ------- |
| Interface        | RS485   |
| Baudrate         | 9600    |
| Data Bits        | 8       |
| Parity           | None    |
| Stop Bits        | 1       |
| Mode             | Binary  |
| Frame Terminator | `0D 0A` |

---

# Frame Structure

```text
AA 55 [ADDR] [CMD] [LEN] [PAYLOAD...] [CRC_H] [CRC_L] 0D 0A
```

| Field         | Description             |
| ------------- | ----------------------- |
| `AA 55`       | Fixed packet header     |
| `ADDR`        | Battery address         |
| `CMD`         | Command / function code |
| `LEN`         | Payload length          |
| `PAYLOAD`     | Data payload            |
| `CRC_H CRC_L` | Checksum                |
| `0D 0A`       | Packet terminator       |

Confirmed battery address:

```text
01
```

---

# Confirmed Commands

| Command | Request              | Description                 |
| ------- | -------------------- | --------------------------- |
| `01`    | `AA5501010000200D0A` | Main battery information    |
| `02`    | `AA5501020000D00D0A` | Alarm / status block        |
| `03`    | `AA5501030001400D0A` | Operational limits / status |
| `04`    | `AA5501040003700D0A` | Cell voltage array          |
| `05`    | `AA5501050002E00D0A` | Extended / reserved block   |

---

# Command 01 — Main Battery Information

## Request

```text
AA5501010000200D0A
```

## Example Response

```text
AA5501013D0000033803E8480000000100064008FC00010F021B76200338031B0506104C5645535331353235383134423031004C5645535331354D4F5320202020011E680D0A
```

Payload begins after:

```text
AA 55 01 01 3D
```

## Decoded Fields

| Payload Offset |  Length | Description               | Example           | Parsed Value       |
| -------------- | ------: | ------------------------- | ----------------- | ------------------ |
| `2-3`          | 2 bytes | State of Charge (SOC)     | `03 38`           | `824 / 10 = 82.4%` |
| `4-5`          | 2 bytes | SOH / Capacity-like value | `03 E8`           | `1000 = 100.0`     |
| `22`           |  1 byte | Temperature               | `1B`              | `27°C`             |
| `31-46`        |   ASCII | Battery Serial Number     | `LVESS1525814B01` | Serial             |
| `48-61`        |   ASCII | Board / Model             | `LVESS15MOS`      | Model              |

---

# Command 02 — Alarm / Status Block

## Request

```text
AA5501020000D00D0A
```

## Example Response

```text
AA5501020C000000000000000000000000B90F0D0A
```

Payload length:

```text
0C = 12 bytes
```

Likely contains:

* alarm bits
* protection states
* operational status flags

Field mapping still in progress.

---

# Command 03 — Operational Limits / Status

## Request

```text
AA5501030001400D0A
```

## Example Response

```text
AA55010315440643020144434343444400000000000000000000812A0D0A
```

Payload length:

```text
15 = 21 bytes
```

Suspected fields:

* charge current limit
* discharge current limit
* operational profile
* status bytes

---

# Command 04 — Cell Voltage Array

## Request

```text
AA5501040003700D0A
```

## Example Response

```text
AA550104480D2B0F0D290100020D290D290D290D290D2A0D2B0D290D290D2B0D2B0D2B0D2B0D2B0D2B0D2B0D29000000000000000000000000000000000000000000000000000000000000000060470D0A
```

Payload begins after:

```text
AA 55 01 04 48
```

## Cell Voltage Format

Each cell voltage is stored as:

```text
16-bit Big Endian
Unit: millivolts (mV)
```

Examples:

| Raw Hex |  Parsed |
| ------- | ------: |
| `0D2B`  | 3371 mV |
| `0D29`  | 3369 mV |
| `0D2A`  | 3370 mV |

Total pack voltage from all 16 cells:

```text
≈ 53.9V
```

which matches the battery display (~54V).

---

# Command 05 — Extended / Reserved

## Request

```text
AA5501050002E00D0A
```

## Example Response

```text
AA550105400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000051950D0A
```

Mostly zero / reserved in tested firmware.

---

# ESP32 Tasmota Berry Test Code

```berry
var battery = serial(16, 17, 9600, serial.SERIAL_8N1)

def tx_frame(h)
  while battery.available() > 0
    battery.read()
  end

  print("TX HEX -> " + h)
  battery.write(bytes().fromhex(h))

  tasmota.delay(1200)

  if battery.available() > 0
    var rx_data = battery.read()
    print("RX HEX -> " + rx_data.tohex())
  else
    print("RX -> SILENCE")
  end
end
```

# Example Usage

```berry
tx_frame("AA5501010000200D0A")
tx_frame("AA5501020000D00D0A")
tx_frame("AA5501030001400D0A")
tx_frame("AA5501040003700D0A")
tx_frame("AA5501050002E00D0A")
```

---

# Additional Observed ASCII Protocol

An additional ASCII/Pylon-style protocol was also observed on the same battery interface.

## Request

```text
7E3230303234363432453030323032464433330D
```

ASCII:

```text
~20024642E00202FD33
```

## Response

```text
~200246000000FDB2
```

This appears to be an ACK-style protocol and was not used for telemetry extraction.

---

# Current Reverse Engineering Status

## Confirmed Working

* RS485 communication stable
* Binary `AA55` protocol working
* SOC decoding working
* Cell voltage decoding working
* Serial/model extraction working

## Remaining Work

* Decode remaining Command `01` fields
* Decode Command `02` alarm/status bits
* Decode Command `03` operational limits/status
* Build MQTT/Home Assistant automation
