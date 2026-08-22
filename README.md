# MotionApi Public API — Developer Guide

Developer-facing guide for the **MotionApi `/v1` public API**. It tells you how to issue an API key, what endpoints exist, how to receive real-time data, and how to run the two reference examples we ship.

> The auto-generated OpenAPI reference at <https://api.motionapi.pro/v1/docs> is the source of truth for request and response shapes. This guide explains the **how** and the **why** around it.

---

## Contents

1. [What you can do with the API](#what-you-can-do-with-the-api)
2. [Getting an API key](#getting-an-api-key)
3. [Authentication](#authentication)
4. [Scopes](#scopes)
5. [Endpoints — REST](#endpoints--rest)
6. [Streaming — WebSocket & SSE](#streaming--websocket--sse)
7. [Packet formats — `raw` vs `formatted`](#packet-formats--raw-vs-formatted)
8. [Rate limits & error envelope](#rate-limits--error-envelope)
9. [Device modes](#device-modes)
10. [Paddling cadence](#paddling-cadence)
11. [Reference examples](#reference-examples)
    - [Browser — API Explorer](#example-1--browser-api-explorer)
    - [Python — Stream logger](#example-2--python-stream-logger)
12. [Binary protocol reference](binary-protocol.md) — wire format for `format=raw`

---

## What you can do with the API

MotionApi sells GPS+LTE-M tracker hardware. The `/v1` API lets you build on top of those trackers without touching MQTT directly:

- **List your devices** and their last-known position / battery / signal.
- **Read snapshots** — the most recent packet on every topic per device (`bin`, `response`, plus the legacy `gps`, `status`, `event`, `imu` topic names).
- **Subscribe to live data** over WebSocket or Server-Sent Events.
- **Send commands** (identify, buzzer, LED, reboot, shutdown, sleep / wakeup, GPS sampling rate, …).
- **Change the device mode preset** (18 presets — standard 1–10, logging 20–23 and 25–28).

All real-time data flows through the backend; you never need MQTT credentials.

> **Reality check on topics.** A current device publishes on exactly two topics: `t/{ICCID}/bin` (all telemetry — positions, status, MARK, as binary frames) and `t/{ICCID}/response` (command acknowledgements and OTA / IMU-upload progress, as JSON). The `gps`, `status`, `event` and `imu` topic names are still accepted by the backend as vestigial back-compat for pre-binary firmware, but no shipping device produces them — expect `data: null` if you read them.

> **MARK is not a command.** A MARK waypoint is triggered on the device by a double-click of the button; there is no `mark` command on the API. See [Device modes → How modes look on the wire](#how-modes-look-on-the-wire).

**Base URL:** `https://api.motionapi.pro`

---

## Getting an API key

1. Sign in to the **user panel** at <https://app.motionapi.pro>.
2. From the left nav choose **API Access**.

   ![API Access page in the user panel](screenshots/api-access-page.png)

3. Click **Create Key** in the top-right.
4. The modal has **two** fields:
   - **Key Name** — anything that helps you remember which integration uses it. Required.
   - **Scopes** — tick what you need. Pre-ticked: `read:devices`, `read:stream`, `read:last`. At least one is required.

   ![Create Key modal with scope checkboxes](screenshots/create-key-modal.png)

   > Keys created from the panel are always **`live`** environment, cover **all** your devices, use the default rate limit, and **never expire** — the UI has no selector for any of those. (The backend's own `POST /api-keys` route does accept `env`, `device_ids`, `rate_limit_per_minute` and `expires_at`, but nothing in the panel supplies them today.)

5. Click **Create**. The full key is shown **once** — copy it now. After you close the modal the secret is gone (only the prefix is retrievable later). If you lose it, **rotate the key**.

   ![Created key reveal — copy the full secret before closing](screenshots/key-created-reveal.png)

A key looks like:

```
mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
└┬┘ └┬─┘ └───┬───┘ └─────────────┬─────────────┘
 │   │       │                   └── 32-char secret (kept server-side as sha256)
 │   │       └── 8-char prefix tail — used for listings + audit log
 │   └── environment: `live` or `test`
 └── product prefix: "MotionApi Key"
```

The **prefix** (`mak_live_abcd1234`) is safe to log. The full key is not. Treat it like a password.

---

## Authentication

Send the full key as a `Bearer` token in the `Authorization` header on every request:

```http
GET /v1/devices HTTP/1.1
Host: api.motionapi.pro
Authorization: Bearer mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

For browser-native **WebSocket** and **Server-Sent Events** (where custom headers aren't allowed) pass the key in the `?token=` query string instead:

```
wss://api.motionapi.pro/v1/stream?token=mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

Authorization header always wins if both are set.

---

## Scopes

Each API key carries a list of scopes. Endpoints require specific scopes; if a scope is missing you get **403 Forbidden**.

**Enforced today** — these five gate real routes:

| Scope             | Grants                                                                  |
|-------------------|-------------------------------------------------------------------------|
| `read:devices`    | List devices, read metadata, read current mode                          |
| `read:last`       | Read the latest cached packet (`/last`, `/last/:topic`)                 |
| `read:stream`     | Subscribe to WebSocket / SSE real-time streams                          |
| `write:commands`  | Send commands (buzzer, LED, identify, reboot, …)                        |
| `write:mode`      | Change device mode preset                                               |

**Reserved — granting them unlocks nothing yet.** The panel offers them and the backend stores them, but no `/v1` route checks them because the matching endpoints do not exist:

| Scope             | Reserved for                                                            |
|-------------------|-------------------------------------------------------------------------|
| `read:positions`  | Historical GPS query (not implemented)                                  |
| `read:status`     | Historical device-status query (not implemented)                        |
| `read:events`     | Historical events query (not implemented)                               |
| `read:imu`        | Raw IMU data (not implemented — no device firmware emits IMU frames)    |
| `write:ota`       | Firmware upload (not implemented on `/v1`)                              |

Principle of least privilege — for a dashboard that just shows positions you only need `read:devices` + `read:last` + `read:stream`, which is exactly the pre-ticked default.

---

## Endpoints — REST

All endpoints below sit under `https://api.motionapi.pro/v1` and return the same envelope:

```json
{ "success": true,  "data": ..., "meta": { ... } }
{ "success": false, "error": { "code": "rate_limited", "message": "...", "details": {} } }
```

### Devices

| Method | Path                                         | Scope          | Description                                          |
|--------|----------------------------------------------|----------------|------------------------------------------------------|
| GET    | `/v1/devices`                                | `read:devices` | List devices your key can access                     |
| GET    | `/v1/devices/:iccid`                         | `read:devices` | Single device with cached status + current mode      |
| GET    | `/v1/devices/:iccid/mode`                    | `read:devices` | Full current mode preset (timings, IMU flag, …)      |
| PUT    | `/v1/devices/:iccid/mode`                    | `write:mode`   | Switch the device to a different mode preset         |
| PUT    | `/v1/devices/_all/mode`                      | `write:mode`   | Switch **every device your key can control** to a mode preset — see [Fan-out](#fan-out--all-your-devices-at-once) |
| GET    | `/v1/devices/_modes`                         | (any key)      | Static catalog of all 18 mode presets                |

No specific scope is required for `_modes`, but **every** `/v1` route requires authentication — an unauthenticated call still gets 401.

### Snapshots — last cached packet per topic

| Method | Path                                         | Scope       | Description                                                       |
|--------|----------------------------------------------|-------------|-------------------------------------------------------------------|
| GET    | `/v1/devices/:iccid/last`                    | `read:last` | Latest packet on every topic the device has spoken                |
| GET    | `/v1/devices/:iccid/last/:topic`             | `read:last` | Latest packet on one specific topic (see below)                   |

Valid `:topic` values are exactly `bin`, `response`, `gps`, `status`, `event`, `imu`. Anything else is a **400 `bad_request`**, not a 404. In practice a current device only ever populates `bin` (all telemetry) and `response` (command acks, OTA / IMU-upload progress) — the other four are legacy JSON names kept for back-compat and will read back `null`.

Add `?format=raw` to get the bytes the device actually sent (base64 + frame metadata), or `?format=formatted` for human-readable JSON (default).

### Commands

| Method | Path                                         | Scope             | Description                                              |
|--------|----------------------------------------------|-------------------|----------------------------------------------------------|
| POST   | `/v1/devices/:iccid/commands`                | `write:commands`  | Send a command to the device over the MQTT downlink      |
| POST   | `/v1/devices/_all/commands`                  | `write:commands`  | Same command to **every device your key can control** — see [Fan-out](#fan-out--all-your-devices-at-once) |

Body shape is `{ "cmd": "<name>", "params": { … } }`. Accepted commands:

| `cmd`          | Params                                                    | Notes                                                                     |
|----------------|-----------------------------------------------------------|---------------------------------------------------------------------------|
| `identify`     | —                                                         | Flashes the LED and beeps 5× so you can locate the unit                    |
| `buzzer`       | `duration_ms` (100–10000, default 1000)                   | Fixed tone — the device has no frequency control                           |
| `led`          | `state` (bool), `blink_count` (default 0), `blink_interval_ms` (default 500) | `blink_count > 0` blinks and ignores `state`; `0` latches the LED to `state` |
| `reboot`       | —                                                         | Restart, ~30 s offline                                                     |
| `shutdown`     | —                                                         | Powers the unit down; only the physical button brings it back              |
| `sleep`        | —                                                         | Force stationary mode: one combined STATUS+GPS packet every 60 s, or at the preset interval when that is slower. Motion still wakes the device |
| `wakeup`       | —                                                         | Leave stationary mode, and block re-entry for 3 minutes                    |
| `set_gps_rate` | `rate_hz` ∈ 1, 4, 5, 10, 20                               | GNSS **sampling** rate — orthogonal to the mode preset. 20 Hz requires `gps_otp` first |
| `gps_otp`      | —                                                         | **Irreversible** one-way OTP write that unlocks 20 Hz                      |
| `status`       | —                                                         | Acknowledged, but a **no-op on the device** — it does not force a STATUS publish; the next one arrives on the preset schedule |
| `ota_update`   | `manifest_url`, `version`, `sha256`, `size`               | Service use. All four are required, but the device only null-checks them — a malformed request still starts an OTA, and telemetry stops until it gives up |

```json
{ "cmd": "buzzer", "params": { "duration_ms": 500 } }
```

Two things this endpoint deliberately will not do:

- **`cmd: "config"` is rejected with 400.** Mode changes go through `PUT /v1/devices/:iccid/mode` instead.
- Because `config` is blocked, the **sport profile that enables paddling cadence cannot be set through `/v1` at all** — it is only reachable from the user panel today. See [Paddling cadence](#paddling-cadence).

Note that `/v1` does not range-check `buzzer` / `led` parameters; the min/max above are UI hints, and out-of-range values reach the device unvalidated. `set_gps_rate` **is** validated (400 on anything outside the five allowed rates).

### Fan-out — all your devices at once

`POST /v1/devices/_all/commands` and `PUT /v1/devices/_all/mode` take exactly the same body as their per-ICCID counterparts and apply it to **every device the key can control** in one request: devices you own plus devices shared with you with `control` or `manage` (a `view`-only share is skipped), further narrowed by the key's device whitelist if it has one. `_all` is a literal path segment, not an ICCID.

Each device gets its own `msg_id`, so command responses still correlate per device; the response lists them:

```json
{ "success": true,
  "data": { "broadcast_id": "7c0b…", "cmd": "buzzer", "total": 12,
            "sent": [ { "iccid": "8988228066680471500", "msg_id": "…" }, … ] } }
```

`PUT /_all/mode` returns the same `broadcast_id` / `total` / `sent` plus `requested_mode`. A key that can control no devices gets `403 forbidden`. Validation is identical to the single-device routes — `config` is still rejected on `/_all/commands`, so the sport profile still cannot be set through `/v1`.

A fan-out is **one request** for rate-limiting purposes, whatever `total` is — prefer it over looping `POST /v1/devices/:iccid/commands` per device, which burns one request each.

### Curl quick-start

```bash
# Replace the bearer token with your real key.
KEY="mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"

# List devices
curl -H "Authorization: Bearer $KEY" https://api.motionapi.pro/v1/devices

# Latest telemetry snapshot for one device.
# Use `bin` — that is the topic production firmware publishes on. `last/gps`
# is the legacy JSON topic and returns null for any current device.
curl -H "Authorization: Bearer $KEY" \
  'https://api.motionapi.pro/v1/devices/8988228066680471572/last/bin?format=formatted'

# Buzz for 500 ms
curl -X POST -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"cmd":"buzzer","params":{"duration_ms":500}}' \
  https://api.motionapi.pro/v1/devices/8988228066680471572/commands
```

The complete request/response schema for each endpoint is at <https://api.motionapi.pro/v1/docs> and <https://api.motionapi.pro/v1/openapi.json>.

---

## Streaming — WebSocket & SSE

Real-time data has two transports. Pick what's easier to consume:

### WebSocket — `wss://api.motionapi.pro/v1/stream`

Scope: `read:stream`. Bidirectional — you tell it which devices and topics you care about.

Flow:

1. **Connect** with the API key in `?token=`.
2. Server sends a `{"type":"ready", "authorized_devices":[…], "valid_topics":[…], "key_id":"…", "scopes":[…], "hint":"…"}` frame so you know what you may subscribe to.
3. You send a subscribe message:
   ```json
   {
     "type": "subscribe",
     "devices": ["8988228066680471572"],
     "topics": ["bin", "response"],
     "format": "formatted"
   }
   ```
   `devices` must be a non-empty array of strings, or the server replies `{"type":"error","error":"devices must be a non-empty array of ICCIDs"}`. ICCIDs your key cannot access are **not** an error — they come back in `devices_denied`.
4. Server responds with `{"type":"subscribed", "devices_authorized":[…], "devices_denied":[…], "topics":[…], "format":"…"}`.
5. Server pushes `{"type":"packet", "iccid":"…", "topic":"bin", "received_at":"…", "seq":…, "payload":{…}}` per packet, plus a `{"type":"heartbeat","ts":…}` frame every 30 s.

If you send `"format":"raw"` packets include `raw.buffer_b64` + `raw.frames[]` (for binary topics like `bin`) instead of `payload`. WebSocket additionally accepts `"format":"both"`, which emits the raw view **and** every formatted view under one `seq` — REST and SSE coerce anything that is not `raw` to `formatted`.

**Client → server messages:**

| Message                                          | Effect                                                                                       |
|--------------------------------------------------|----------------------------------------------------------------------------------------------|
| `{"type":"subscribe","devices":[…],"topics":[…],"format":"…"}` | **Adds** the devices to the existing set; **replaces** `topics` and `format` when you supply them |
| `{"type":"unsubscribe","devices":[…]}`           | Removes those devices from the set; replies `{"type":"unsubscribed","devices":[…]}`           |
| `{"type":"ping"}`                                | Replies `{"type":"pong","ts":…}`                                                              |

> Re-sending `subscribe` with a **different** device list does not narrow the stream — device subscriptions are **additive**. To stop receiving a device you must send `unsubscribe`. `topics` and `format`, in contrast, are replaced whenever you include them — omit them and the previous values stay in force.

### Server-Sent Events — `GET /v1/devices/:iccid/stream`

Scope: `read:stream`. One-way (server → you), one device per connection. Works through stricter corporate proxies that block WS.

```bash
curl -N "https://api.motionapi.pro/v1/devices/8988228066680471572/stream?topics=bin,response&format=formatted" \
  --header "Authorization: Bearer $KEY"
```

The stream uses **named** SSE events — `ready`, `packet` and `heartbeat` — so a plain `EventSource.onmessage` handler receives **nothing**. Per the SSE spec `onmessage` only fires for unnamed events. Attach listeners per event name:

```js
const es = new EventSource(url);   // pass the key in ?token=
es.addEventListener('ready',     e => console.log('connected', JSON.parse(e.data)));
es.addEventListener('packet',    e => handle(JSON.parse(e.data)));   // carries a monotonic `seq`
es.addEventListener('heartbeat', e => console.log('alive'));         // every 30 s
```

Use the `format=raw` querystring for raw bytes (anything else is treated as `formatted`); `topics=…` is a CSV filter. Response headers are `Content-Type: text/event-stream`, `Cache-Control: no-cache, no-transform`, `Connection: keep-alive`, `X-Accel-Buffering: no`.

### Close codes (WebSocket only)

| Code | Meaning                                              |
|------|------------------------------------------------------|
| 1000 | Normal close                                         |
| 4001 | Unauthorized — bad / missing / revoked API key       |
| 4003 | Forbidden — key missing `read:stream` scope          |

---

## Packet formats — `raw` vs `formatted`

Every read endpoint and stream subscription accepts a `format`:

- **`formatted`** *(default)* — JSON, SI units (m/s, meters, degrees), with mode preset expanded inline. Easiest to consume.
- **`raw`** — exact bytes received from the device. Binary frames (`bin` topic — the only telemetry path any shipping device uses) come as base64 plus per-frame metadata so you can decode them yourself. Choose this if you want to write your own decoder, archive bytes 1:1, or debug.

**Caveats for binary-sourced GPS in `formatted`** — i.e. everything a current device sends:

- `heading_deg` is always `0`. The binary position record carries no heading field; it is a constant, not a measurement.
- `satellites`, `hdop`, `battery_v` and `altitude_baro_m` are **absent**. They only ever came from the legacy JSON path. Read satellite count and battery from the accompanying `status` packets instead.
- `speed_ms` / `speed_kmh` are **3D** speed (including the vertical component), not ground speed.
- `fw_version` is `"major.minor"` — the patch component is dropped on the wire. Do **not** use it for feature detection; it was not bumped when cadence was added.
- `device_mode` is the device's **current cached** mode, not its mode at the moment the packet was captured.
- `cadence_spm` is present when the device sent a 25-byte position record. See [Paddling cadence](#paddling-cadence).

The full wire-level frame format for binary topics (sync `0x4D 0x41`, 8-bit type, LE16 length, payload, Fletcher-8 checksum) — including the GPS / Status / MARK record layouts, batch encoding, multi-frame concatenation rules, and a byte-by-byte worked example — is documented in **[Binary protocol reference (binary-protocol.md)](binary-protocol.md)**.

> If you write your own `raw` decoder: **never hardcode the position-record size.** For a GPS batch, derive it per frame as `(payload_length − 1) / count`; for a MARK frame the payload *is* one bare record, so its length is the size. Records only ever grow by appending trailing fields — the cadence byte is the most recent one — and a hardcoded stride silently corrupts every position after the first in a batch.

---

## Rate limits & error envelope

- **Default:** 60 requests/min per key, counted **per endpoint** (each route has its own bucket — 60 `GET …/last` and 60 `POST …/commands` in the same minute are both fine). Per-key custom limit available on request.
- **Headers:** every `/v1` response carries `x-ratelimit-limit`, `x-ratelimit-remaining` and `x-ratelimit-reset` (seconds) — read them instead of guessing.
- **What counts:** only authenticated requests. A `401` is never charged to a key. The `_all` fan-out endpoints count as one request regardless of how many devices they reach. There is no additional per-IP limit on authenticated `/v1` traffic, so several integrators behind one NAT do not share a bucket.
- **WebSocket / SSE:** the initial handshake counts; subsequent frames don't.
- **Limit hit:** HTTP `429 Too Many Requests` with `Retry-After` header **and** body:
  ```json
  { "statusCode": 429,
    "success": false,
    "error": { "code": "rate_limited",
               "message": "Rate limit exceeded (60 req/min)",
               "details": { "retry_after_ms": 12345 } } }
  ```

The complete error-code set:

| Code             | HTTP | Meaning                                                              |
|------------------|------|----------------------------------------------------------------------|
| `bad_request`    | 400  | Body / query failed validation — also unknown command, unknown topic, invalid `mode_id` |
| `unauthorized`   | 401  | Missing / bad / revoked / expired key                                |
| `forbidden`      | 403  | Authenticated but key missing the required scope                     |
| `not_found`      | 404  | No such device or endpoint                                           |
| `mode_disabled`  | 409  | Requested data is off in the device's current mode preset            |
| `rate_limited`   | 429  | Per-key rate limit hit                                               |
| `internal_error` | 500  | Server-side failure                                                  |

Note that an **unknown topic is a 400 `bad_request`**, not a 404, and there is no `invalid_request` code — switch on `bad_request`.

Nearly every error response has `success: false` and an `error` **object** with a `code` you can switch on. The human `error.message` is for logs — don't parse it.

> **One shape inconsistency to code around.** The authentication layer's 401s return `error` as a **flat string**, not the object: `{"success": false, "error": "Invalid API key"}` (also `"Missing Bearer token"`, `"Invalid API key format"`, `"API key revoked"`, `"API key expired"`, `"User account disabled"`, `"Unauthorized"`). On those responses `error.code` is `undefined`. Since 401 is the first thing a new integration hits, handle both shapes: `typeof body.error === 'string' ? body.error : body.error.code`.

---

## Device modes

Each tracker runs in one of **18 mode presets**. A preset controls exactly two things: how often positions leave the device (on an interval, or batched out of an on-device buffer) and how often STATUS frames go out. Switch modes from the API with `PUT /v1/devices/:iccid/mode` (requires `write:mode`); read the current mode with `GET /v1/devices/:iccid/mode`; get the full catalog at `GET /v1/devices/_modes`.

What a preset does **not** control:

- **The GNSS sampling rate.** That is a separate persisted setting, changed with the `set_gps_rate` command (1, 4, 5, 10 or 20 Hz; **default 10 Hz**). 20 Hz additionally requires the irreversible `gps_otp` write and is refused otherwise.
- **IMU streaming.** All 18 presets have it off, and no firmware path emits IMU frames at all.

The 18 presets split into two families:

- **Standard modes** — the device publishes its **most recent** position on a fixed interval, one MQTT message per publish. Only mode 2 (interval 0) publishes on every GNSS fix; every other standard preset discards the fixes between sends — mode 1 sends 1/s while GNSS samples at 10 Hz, so 9 of 10 fixes never leave the device. Use a logging mode if you need every fix.
- **Logging modes** — the device samples GPS at 10 Hz into an on-device ring buffer (4000 positions, 100 000 bytes) and sends them in **batches** over a compact binary frame on the `bin` topic. Pick these when you care about data efficiency, route fidelity, or surviving connectivity gaps. Each drain publishes exactly **one** frame of at most **40** positions — `(1024 − 7 − 1) / 25`, against the LEXI-R422 modem's 1024 B MQTT payload limit. Anything beyond 40 stays in the ring buffer until the next drain; the device never fires back-to-back packets to flush a long interval.

### Standard modes

| ID | Label             | Rate                | Status interval | Best for                                            |
|----|-------------------|---------------------|-----------------|-----------------------------------------------------|
| 1  | Default           | 1 position/s        | 30 s            | Day-to-day tracking, balanced data usage            |
| 2  | High Performance  | Every fix (10 Hz¹)  | 30 s            | Maximum real-time, racing/track-day, demos          |
| 5  | Fast 500ms        | 2 positions/s       | 30 s            | Brisk real-time with moderate data usage            |
| 6  | Fast 250ms        | 4 positions/s       | 30 s            | High real-time, motorsport, drone follow            |
| 7  | Every 2s          | 1 position/2s       | 30 s            | Relaxed live tracking, reduced data usage           |
| 8  | Every 3s          | 1 position/3s       | 30 s            | Relaxed live tracking, reduced data usage           |
| 9  | Every 5s          | 1 position/5s       | 30 s            | Slow-moving assets, low data usage                  |
| 3  | Slow              | 1 position/10s      | 60 s            | Long-haul, conservation of SIM data                 |
| 4  | Very Slow         | 1 position/min      | 180 s           | Stationary assets, ultra-low data usage             |
| 10 | Ultra Slow        | 1 position/10min    | 10 min          | Dormant assets, beacon-style check-ins              |

¹ The "Rate" column is the **publish** interval. Sampling is always at the configured GNSS rate (10 Hz by default) — modes other than 2 simply publish the latest fix and drop the rest.

### Logging modes

Every fix is buffered and sent as a batched binary frame on `t/{ICCID}/bin` → arrives via `/v1/stream` (or the `bin` snapshot) decoded into `gps_batch` packets. Batch sizes below assume the default 10 Hz sampling rate.

| ID | Label        | Drain interval     | Approx batch size          | Best for                                             |
|----|--------------|--------------------|----------------------------|------------------------------------------------------|
| 20 | Log HP       | ASAP (every fix)   | 1 position                 | Highest fidelity, near-real-time, low overhead       |
| 27 | Log 250ms    | 250 ms             | ~2–3 positions             | High fidelity with batching savings                  |
| 28 | Log 500ms    | 500 ms             | ~5 positions               | Smooth track, moderate data usage                    |
| 21 | Log 1s       | 1 s                | ~10 positions              | Common general-purpose logging                       |
| 22 | Log 2s       | 2 s                | ~20 positions              | Cycling / running / hiking                           |
| 23 | Log 3s       | 3 s                | ~30 positions              | Long routes, fewer packets                           |
| 25 | Log 5s       | 5 s                | 40 of ~50 buffered¹        | Drives, long sessions                                |
| 26 | Log 10s      | 10 s               | 40 of ~100 buffered¹       | Maximum data savings, long routes                    |

¹ The firmware drains **one** frame per interval, capped at 40 records. At 10 Hz modes 25 and 26 buffer more than a single frame can carry, so the surplus waits for the next drain instead of going out as extra packets.

> Mode IDs are deliberately sparse (gaps at 11–19, 24, 29+) so new presets can be added later without renumbering. **24 is not a valid ID** — the 18 real IDs are 1–10 and 20, 21, 22, 23, 25, 26, 27, 28.

### How modes look on the wire

Both families publish **exclusively** on `t/{ICCID}/bin`, as binary frames. Via `/v1` you therefore always see `topic: "bin"`; only `payload.kind` differs:

| Mode family | Topic           | What you see via `/v1/stream` (formatted)                                                        |
|-------------|-----------------|---------------------------------------------------------------------------------------------------|
| Standard    | `t/{ICCID}/bin` | `payload.kind = "gps_batch"`, `payload.data` an array of exactly **one** position — a standard-mode publish is a 1-record batch |
| Logging     | `t/{ICCID}/bin` | `payload.kind = "gps_batch"`, `payload.data` an array of **N** positions                            |

A batch arrives as **one** packet with an array inside — the server does not split it into one delivery per position. Iterate `payload.data` yourself. (The Python example below prints `#01/06`, `#02/06`, … markers because *it* expands the array client-side.)

Periodic STATUS frames and MARK waypoints ride on the **same** `bin` topic, as `payload.kind = "status"` and `payload.kind = "mark"` (both with a single object in `payload.data`). One MQTT message may carry several concatenated frames, each delivered as its own packet — in stationary mode the device sends a STATUS and a 1-record GPS frame in a single publish.

- **MARK** is device-initiated only: a double-click of the button. There is no `mark` command. It is published at QoS 1, and only when the device has a 2D-or-better fix with horizontal accuracy ≤ 100 m — otherwise the mark is dropped.
- The legacy `t/{ICCID}/gps`, `/status` and `/event` topics have **no publisher** in current firmware. The backend still subscribes to them for pre-binary back-compat, so they remain valid `/v1` topic names, but nothing populates them.
- Command acknowledgements and OTA / IMU-upload progress arrive as JSON on `t/{ICCID}/response` (`payload.kind = "response"`).

### Switching modes

```bash
# Read current
curl -H "Authorization: Bearer $KEY" \
  https://api.motionapi.pro/v1/devices/8988228066680471572/mode

# Switch to "Log 1s" (mode 21). The body field is `mode_id` — `mode` is a 400.
curl -X PUT -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"mode_id": 21}' \
  https://api.motionapi.pro/v1/devices/8988228066680471572/mode
```

The 200 response is `{ msg_id, requested_mode: {…}, note }` — note the wording: the backend has **published** the change, it has not confirmed it. The device-side switch is asynchronous; the next STATUS packet is what tells you the new mode is live. Once applied it is persisted in device NVS, so it survives reboot / power-cycle.

---

## Paddling cadence

Trackers can report **paddling cadence** alongside each position, for kayak and canoe use. It is computed on the device from the built-in 6-axis IMU (an 8-second rolling window, updated twice a second) and rides on the wire as a single field, `cadence_spm`, on every position record.

**Cadence is off by default.** You have to turn it on per device by selecting a sport profile.

### Sport profiles

| ID | Profile        | Effect                                                        |
|----|----------------|---------------------------------------------------------------|
| 0  | None           | **Default.** Cadence detector off; `cadence_spm` is always `0` |
| 1  | Canoe          | Cadence on, single-blade convention forced                     |
| 2  | Kayak          | Cadence on, double-blade convention forced                     |
| 3  | Paddle (auto)  | Cadence on, the device classifies kayak vs canoe itself        |

The profile is persisted on the device, so it survives reboot and power-cycle.

### How to enable it

Send `{"cmd": "config", "sport": 1|2|3}` on the device's config downlink.

> **Known limitation: you cannot do this through `/v1` today.** `POST /v1/devices/:iccid/commands` rejects `cmd: "config"` outright, and `PUT /v1/devices/:iccid/mode` only ever sends the mode field. There is no `/v1` route for the sport profile. The only place it can be set right now is the **user panel** ("Set Sport" on the device command panel). If you need this on the API, tell us.

Two more things worth knowing:

- **There is no read-back.** Nothing on the wire reports the active sport profile — the STATUS record carries the mode preset, not the sport — and the command acknowledgement carries no data field, so the applied value is not echoed either. A UI has to remember the last value it sent.
- **`{"cmd":"config","boat":N}` is not the switch.** It is a legacy, transient classifier override (0 auto / 1 canoe / 2 kayak), it is not persisted, and it does **not** enable cadence. Only a sport profile of 1, 2 or 3 starts the detector.

### Reading the value

`cadence_spm` is **strokes per minute, counting every blade entry** — the sprint-kayak convention. Converting to full stroke cycles depends on the boat:

| Boat  | Full stroke cycles per minute |
|-------|-------------------------------|
| Kayak | `cadence_spm / 2` (left + right blade per cycle) |
| Canoe | `cadence_spm` (one blade per cycle)              |

Non-zero readings are clamped to the range **28–165 spm**.

**`cadence_spm == 0` is ambiguous.** It means *any* of the following, with no way to tell them apart:

1. The sport profile is `None` — i.e. cadence is off, which is the default state of every device.
2. The device is not currently paddling.
3. The IMU is unavailable or failed to read.

Treat `0` as "no cadence reading", never as "cadence measured at zero".

Only `cadence_spm` reaches the wire. The detector's internal signals — confidence, a paddling flag, stroke counts, blade side, cycles-per-minute — never leave the device, so do not build against them.

### Where you can and cannot get it

| Path                                          | Cadence available?                                                        |
|-----------------------------------------------|---------------------------------------------------------------------------|
| `format=formatted` (REST snapshots, WS, SSE)  | **Yes** — as `cadence_spm` on each position, and on MARK waypoints        |
| `format=raw`                                  | **Yes** — you decode it out of the record yourself                        |
| Stored history                                | No — cadence is not persisted anywhere; it exists only on live packets    |
| Setting the sport profile via `/v1`           | No — see the limitation above                                             |

In `formatted`, the field is **absent** (not `0`) when a device sent a pre-cadence, 24-byte position record. Absent means "this firmware has no cadence field"; `0` means one of the three cases above. Do **not** try to detect the difference from `fw_version` — the wire carries only major.minor, and the version was not bumped when cadence was added.

The exact byte position, record size and version-detection rule are in the GPS-record section of the **[Binary protocol reference](binary-protocol.md)**.

---

## Reference examples

We ship two open examples — pick whichever matches your stack.

### Example 1 — Browser API Explorer

A single-file HTML demo of every `/v1` surface (snapshots, streams, commands, mode switching). Zero build, no dependencies — open the file and go.

- **Repo:** <https://github.com/MotionApi/example-api-explorer>
- **Live:** <https://motionapi.github.io/example-api-explorer/>
- **What's inside:** one `index.html`, ~56 KB, vanilla JS — copy-paste-ready.

**How to use:**

1. Open the live URL or clone the repo and `open index.html`.
2. Paste your API key into the header field and click **Connect**.

   ![Explorer header — API URL + key fields](screenshots/explorer-connect.png)

3. Pick a device from the left sidebar. The Snapshot tab opens automatically with the latest packet on every topic the device speaks.
4. Use the tabs:
   - **Snapshot** — latest packet on every topic, toggle `formatted` ↔ `raw`.
   - **Live Stream** — `wss://…/v1/stream` with per-topic filter and format selector. Tail-scrolling log of every packet:

     ![Explorer Live Stream tab — real-time WebSocket data](screenshots/explorer-live-stream.png)

   - **Commands** — quick buttons that hit `POST /v1/devices/:iccid/commands`:

     ![Explorer Commands tab — quick action buttons](screenshots/explorer-commands-tab.png)

   - **Mode** — read + change the mode preset (`PUT /v1/devices/:iccid/mode`):

     ![Explorer Mode tab — switch between the 18 mode presets](screenshots/explorer-mode-tab.png)

Required scopes follow the table in [§ Endpoints](#endpoints--rest). The explorer surfaces backend 401/403 errors inline so you know exactly which scope you forgot.

**When to use:** as a sanity check after issuing a key, to demo the system, or as a starting point for a custom dashboard (fork the HTML, strip what you don't need).

### Example 2 — Python stream logger

A single-file Python script that connects to `/v1/stream` for one device and appends a human-readable, one-line-per-packet log file. Positions, status frames and command responses each get a timestamped row. Useful for field testing, drive runs, batch analysis, or as a starting point for any "background process that needs every packet" integration.

- **Repo:** <https://github.com/MotionApi/example-api-python-logger>
- **What's inside:** one `motionapi_logger.py` (~26 KB), one dependency (`websockets>=12,<14`), Python 3.9+.

**Required scope:** `read:stream` (plus `read:devices` for nicer startup output).

**Run:**

```bash
git clone https://github.com/MotionApi/example-api-python-logger.git
cd example-api-python-logger
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python motionapi_logger.py \
  --api-key mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6 \
  --device 8988228066680471572
```

Or via environment variables (drop them in `.env` next to the script):

```bash
export MOTIONAPI_KEY=mak_live_…
export MOTIONAPI_DEVICE=8988228066680471572
python motionapi_logger.py
```

**Log output looks like** (verbatim from a real capture — `device.log` in the repo):

```
# motionapi-logger session  start=2026-05-25 16:57:57.125  backend=https://api.motionapi.pro  device=8988228066680471572  topics=gps,status,event,imu,response,bin
2026-05-25 16:57:57.298  -                             ready      ws_connected scopes=read:devices,read:stream,read:last,read:positions,read:status,read:events,read:imu,write:commands,write:mode,write:ota authorized_devices=1
2026-05-25 16:57:57.317  -                             subscribed topics=gps,status,event,imu,response,bin format=formatted authorized=1 denied=0
2026-05-25 16:57:57.671  fix=2026-05-25T14:57:57.000Z  gps_batch  seq=1 #01/06 lat=52.102362 lon=21.062391 speed=1.7km/h hdg=0.0° alt=109.1m
2026-05-25 16:57:57.671  fix=2026-05-25T14:57:57.100Z  gps_batch  seq=1 #02/06 lat=52.102360 lon=21.062390 speed=1.1km/h hdg=0.0° alt=109.2m
2026-05-25 16:57:59.910  -                             status     seq=4 batt=3.70V rssi=-69dBm rsrp=-99dBm cell=260-3#46409998 tac=58160 mode=Log 500ms(28) moving=no fw=0.1
```

Each row carries **two timestamps** — local wall-clock receive, then the device-side clock: `fix=` on `gps_batch` / `mark` rows, `ts=` on `status` rows, and `-` where there is no device clock at all (connection-state rows, command responses). So you can spot delivery lag at a glance. The capture above predates the `ts=` column, so its `status` row still shows `-`.

Note what real output does **not** contain: every position row is `gps_batch`, never a bare `gps` (that kind only comes from the legacy JSON topic no shipping device uses), and position rows carry no `sats=` or `batt=` — those come from the `status` rows instead, because the binary position record does not carry them. `hdg=0.0°` on every row is the constant described in [§ Packet formats](#packet-formats--raw-vs-formatted), not a measurement.

**When to use:** drive-test logging, headless field captures, batch / offline pipelines, anywhere a small, deps-light Python process is more natural than a browser tab. Copy the file, strip what you don't need; it's MIT-licensed.

---

## Questions / feedback

Reach out via <https://motionapi.pro>.
