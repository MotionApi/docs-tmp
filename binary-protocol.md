# MA Binary Protocol — Wire Format Reference

This document specifies the **MA Protocol** (MotionApi binary protocol), the on-the-wire frame format that the tracker uses to publish GPS, status and MARK data to the broker on the `bin` topic. It's UBX-inspired (uBlox GNSS protocol family) and is what you receive byte-for-byte from the `/v1` packet surfaces — `GET /v1/devices/:iccid/last[/:topic]`, `WS /v1/stream` and SSE `/v1/devices/:iccid/stream` — when you ask for `format=raw`.

> **You usually don't need this.** The `/v1` API gives you `format=formatted` by default — fully decoded JSON in SI units. Use `format=raw` (and this document) only when you want to write your own decoder, archive bytes exactly as transmitted, or debug protocol-level issues.

---

## Why binary

Over LTE-M with a 1024 B MQTT payload cap and SIM data budgets that matter, encoding overhead is the dominant cost. A complete GPS fix fits in **25 B** and a full device status in **34 B**. Those same eight GPS fields encoded as minified JSON — `{"ts":1785489247,"ms":200,"lat":52.1846384,"lon":21.0581505,"speed":4.12,"alt":108.25,"hacc":1.45,"cadence":74}` — take 111 B, and that is before MQTT topic and framing overhead.

Compactness is what makes batching practical: at 25 B per record, **40 positions** fit in a single 1008-byte frame.

All telemetry — GPS, STATUS and MARK, in every mode — is published as binary MA frames on `t/{ICCID}/bin`. The device publishes JSON on exactly one uplink topic, `t/{ICCID}/response`, and only for command / OTA acknowledgements. Downlink commands arrive on `c/{ICCID}/#`.

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
| `MQTT_MAX_PAYLOAD_BYTES`  | 1024  | Hard limit — uBlox LEXI-R422 `AT+UMQTTC=9` MQTT payload max        |
| `GPS_BATCH_MAX_COUNT`     | 40    | `(1024 − 7 overhead − 1 count) / 25 B per record`                  |

`GPS_BATCH_MAX_COUNT` is **derived, not fixed** — the firmware computes it at runtime from `sizeof(gps_record_t)`. It was 42 while the record was 24 B. Compute it yourself rather than hard-coding it; see [Versioning and forward compatibility](#versioning-and-forward-compatibility).

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

The snippet above is byte-identical to both first-party implementations, so it is a usable reference on its own. Those implementations are `ma_checksum_compute()` in the firmware's `main/binary_protocol.cpp` (declared in `main/binary_protocol.h`) and `fletcher8()` in the platform's `packages/shared/constants/protocol.ts` — both repositories are private. Any UBX Fletcher-8 implementation will also work.

---

## Message types

| Constant       | Value | Topic mapping     | Payload                                                        |
|----------------|-------|-------------------|-----------------------------------------------------------------|
| `MA_MSG_GPS`   | 0x01  | `t/{ICCID}/bin`   | `[count:u8][gps_record_t × count]` (batched, records 25 B)      |
| `MA_MSG_STATUS`| 0x02  | `t/{ICCID}/bin`   | One `status_record_t` (34 B on current firmware)                |
| `MA_MSG_IMU`   | 0x03  | —                 | **Reserved** — no firmware emits this type. See [IMU](#payload--imu-samples-msg_type--0x03-reserved) |
| `MA_MSG_MARK`  | 0x04  | `t/{ICCID}/bin`   | One bare `gps_record_t` (25 B, no count byte) — QoS 1           |

Unknown `MSG_TYPE` values: a decoder should validate the checksum, skip the frame, and continue scanning the buffer for the next `0x4D 0x41` sync pair.

---

## Versioning and forward compatibility

There is **no protocol version byte** in an MA frame, and the `fw_version` field inside `status_record_t` **must not be used for feature detection**. It is encoded as `major × 256 + minor`, so the patch component is discarded, and it was not bumped when `cadence_spm` was added — a device sending 25-byte records reports exactly the same value as one sending 24-byte records.

Records only ever grow by **appending trailing fields**. Existing offsets never move and existing fields are never resized or reordered. The normative rule for telling generations apart is therefore length arithmetic, and it is what both the firmware and the server use:

| Message  | Rule                                                                                                    |
|----------|---------------------------------------------------------------------------------------------------------|
| GPS `0x01` | `record_size = (payload_len − 1) / count`. Read `cadence_spm` only when `record_size >= 25`.           |
| MARK `0x04`| Read `cadence_spm` only when `payload_len >= 25`.                                                      |
| STATUS `0x02` | `temp_c` present when `payload_len >= 33`; `rat` present when `payload_len >= 34`.                  |

Write your decoder so that:

- it never hard-codes a record size — always derive it from the frame;
- `(payload_len − 1) % count != 0` is treated as a malformed frame, not as something to round;
- bytes beyond the layout you know about are ignored rather than rejected, so a future trailing field doesn't break you;
- a missing trailing field is reported as **absent**, not as `0` — those mean different things (see [Stroke cadence](#stroke-cadence-cadence_spm)).

---

## Payload — GPS record (`gps_record_t`)

25 bytes, packed, little-endian. Used for **single positions** (`MARK`) and inside the **GPS batch** payload.

| Offset | Field          | Type   | Size | Unit              | Notes                                          |
|--------|----------------|--------|------|-------------------|------------------------------------------------|
| 0      | `timestamp_s`  | u32 LE | 4    | Unix epoch seconds (UTC) | From UBX NAV-PVT date/time fields        |
| 4      | `timestamp_ms` | u16 LE | 2    | milliseconds (0–999)     |                                            |
| 6      | `lat`          | i32 LE | 4    | degrees × 1e7     | Direct from NAV-PVT (e.g. `521846384` = 52.1846384°) |
| 10     | `lon`          | i32 LE | 4    | degrees × 1e7     |                                                |
| 14     | `speed_3d`     | u16 LE | 2    | cm/s              | **3D** speed, not ground speed. Max 655.35 m/s ≈ 2359 km/h |
| 16     | `hMSL`         | i32 LE | 4    | millimeters above mean sea level | Direct from NAV-PVT             |
| 20     | `hAcc`         | u32 LE | 4    | millimeters (horizontal accuracy estimate) |                       |
| 24     | `cadence_spm`  | u8     | 1    | strokes/min       | Paddling cadence — trailing field, see below   |

There is no heading/course field in this record.

`cadence_spm` was appended after the original 24-byte layout, so a decoder must confirm the record really is 25 bytes before reading it — see [Versioning and forward compatibility](#versioning-and-forward-compatibility).

**Decode example (Python):**

```python
import struct
GPS_STRUCT_BASE    = struct.Struct("<IHiiHiI")   # 24 bytes — pre-cadence firmware
GPS_STRUCT_CADENCE = struct.Struct("<IHiiHiIB")  # 25 bytes — current firmware

def decode_gps_record(buf: bytes, off: int, rec_size: int) -> dict:
    if rec_size >= GPS_STRUCT_CADENCE.size:
        (ts_s, ts_ms, lat_e7, lon_e7, speed_cms,
         hmsl_mm, h_acc_mm, cadence) = GPS_STRUCT_CADENCE.unpack_from(buf, off)
    else:
        (ts_s, ts_ms, lat_e7, lon_e7, speed_cms,
         hmsl_mm, h_acc_mm) = GPS_STRUCT_BASE.unpack_from(buf, off)
        cadence = None  # field absent — NOT the same as 0
    return {
        "timestamp_s": ts_s,
        "timestamp_ms": ts_ms,
        "lat_deg": lat_e7 / 1e7,
        "lon_deg": lon_e7 / 1e7,
        "speed_ms": speed_cms / 100,
        "altitude_m": hmsl_mm / 1000,
        "h_acc_m": h_acc_mm / 1000,
        "cadence_spm": cadence,
    }
```

### GPS batch payload (`MSG_TYPE = 0x01`)

The `bin` topic only ever publishes GPS in **batched** form, even for a single position. The payload prefix is one byte: the number of records, followed by that many `gps_record_t` back-to-back.

```
+-------+-----------------+-----------------+---     ---+-----------------+
| count |  gps_record #1  |  gps_record #2  |   ...    |  gps_record #N  |
| u8    |   25 bytes      |   25 bytes      |          |   25 bytes      |
+-------+-----------------+-----------------+---     ---+-----------------+

Total payload size = 1 + (count × 25)
Max count per frame = 40   →  payload 1 + 40×25 = 1001 B, frame 1008 B ≤ 1024 B
```

Derive the count limit as `floor((MQTT_MAX_PAYLOAD_BYTES − 7 − 1) / record_size)` rather than hard-coding 40.

Logging modes never exceed this. At the default 10 Hz mode `Log10s` accumulates ~100 positions per 10 s interval, but the firmware drains **one batch per send interval** and publishes it as its **own MQTT message**: 40 records go out and the remainder stays in the device's ring buffer until the next interval. Batches are never merged into a larger frame, and every frame is decoded independently — see [Multi-frame concatenation](#multi-frame-concatenation). This caps drain throughput at 40 records per interval — at 10 Hz sampling, `Log5s` (8 rec/s) and `Log10s` (4 rec/s) drain slower than fixes accumulate, so they only keep up at lower GNSS rates.

---

## Stroke cadence (`cadence_spm`)

Byte 24 of every 25-byte `gps_record_t`, on both GPS batch records and MARK payloads. `uint8`, unit **strokes per minute**.

- **Counting convention.** `spm` counts **every blade entry** (sprint-kayak convention). Full stroke cycles per minute are therefore `spm / 2` for a **kayak** (left + right blade per cycle) and `spm` for a **canoe** (single-sided).
- **Value range.** Either `0`, or a non-zero value clamped to **28–165**. There is no separate "invalid" sentinel.
- **`0` is ambiguous.** It means **any** of: the device's sport profile is off (this is the default), the paddler is not currently paddling, or the IMU is unavailable. Nothing on the wire reports the active sport profile — `status_record_t.mode` is the mode preset, not the sport — so you cannot distinguish these three cases from the telemetry alone. Confirm out of band.
- **Absent ≠ 0.** A 24-byte record has no cadence field at all. Keep that distinct from an on-wire `0`.
- **Enabling it.** Cadence is only produced when the device's persisted sport profile is `1` (canoe), `2` (kayak) or `3` (paddle, auto-detect); the default is `0` (none). The profile is set with `{"cmd":"config","sport":N}` published on `c/{ICCID}/config`.

The profile can also be set through the public `/v1` API: `PUT /v1/devices/:iccid/sport` with `{"sport_id": N}` (scope `write:mode`; catalog at `GET /v1/devices/_sports`, bulk variant `PUT /v1/devices/_all/sport`). `POST /v1/devices/:iccid/commands` still rejects `cmd:"config"` — the dedicated endpoint is the only REST path.

**Where you can read it:** raw `bin` frames (this document), `/v1` with `format=raw` (decode the base64 yourself) and `/v1` with `format=formatted` (`cadence_spm` on the GPS object, omitted entirely for 24-byte records). It is **not persisted** to the time-series store, so it does not appear in historical position queries — only on live packets and streams.

---

## Payload — Status record (`status_record_t`)

34 bytes on current firmware, packed, little-endian. Sent on every `STATUS` interval (default 30 s — see [§ Device modes](README.md#device-modes)). The first 32 bytes are the original layout; `temp_c` and `rat` are optional trailing fields.

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
| 28     | `fw_version`     | u16 LE | 2    | `major × 256 + minor` — patch dropped; **not** a feature flag |
| 30     | `mode`           | u8     | 1    | active mode **preset ID** (1–10 standard, 20–28 logging) |
| 31     | `flags`          | u8     | 1    | bit 0: `is_moving`, bit 1: `stationary_active`  |
| 32     | `temp_c`         | i8     | 1    | °C ambient — **optional**; `-128` (`INT8_MIN`) = no reading |
| 33     | `rat`            | u8     | 1    | radio access technology — **optional**; `0xFF` = unknown |

`mode` is a preset ID, not a type flag. Note that the logging IDs are not contiguous: the 18 presets are 1–10 and 20, 21, 22, 23, 25, 26, 27, 28.

`rat` carries the ubxlib `uCellNetRat_t` enum verbatim. The values you are likely to see: `8` = LTE, `10` = LTE-M (CAT-M1), `11` = NB-IoT (NB1); `0` and `0xFF` both mean unknown.

Accept 32-, 33- and 34-byte STATUS payloads and report missing trailing fields as absent.

**Decode example (Python):**

```python
STATUS_STRUCT_BASE = struct.Struct("<IHHbhbHHIHIBBHBB")    # 32 bytes — original layout
STATUS_STRUCT_FULL = struct.Struct("<IHHbhbHHIHIBBHBBbB")  # 34 bytes — current firmware

(ts_s, ts_ms, batt_mv, rssi, rsrp, rsrq,
 mcc, mnc, cell_id, tac, earfcn,
 num_sv, fix_type, fw_raw, mode_id, flags) = STATUS_STRUCT_BASE.unpack_from(payload, 0)

temp_c = None
if len(payload) >= 33:
    raw_temp = payload[32] - 256 if payload[32] > 127 else payload[32]  # int8
    if raw_temp != -128:                       # INT8_MIN = no reading
        temp_c = raw_temp

rat = None
if len(payload) >= 34 and payload[33] not in (0x00, 0xFF):
    rat = payload[33]

battery_v = batt_mv / 1000
fw_version = f"{fw_raw >> 8}.{fw_raw & 0xFF}"   # patch component is not on the wire
is_moving = bool(flags & 0x01)
stationary_active = bool(flags & 0x02)
```

---

## Payload — IMU samples (`MSG_TYPE = 0x03`, reserved)

**Reserved — you will not receive these frames.** `MA_MSG_IMU` and `imu_sample_t` are declared in the protocol header and understood by the server's decoder, but no current firmware emits them: there is no encoder for type `0x03` and no mode preset that enables IMU streaming. The layout is documented so decoders can be forward-compatible.

The payload would be a **7-byte header** — base timestamp plus an explicit sample count — followed by N samples that carry only a 16-bit offset, to keep the per-sample size down.

```
+--------------+----------------+---------+-------------+-------------+---    ---+
| base_ts_s    | base_ts_ms     | count   | sample #1   | sample #2   |   ...   |
| u32 LE @0    | u16 LE @4      | u8 @6   | 15 bytes    | 15 bytes    |         |
+--------------+----------------+---------+-------------+-------------+---    ---+

Total payload size = 7 + (count × 15)
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

One bare `gps_record_t` (25 B on current firmware) — **no leading count byte**, unlike `MA_MSG_GPS`. That makes the whole frame 32 bytes. A MARK is raised on the device by a double-click, not by a remote command.

The MARK frame is published with **MQTT QoS 1** instead of the QoS 0 used for GPS and STATUS, so a user button press doesn't get lost in transit. The broker may therefore deliver it **more than once**. The server does **not** de-duplicate: a retried MARK is stored and broadcast again. If duplicates matter to you, key on `(ICCID, timestamp_s, timestamp_ms)` yourself.

Because MARK has no count byte, its record size is derived from the payload length alone: read `cadence_spm` only when `payload_len >= 25`.

---

## Multi-frame concatenation

A single MQTT message **MAY contain multiple back-to-back frames**. The case that actually occurs today is the stationary-sleep update: the firmware concatenates a STATUS frame and a single-record GPS frame into one buffer and publishes them together — 41 + 33 = **74 bytes, two frames, one payload**.

Logging batches are *not* concatenated this way: each drained batch is its own MQTT publish. Decoders must still loop, because concatenation can appear on any binary payload.

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

The `/v1` server already does this for you. When you fetch a binary topic with `format=raw`, the response includes per-frame metadata so you don't have to re-scan. Here is the stationary STATUS+GPS bundle described above, exactly as it comes back:

```json
{
  "type": "packet",
  "iccid": "8988228066612345678",
  "topic": "bin",
  "format": "raw",
  "received_at": "2026-07-31T09:14:07.200Z",
  "device_mode": { "id": 1, "name": "Default", "label": "Default", "type": "standard" },
  "raw": {
    "buffer_b64": "TUECIgBfZ2xqyACsD7mh//UEAQYATmG8AOEQnBgAAAsDAwABAhgK4ltNQQEaAAFfZ2xqyABwvhofATiNDJwB2qYBAKoFAABK0LM=",
    "size": 74,
    "frames": [
      { "msg_type": 2, "msg_type_name": "STATUS", "length": 34, "checksum_ok": true, "offset": 0 },
      { "msg_type": 1, "msg_type_name": "GPS",    "length": 26, "checksum_ok": true, "offset": 41 }
    ]
  }
}
```

`frames[].offset` is the byte offset inside the decoded `buffer_b64` where each frame begins, and `frames[].length` is that frame's **payload** length — the frame occupies `7 + length` bytes. `checksum_ok` is a real per-frame Fletcher-8 result recomputed from those bytes: `false` means the frame is present in the raw buffer but the parser dropped it, so it has no counterpart in the `formatted` view.

Each frame is decoded independently. With `format=formatted`, one MQTT payload yields one delivery per frame — a GPS frame becomes `payload.kind = "gps_batch"` with `data` containing only *that frame's* positions. There is no cross-frame or cross-packet reassembly; stitch batches together yourself if you need a continuous track.

---

## Worked example — decode a 25 B GPS record

A single-position frame on `bin` is **33 bytes total**: 5-byte header + 1-byte count + 25-byte record + 2-byte checksum. This is the second frame of the bundle shown above.

```
4D 41 01 1A 00 01 5F 67 6C 6A C8 00 70 BE 1A 1F 01 38 8D 0C
9C 01 DA A6 01 00 AA 05 00 00 4A D0 B3
```

```
offset  bytes                                            field
─────── ─────────────────────────────────────────────── ────────────────────────────
0       4D 41                                            SYNC_1 SYNC_2
2       01                                               MSG_TYPE = MA_MSG_GPS
3       1A 00                                            LENGTH   = 26 (LE u16)
5       01                                               batch count = 1
6       5F 67 6C 6A                                      gps.timestamp_s
10      C8 00                                            gps.timestamp_ms
12      70 BE 1A 1F                                      gps.lat
16      01 38 8D 0C                                      gps.lon
20      9C 01                                            gps.speed_3d
22      DA A6 01 00                                      gps.hMSL
26      AA 05 00 00                                      gps.hAcc
30      4A                                               gps.cadence_spm
31      D0 B3                                            CK_A CK_B — Fletcher-8 over bytes 2..30
```

`LENGTH` is 26 = 1 count byte + 1 × 25 B record. Derived record size: `(26 − 1) / 1 = 25`, so `cadence_spm` is present.

Decoded values:

| Field              | Raw little-endian   | Decoded value                                         |
|--------------------|---------------------|-------------------------------------------------------|
| `timestamp_s`      | `5F 67 6C 6A`       | 0x6A6C675F = 1785489247 → 2026-07-31 09:14:07 UTC     |
| `timestamp_ms`     | `C8 00`             | 200                                                   |
| `lat` (×1e7)       | `70 BE 1A 1F`       | 0x1F1ABE70 = 521846384 → **52.1846384°**              |
| `lon` (×1e7)       | `01 38 8D 0C`       | 0x0C8D3801 = 210581505 → **21.0581505°**              |
| `speed_3d` (cm/s)  | `9C 01`             | 0x019C = 412 → **4.12 m/s (14.83 km/h)**              |
| `hMSL` (mm)        | `DA A6 01 00`       | 0x0001A6DA = 108250 → **108.250 m**                   |
| `hAcc` (mm)        | `AA 05 00 00`       | 0x000005AA = 1450 → **1.450 m**                       |
| `cadence_spm`      | `4A`                | 0x4A = 74 → **74 strokes/min** (= 37 kayak cycles/min) |

The checksum at offset 31–32 is the Fletcher-8 of the 29 bytes `01 1A 00 01 5F 67 6C 6A C8 00 70 BE 1A 1F 01 38 8D 0C 9C 01 DA A6 01 00 AA 05 00 00 4A` (MSG_TYPE + LENGTH + PAYLOAD), giving `CK_A = 0xD0`, `CK_B = 0xB3`. Run [`fletcher8()`](#fletcher-8-checksum) over those bytes to verify any captured frame.

---

## Implementations to read

The authoritative encoder is the firmware:

- **C++ (firmware encoder, authoritative):** `main/binary_protocol.h` + `main/binary_protocol.cpp`, relative to the firmware repository root *(private)*.
- **TypeScript (server decoder):** `packages/shared/constants/protocol.ts` and `apps/backend/src/mqtt/binaryProtocol.ts`, relative to the platform repository root *(private)*.

Our public example client, [`motionapi_logger.py`](https://github.com/MotionApi/example-api-python-logger/blob/main/motionapi_logger.py), consumes `format=formatted` JSON over the WebSocket stream and does **not** decode binary frames — use it as an authentication/streaming reference only. The `fletcher8()`, `decode_gps_record()` and `iter_frames()` snippets in this document are enough to build a full decoder.

If you spot a discrepancy between this document and the firmware source, the firmware wins — please open an issue and we'll fix the doc.
