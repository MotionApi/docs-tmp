# MA Binary Protocol — Wire Format Reference

This document specifies the **MA Protocol** (MotionApi binary protocol), the on-the-wire frame format that the tracker uses to publish GPS, status, IMU, and MARK data to the broker on the `bin` topic. It's UBX-inspired (uBlox GNSS protocol family) and is what you receive byte-for-byte when you call any `/v1` read endpoint with `format=raw`.

> **You usually don't need this.** The `/v1` API gives you `format=formatted` by default — fully decoded JSON in SI units. Use `format=raw` (and this document) only when you want to write your own decoder, archive bytes exactly as transmitted, or debug protocol-level issues.

---

## Why binary

Standard MQTT JSON payloads are ~250 B per GPS fix and ~500 B per status frame. Over LTE-M with a 1024 B MQTT cap and SIM data budgets that matter, that's a lot of overhead. The binary protocol gives ~90 % reduction: a complete GPS fix fits in 24 B, a full status frame in 32 B. That makes batching practical — a 10-second logging-mode burst of 100 positions is ~3 packets instead of dozens.

The tracker uses this format on `t/{ICCID}/bin`. Standard modes (1–6) also still publish JSON on `t/{ICCID}/gps` for backwards compatibility; logging modes (20–28) only use `bin`.

---

## Frame layout

Every binary message is a self-contained **frame**:

```
+--------+--------+----------+-----------+----------+--------+--------+
| SYNC_1 | SYNC_2 | MSG_TYPE |  LENGTH   | PAYLOAD  |  CK_A  |  CK_B  |
| 0x4D   | 0x41   |  u8      |  u16 LE   |  N bytes |  u8    |  u8    |
+--------+--------+----------+-----------+----------+--------+--------+
|<-- 1 -->|<-- 1 -->|<-- 1 -->|<-- 2 ---->|<-- N --->|<-- 1 -->|<-- 1 -->|

Total size = 7 + N bytes (5-byte header + payload + 2-byte checksum)
```

| Field      | Size | Description                                                                |
|------------|------|----------------------------------------------------------------------------|
| `SYNC_1`   | 1    | `0x4D` (`'M'`)                                                             |
| `SYNC_2`   | 1    | `0x41` (`'A'`)                                                             |
| `MSG_TYPE` | 1    | Message type — see [Message types](#message-types) below                   |
| `LENGTH`   | 2    | Payload length in bytes, **little-endian unsigned 16-bit**                 |
| `PAYLOAD`  | N    | Body — interpretation depends on `MSG_TYPE`                                |
| `CK_A`     | 1    | Fletcher-8 checksum A                                                      |
| `CK_B`     | 1    | Fletcher-8 checksum B                                                      |

**Endianness:** every multi-byte integer in this protocol is **little-endian**.

**Constants:**

| Constant                  | Value | Meaning                                                            |
|---------------------------|-------|--------------------------------------------------------------------|
| `MA_HEADER_SIZE`          | 5     | `SYNC_1 + SYNC_2 + MSG_TYPE + LENGTH`                              |
| `MA_CHECKSUM_SIZE`        | 2     | `CK_A + CK_B`                                                      |
| `MA_FRAME_OVERHEAD`       | 7     | Header + checksum                                                  |
| `MQTT_MAX_PAYLOAD_BYTES`  | 1024  | Hard limit — uBlox SARA-R422 `AT+UMQTTC=9` MQTT payload max        |
| `GPS_BATCH_MAX_COUNT`     | 42    | `(1024 − 7 overhead − 1 count) / 24 B per record`                  |

---

## Fletcher-8 checksum

Identical to the UBX Fletcher-8 algorithm. The checksum covers **MSG_TYPE + LENGTH + PAYLOAD** — i.e. every byte between the two SYNC bytes and the two checksum bytes.

```python
def fletcher8(data: bytes) -> tuple[int, int]:
    ck_a = 0
    ck_b = 0
    for b in data:
        ck_a = (ck_a + b) & 0xFF
        ck_b = (ck_b + ck_a) & 0xFF
    return ck_a, ck_b
```

Reference C implementation lives in firmware [`main/binary_protocol.h`](https://github.com/MotionApi/example-api-python-logger/blob/main/motionapi_logger.py) and TypeScript in `packages/shared/constants/protocol.ts` (private). Any UBX Fletcher-8 implementation will work — the algorithm is byte-identical.

---

## Message types

| Constant       | Value | Topic mapping     | Payload                                              |
|----------------|-------|-------------------|------------------------------------------------------|
| `MA_MSG_GPS`   | 0x01  | `t/{ICCID}/bin`   | `[count:u8][gps_record_t × count]` (batched)         |
| `MA_MSG_STATUS`| 0x02  | `t/{ICCID}/bin`   | One `status_record_t` (32 B)                         |
| `MA_MSG_IMU`   | 0x03  | `t/{ICCID}/bin`   | `[base_ts:u32][base_ts_ms:u16][imu_sample_t × N]`    |
| `MA_MSG_MARK`  | 0x04  | `t/{ICCID}/bin`   | One `gps_record_t` (24 B) — QoS 1, deduped server-side |

Unknown `MSG_TYPE` values: a decoder should validate the checksum, skip the frame, and continue scanning the buffer for the next `0x4D 0x41` sync pair.

---

## Payload — GPS record (`gps_record_t`)

24 bytes, packed, little-endian. Used for **single positions** (`MARK`) and inside the **GPS batch** payload.

| Offset | Field          | Type   | Size | Unit              | Notes                                          |
|--------|----------------|--------|------|-------------------|------------------------------------------------|
| 0      | `timestamp_s`  | u32 LE | 4    | Unix epoch seconds (UTC) | From UBX NAV-PVT date/time fields        |
| 4      | `timestamp_ms` | u16 LE | 2    | milliseconds (0–999)     |                                            |
| 6      | `lat`          | i32 LE | 4    | degrees × 1e7     | Direct from NAV-PVT (e.g. `522277620` = 52.2277620°) |
| 10     | `lon`          | i32 LE | 4    | degrees × 1e7     |                                                |
| 14     | `speed_3d`     | u16 LE | 2    | cm/s              | Max 655.35 m/s ≈ 2359 km/h                     |
| 16     | `hMSL`         | i32 LE | 4    | millimeters above mean sea level | Direct from NAV-PVT             |
| 20     | `hAcc`         | u32 LE | 4    | millimeters (horizontal accuracy estimate) |                       |

**Decode example (Python):**

```python
import struct
GPS_STRUCT = struct.Struct("<IHiiHiI")  # 24 bytes, little-endian

ts_s, ts_ms, lat_e7, lon_e7, speed_cms, hmsl_mm, h_acc_mm = GPS_STRUCT.unpack(payload[:24])
lat_deg = lat_e7 / 1e7
lon_deg = lon_e7 / 1e7
speed_ms = speed_cms / 100
altitude_m = hmsl_mm / 1000
h_acc_m = h_acc_mm / 1000
```

### GPS batch payload (`MSG_TYPE = 0x01`)

The `bin` topic only ever publishes GPS in **batched** form, even for a single position. The payload prefix is one byte: the number of records, followed by that many `gps_record_t` back-to-back.

```
+-------+-----------------+-----------------+---     ---+-----------------+
| count |  gps_record #1  |  gps_record #2  |   ...    |  gps_record #N  |
| u8    |   24 bytes      |   24 bytes      |          |   24 bytes      |
+-------+-----------------+-----------------+---     ---+-----------------+

Total payload size = 1 + (count × 24)
Max count per frame = 42 (because 1 + 42×24 = 1009 ≤ 1017 = 1024 − 7 overhead)
```

A 100-position logging burst (mode `Log 10s`) gets split into **3 back-to-back frames** of 42 + 42 + 16 positions. The server reassembles and emits them as one `gps_batch` packet with `data: HumanGps[]`.

---

## Payload — Status record (`status_record_t`)

32 bytes, packed, little-endian. Sent on every `STATUS` interval (default 30 s — see [§ Device modes](README.md#device-modes)).

| Offset | Field            | Type   | Size | Unit                                            |
|--------|------------------|--------|------|-------------------------------------------------|
| 0      | `timestamp_s`    | u32 LE | 4    | Unix epoch seconds (UTC)                        |
| 4      | `timestamp_ms`   | u16 LE | 2    | milliseconds (0–999)                            |
| 6      | `battery_mv`     | u16 LE | 2    | millivolts                                      |
| 8      | `rssi`           | i8     | 1    | dBm                                             |
| 9      | `rsrp`           | i16 LE | 2    | dBm                                             |
| 11     | `rsrq`           | i8     | 1    | dB                                              |
| 12     | `mcc`            | u16 LE | 2    | Mobile Country Code                             |
| 14     | `mnc`            | u16 LE | 2    | Mobile Network Code                             |
| 16     | `cell_id`        | u32 LE | 4    |                                                 |
| 20     | `tac`            | u16 LE | 2    | Tracking Area Code                              |
| 22     | `earfcn`         | u32 LE | 4    | E-UTRA channel number                           |
| 26     | `num_sv`         | u8     | 1    | satellites in fix                               |
| 27     | `fix_type`       | u8     | 1    | from NAV-PVT (0 = no fix, 2 = 2D, 3 = 3D, …)    |
| 28     | `fw_version`     | u16 LE | 2    | encoded as `major × 256 + minor`                |
| 30     | `mode`           | u8     | 1    | operating mode ID (see Device modes)            |
| 31     | `flags`          | u8     | 1    | bit 0: `is_moving`, bit 1: `stationary_active`  |

**Decode example (Python):**

```python
STATUS_STRUCT = struct.Struct("<IHHbhbHHIHIBBHBB")  # 32 bytes

(ts_s, ts_ms, batt_mv, rssi, rsrp, rsrq,
 mcc, mnc, cell_id, tac, earfcn,
 num_sv, fix_type, fw_raw, mode_id, flags) = STATUS_STRUCT.unpack(payload[:32])

battery_v = batt_mv / 1000
fw_version = f"{fw_raw >> 8}.{fw_raw & 0xFF}"
is_moving = bool(flags & 0x01)
stationary_active = bool(flags & 0x02)
```

---

## Payload — IMU samples (`MSG_TYPE = 0x03`)

Used only when IMU streaming is enabled (no preset enables it by default). Payload is a base timestamp followed by N samples that carry only a 16-bit offset, to keep the per-sample size down.

```
+--------------+----------------+-------------+-------------+---    ---+
| base_ts_s    | base_ts_ms     | sample #1   | sample #2   |   ...   |
| u32 LE       | u16 LE         | 15 bytes    | 15 bytes    |         |
+--------------+----------------+-------------+-------------+---    ---+
```

**`imu_sample_t` — 15 bytes:**

| Offset | Field          | Type   | Unit              |
|--------|----------------|--------|-------------------|
| 0      | `ts_offset_ms` | u16 LE | ms since base_ts  |
| 2      | `ax`           | i16 LE | mg (milli-g)      |
| 4      | `ay`           | i16 LE | mg                |
| 6      | `az`           | i16 LE | mg                |
| 8      | `gx`           | i16 LE | 0.1 dps           |
| 10     | `gy`           | i16 LE | 0.1 dps           |
| 12     | `gz`           | i16 LE | 0.1 dps           |
| 14     | `flags`        | u8     | bit 0: moving, bit 1: fall, bit 2: impact |

To convert mg → m/s²: `ax_ms2 = (ax / 1000) × 9.80665`. To convert 0.1 dps → dps: `gx_dps = gx / 10`.

---

## Payload — MARK (`MSG_TYPE = 0x04`)

Identical layout to a single `gps_record_t` (24 B). The MARK frame is sent with **MQTT QoS 1** instead of the default QoS 0 used by the other types, so a user button press doesn't get lost in transit. The server de-duplicates by `(ICCID, timestamp_s, timestamp_ms)` should the QoS retry deliver twice.

---

## Multi-frame concatenation

A single MQTT message **MAY contain multiple back-to-back frames**. The firmware splits long logging batches like this when the natural drain produces more than 42 positions: e.g. mode `Log 10s` typically emits 100 positions as `[gps_batch(42)][gps_batch(42)][gps_batch(16)]` inside one MQTT publish.

A decoder must therefore loop:

```python
def iter_frames(buffer: bytes):
    offset = 0
    while offset < len(buffer):
        if len(buffer) - offset < 7:
            return  # not enough for even an empty frame
        if buffer[offset] != 0x4D or buffer[offset + 1] != 0x41:
            return  # corrupted/non-frame trailing data — stop
        msg_type = buffer[offset + 2]
        length   = buffer[offset + 3] | (buffer[offset + 4] << 8)
        if offset + 5 + length + 2 > len(buffer):
            return  # truncated
        payload  = buffer[offset + 5 : offset + 5 + length]
        ck_a     = buffer[offset + 5 + length]
        ck_b     = buffer[offset + 5 + length + 1]
        # verify checksum over MSG_TYPE + LENGTH + PAYLOAD
        calc_a, calc_b = fletcher8(buffer[offset + 2 : offset + 5 + length])
        if (calc_a, calc_b) == (ck_a, ck_b):
            yield msg_type, payload
        # advance to next frame
        offset += 7 + length
```

The `/v1` server already does this for you. When you fetch a binary topic with `format=raw`, the response includes per-frame metadata so you don't have to re-scan:

```json
{
  "type": "packet",
  "topic": "bin",
  "format": "raw",
  "received_at": "2026-05-25T14:30:30.701Z",
  "raw": {
    "buffer_b64": "TUEB…",
    "size": 1009,
    "frames": [
      { "offset": 0,   "msg_type": 1, "msg_type_name": "GPS", "length": 1009, "checksum_ok": true }
    ]
  }
}
```

`frames[].offset` is the byte offset inside the decoded `buffer_b64` where each frame begins. `checksum_ok` reflects what the server saw — if a frame fails its own checksum, it's still listed so you can decide whether to attempt recovery or skip.

---

## Worked example — decode a 24 B GPS record

A single-position frame on `bin` is **32 bytes total**: 5-byte header + 1-byte count + 24-byte record + 2-byte checksum. Hex dump (bytes 0–31, plus annotations):

```
offset  bytes                                            field
─────── ─────────────────────────────────────────────── ────────────────────────────
0       4D 41                                            SYNC_1 SYNC_2
2       01                                               MSG_TYPE = MA_MSG_GPS
3       19 00                                            LENGTH   = 25 (LE u16)
5       01                                               batch count = 1
6       1F 8D B9 6B                                      gps.timestamp_s
10      02 00                                            gps.timestamp_ms
12      70 BE 1A 1F                                      gps.lat
16      01 38 8D 0C                                      gps.lon
20      18 00                                            gps.speed_3d
22      D4 BD 06 00                                      gps.hMSL
26      78 0C 00 00                                      gps.hAcc
30      CK_A CK_B                                        Fletcher-8 over bytes 2..29
```

Decoded values:

| Field            | Raw little-endian   | Decoded value                                         |
|------------------|---------------------|-------------------------------------------------------|
| `timestamp_s`    | `1F 8D B9 6B`       | 0x6BB98D1F = 1808237343 → 2027-04-26 11:42:23 UTC     |
| `timestamp_ms`   | `02 00`             | 2                                                     |
| `lat` (×1e7)     | `70 BE 1A 1F`       | 0x1F1ABE70 = 521903728 → **52.1903728°**              |
| `lon` (×1e7)     | `01 38 8D 0C`       | 0x0C8D3801 = 210731009 → **21.0731009°**              |
| `speed_3d` (cm/s)| `18 00`             | 0x0018 = 24 → **0.24 m/s (0.86 km/h)**                |
| `hMSL` (mm)      | `D4 BD 06 00`       | 0x0006BDD4 = 441812 → **441.812 m**                   |
| `hAcc` (mm)      | `78 0C 00 00`       | 0x00000C78 = 3192 → **3.192 m**                       |

The checksum at offset 30–31 is the Fletcher-8 of bytes `01 19 00 01 1F 8D B9 6B 02 00 70 BE 1A 1F 01 38 8D 0C 18 00 D4 BD 06 00 78 0C 00 00` (MSG_TYPE + LENGTH + PAYLOAD, 28 bytes). Run [`fletcher8()`](#fletcher-8-checksum) over those bytes to verify any captured frame.

---

## Implementations to read

These two are the canonical reference implementations:

- **C (firmware encoder, authoritative):** `LTE-M-GPS-Tracker-VSC/main/binary_protocol.h` + `.c` *(private)*
- **Python (decoder, public):** [`motionapi_logger.py`](https://github.com/MotionApi/example-api-python-logger/blob/main/motionapi_logger.py) — uses `struct.Struct("<IHiiHiI")` for `gps_record_t` and the same `fletcher8` algorithm shown above.

If you spot a discrepancy between this document and the firmware source, the firmware wins — please open an issue and we'll fix the doc.
