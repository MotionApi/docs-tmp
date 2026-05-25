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
10. [Reference examples](#reference-examples)
    - [Browser — API Explorer](#example-1--browser-api-explorer)
    - [Python — Stream logger](#example-2--python-stream-logger)
11. [Binary protocol reference](binary-protocol.md) — wire format for `format=raw`

---

## What you can do with the API

MotionApi sells GPS+LTE-M tracker hardware. The `/v1` API lets you build on top of those trackers without touching MQTT directly:

- **List your devices** and their last-known position / battery / signal.
- **Read snapshots** — the most recent packet on every topic per device (`gps`, `status`, `event`, `imu`, `bin`).
- **Subscribe to live data** over WebSocket or Server-Sent Events.
- **Send commands** (identify, buzzer, LED, reboot, MARK).
- **Change the device mode preset** (14 presets — standard 1–6, logging 20–28).

All real-time data flows through the backend; you never need MQTT credentials.

**Base URL:** `https://api.motionapi.pro`

---

## Getting an API key

1. Sign in to the **user panel** at <https://app.motionapi.pro>.
2. From the left nav choose **API Access**.

   ![API Access page in the user panel](screenshots/api-access-page.png)

3. Click **Create Key** in the top-right.
4. In the modal, fill in:
   - **Name** — anything that helps you remember which integration uses it.
   - **Environment** — `live` (production) or `test`.
   - **Scopes** — pick only what you need. Defaults to `read:devices`, `read:stream`, `read:last`.
   - **Devices** — leave empty for "all my devices" or pick a subset.
   - **Rate limit** — leave empty to use the default 60 req/min; raise it if you have an approved use case.
   - **Expires at** — optional ISO date for an auto-expiring key.

   ![Create Key modal with scope checkboxes](screenshots/create-key-modal.png)

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

| Scope             | Grants                                                                  |
|-------------------|-------------------------------------------------------------------------|
| `read:devices`    | List devices, read metadata, read current mode                          |
| `read:last`       | Read the latest cached packet (`/last`, `/last/:topic`)                 |
| `read:stream`     | Subscribe to WebSocket / SSE real-time streams                          |
| `read:positions`  | Historical GPS query                                                    |
| `read:status`     | Historical device-status query                                          |
| `read:events`     | Historical events query                                                 |
| `read:imu`        | Raw IMU data                                                            |
| `write:commands`  | Send commands (buzzer, LED, identify, reboot, MARK)                     |
| `write:mode`      | Change device mode preset                                               |
| `write:ota`       | Upload firmware (off by default — contact us to enable)                 |

Principle of least privilege — for a dashboard that just shows positions you only need `read:devices` + `read:last` + `read:stream`.

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
| GET    | `/v1/devices/_modes`                         | (none)         | Static catalog of all 14 mode presets                |

### Snapshots — last cached packet per topic

| Method | Path                                         | Scope       | Description                                                       |
|--------|----------------------------------------------|-------------|-------------------------------------------------------------------|
| GET    | `/v1/devices/:iccid/last`                    | `read:last` | Latest packet on every topic the device has spoken                |
| GET    | `/v1/devices/:iccid/last/:topic`             | `read:last` | Latest packet on one specific topic (`gps`, `status`, `bin`, …)   |

Add `?format=raw` to get the bytes the device actually sent (base64 + frame metadata), or `?format=formatted` for human-readable JSON (default).

### Commands

| Method | Path                                         | Scope             | Description                                              |
|--------|----------------------------------------------|-------------------|----------------------------------------------------------|
| POST   | `/v1/devices/:iccid/commands`                | `write:commands`  | Send `identify` / `buzzer` / `led` / `status` / `reboot` |

Body example:

```json
{ "cmd": "buzzer", "params": { "duration_ms": 500, "frequency": 4000 } }
```

Mode changes are **not** accepted here — use `PUT /v1/devices/:iccid/mode` instead.

### Curl quick-start

```bash
# Replace the bearer token with your real key.
KEY="mak_live_abcd1234_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"

# List devices
curl -H "Authorization: Bearer $KEY" https://api.motionapi.pro/v1/devices

# Latest GPS snapshot for one device
curl -H "Authorization: Bearer $KEY" \
  'https://api.motionapi.pro/v1/devices/8988228066680471572/last/gps?format=formatted'

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
2. Server sends a `{"type":"ready", "authorized_devices":[…]}` frame so you know what you may subscribe to.
3. You send a subscribe message:
   ```json
   {
     "type": "subscribe",
     "devices": ["8988228066680471572"],
     "topics": ["gps", "status", "event", "bin"],
     "format": "formatted"
   }
   ```
4. Server responds with `{"type":"subscribed", "devices_authorized":[…], "devices_denied":[…], "topics":[…], "format":"…"}`.
5. Server pushes `{"type":"packet", "iccid":"…", "topic":"gps", "received_at":"…", "seq":…, "payload":{…}}` per packet, plus periodic `{"type":"heartbeat"}` frames.

If you send `"format":"raw"` packets include `raw.buffer_b64` + `raw.frames[]` (for binary topics like `bin`) instead of `payload`.

You may re-send `subscribe` at any time to change filter or format — the server will replace the previous subscription on the same socket.

### Server-Sent Events — `GET /v1/devices/:iccid/stream`

Scope: `read:stream`. One-way (server → you), one device per connection. Works through stricter corporate proxies that block WS.

```bash
curl -N "https://api.motionapi.pro/v1/devices/8988228066680471572/stream?topics=gps,status&format=formatted" \
  --header "Authorization: Bearer $KEY"
```

Each event is a standard `data: { …packet… }\n\n` SSE chunk. Use the `format=raw` querystring for raw bytes; `topics=…` is a CSV filter.

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
- **`raw`** — exact bytes received from the device. Binary frames (`bin` topic, currently the production path for GPS) come as base64 plus per-frame metadata so you can decode them yourself. Choose this if you want to write your own decoder, archive bytes 1:1, or debug.

The full wire-level frame format for binary topics (sync `0x4D 0x41`, 8-bit type, LE16 length, payload, Fletcher-8 checksum) — including the GPS / Status / IMU / MARK record layouts, batch encoding, multi-frame concatenation rules, and a byte-by-byte worked example — is documented in **[Binary protocol reference (binary-protocol.md)](binary-protocol.md)**.

---

## Rate limits & error envelope

- **Default:** 60 requests/min per key. Per-key custom limit available on request.
- **WebSocket / SSE:** the initial handshake counts; subsequent frames don't.
- **Limit hit:** HTTP `429 Too Many Requests` with `Retry-After` header **and** body:
  ```json
  { "success": false,
    "error": { "code": "rate_limited",
               "message": "Rate limit exceeded (60 req/min)",
               "details": { "retry_after_ms": 12345 } } }
  ```

Other common error codes:

| Code              | HTTP | Meaning                                            |
|-------------------|------|----------------------------------------------------|
| `unauthorized`    | 401  | Missing / bad / revoked / expired key              |
| `forbidden`       | 403  | Authenticated but key missing required scope       |
| `not_found`       | 404  | No such device, topic, or endpoint                 |
| `invalid_request` | 400  | Body / query failed schema validation              |
| `rate_limited`    | 429  | Per-key rate limit hit                             |

Every error response has `success: false` and an `error.code` you can switch on. The human `error.message` is for logs — don't parse it.

---

## Device modes

Each tracker runs in one of **14 mode presets**. The mode controls how often GPS is read, whether the device sends each fix immediately or buffers them, how often status frames go out, and whether IMU streaming is on. Switch modes from the API with `PUT /v1/devices/:iccid/mode` (requires `write:mode`); read the current mode with `GET /v1/devices/:iccid/mode`; get the full catalog at `GET /v1/devices/_modes`.

The 14 presets split into two families:

- **Standard modes** — the device sends **one position per fix, immediately**. Pick these for real-time tracking where each update should leave the device as fast as possible. Data on the wire is one MQTT message per fix.
- **Logging modes** — the device samples GPS at 10 Hz into an on-device ring buffer (4000 positions) and sends them in **batches** over a compact binary frame on the `bin` topic. Pick these when you care about data efficiency, route fidelity, or surviving connectivity gaps. A 10-second batch arrives as ~100 positions across 3 MQTT packets (max 42 positions/packet — SARA-R422's 1024 B MQTT limit).

### Standard modes

| ID | Label             | Rate                | Status interval | Best for                                            |
|----|-------------------|---------------------|-----------------|-----------------------------------------------------|
| 1  | Default           | 1 position/s        | 30 s            | Day-to-day tracking, balanced data usage            |
| 2  | High Performance  | 10 Hz (every fix)   | 30 s            | Maximum real-time, racing/track-day, demos          |
| 5  | Fast 500ms        | 2 positions/s       | 30 s            | Brisk real-time with moderate data usage            |
| 6  | Fast 250ms        | 4 positions/s       | 30 s            | High real-time, motorsport, drone follow            |
| 3  | Slow              | 1 position/10s      | 60 s            | Long-haul, conservation of SIM data                 |
| 4  | Very Slow         | 1 position/min      | 180 s           | Stationary assets, ultra-low data usage             |

### Logging modes

10 Hz GPS sampling, batched binary send over `t/{ICCID}/bin` → arrives via `/v1/stream` (or `bin` snapshot) decoded into individual `gps_batch` packets.

| ID | Label        | Drain interval     | Approx batch size           | Best for                                              |
|----|--------------|--------------------|-----------------------------|-------------------------------------------------------|
| 20 | Log HP       | ASAP (every fix)   | 1 position                  | Highest fidelity, near-real-time, low overhead       |
| 27 | Log 250ms    | 250 ms             | ~2–3 positions              | High fidelity with batching savings                  |
| 28 | Log 500ms    | 500 ms             | ~5 positions                | Smooth track, moderate data usage                    |
| 21 | Log 1s       | 1 s                | ~10 positions               | Common general-purpose logging                       |
| 22 | Log 2s       | 2 s                | ~20 positions                | Cycling / running / hiking                          |
| 23 | Log 3s       | 3 s                | ~30 positions                | Long routes, fewer packets                          |
| 25 | Log 5s       | 5 s                | ~50 positions (2 packets)    | Drives, long sessions                               |
| 26 | Log 10s     | 10 s               | ~100 positions (3 packets)   | Maximum data savings, long routes                   |

> Mode IDs are deliberately sparse (gaps at 7–19, 24, 29+) so new presets can be added later without renumbering.

### How modes look on the wire

| Mode family | Topic the device publishes to | What you see via `/v1/stream` (formatted) |
|---|---|---|
| Standard    | `t/{ICCID}/gps`               | `payload.kind = "gps"` — one packet per fix         |
| Logging     | `t/{ICCID}/bin`               | `payload.kind = "gps_batch"` — N positions inside one frame, server expands each into its own delivery (you'll see `#01/05 … #02/05 …` markers per batch) |

Both families also publish `t/{ICCID}/status` periodically and `t/{ICCID}/event` on motion-start / fall / button press / MARK.

### Switching modes

```bash
# Read current
curl -H "Authorization: Bearer $KEY" \
  https://api.motionapi.pro/v1/devices/8988228066680471572/mode

# Switch to "Log 1s" (mode 21)
curl -X PUT -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"mode": 21}' \
  https://api.motionapi.pro/v1/devices/8988228066680471572/mode
```

Mode changes are delivered via downlink MQTT and applied on the device immediately. Persisted in device NVS, so it survives reboot/power-cycle.

---

## Reference examples

We ship two open examples — pick whichever matches your stack.

### Example 1 — Browser API Explorer

A single-file HTML demo of every `/v1` surface (snapshots, streams, commands, mode switching). Zero build, no dependencies — open the file and go.

- **Repo:** <https://github.com/MotionApi/example-api-explorer>
- **Live:** <https://motionapi.github.io/example-api-explorer/>
- **What's inside:** one `index.html`, ~38 KB, vanilla JS — copy-paste-ready.

**How to use:**

1. Open the live URL or clone the repo and `open index.html`.
2. Paste your API key into the header field and click **Connect**.

   ![Explorer header — API URL + key fields](screenshots/explorer-connect.png)

3. Pick a device from the left sidebar. The Snapshot tab opens automatically with the latest packet on every topic the device speaks.

   ![Explorer Snapshot tab — device list on the left, last packets per topic on the right](screenshots/explorer-snapshot-tab.png)

4. Use the tabs:
   - **Snapshot** — latest packet on every topic, toggle `formatted` ↔ `raw` (shown above).
   - **Live Stream** — `wss://…/v1/stream` with per-topic filter and format selector. Tail-scrolling log of every packet:

     ![Explorer Live Stream tab — real-time WebSocket data](screenshots/explorer-live-stream.png)

   - **Commands** — quick buttons that hit `POST /v1/devices/:iccid/commands`:

     ![Explorer Commands tab — quick action buttons](screenshots/explorer-commands-tab.png)

   - **Mode** — read + change the mode preset (`PUT /v1/devices/:iccid/mode`):

     ![Explorer Mode tab — switch between the 14 mode presets](screenshots/explorer-mode-tab.png)

Required scopes follow the table in [§ Endpoints](#endpoints--rest). The explorer surfaces backend 401/403 errors inline so you know exactly which scope you forgot.

**When to use:** as a sanity check after issuing a key, to demo the system, or as a starting point for a custom dashboard (fork the HTML, strip what you don't need).

### Example 2 — Python stream logger

A single-file Python script that connects to `/v1/stream` for one device and appends a human-readable, one-line-per-packet log file. GPS positions, status frames, events, IMU batches, and command responses each get a timestamped row. Useful for field testing, drive runs, batch analysis, or as a starting point for any "background process that needs every packet" integration.

- **Repo:** <https://github.com/MotionApi/example-api-python-logger>
- **What's inside:** one `motionapi_logger.py` (~19 KB), one dependency (`websockets`), Python 3.9+.

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

**Log output looks like:**

```
# motionapi-logger session  start=2026-05-25 14:30:00.001  backend=https://api.motionapi.pro  device=8931… topics=gps,status,event,imu,response,bin
2026-05-25 14:30:00.118  -                         ready      ws_connected scopes=read:stream,read:devices authorized_devices=3
2026-05-25 14:30:01.503  2026-05-25T12:30:01.000Z  gps        seq=42 lat=52.227762 lon=21.011510 speed=0.0km/h hdg=0.0° alt=118.4m sats=10 batt=4.05V
2026-05-25 14:30:01.612  -                         status     seq=43 batt=4.05V rssi=-67dBm rsrp=-87dBm cell=260-1#12345 mode=Default(1) moving=no
2026-05-25 14:30:30.701  2026-05-25T12:30:28.000Z  gps_batch  seq=46 #01/05 lat=52.227800 lon=21.011520 speed=12.4km/h …
2026-05-25 14:30:30.701  2026-05-25T12:30:29.000Z  gps_batch  seq=46 #02/05 lat=52.227830 lon=21.011540 speed=13.1km/h …
```

Each row carries **two timestamps** — local wall-clock receive and the device-side timestamp (`-` when there is none, e.g. for `status`) — so you can spot delivery lag at a glance.

**When to use:** drive-test logging, headless field captures, batch / offline pipelines, anywhere a small, deps-light Python process is more natural than a browser tab. Copy the file, strip what you don't need; it's MIT-licensed.

---

## Questions / feedback

Reach out via <https://motionapi.pro>.
