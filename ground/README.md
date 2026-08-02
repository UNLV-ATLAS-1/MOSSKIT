# Ground Station Software — ATLAS-1 / MOSS Kit

This folder holds everything that runs **on the ground Pi**: the receiver that
listens on the radio, and the web dashboard that displays what it hears. It is
the mirror image of the flight Pi software.

If you are setting up the Pi from scratch (flashing the SD card, enabling serial,
configuring nginx and the hotspot), start with the MOSS Kit documentation site
and come back here once the Pi boots and serves a page.

---

## Overview of rx_to_latest

`rx_to_latest.py` sits in a loop reading bursts of bytes from the LoRa module. It
strips the radio's own header, hunts for CRC-checked TM frames in the byte
stream, reassembles them, decodes them with `tm_schema.unpack()`, and writes two
files: `latest.json` (the newest reading) and `logs/history.jsonl` (every
reading). nginx serves those files as static content. `index.html` polls
`latest.json` four times a second and redraws six live charts, a panel of tiles,
and a scrolling table. There is no backend — the dashboard reads files, nothing
more.

```
radio bytes → strip radio header → find TM frames (CRC) → reassemble
            → tm_schema.unpack() → flatten → latest.json + history.jsonl
            → nginx → browser
```

---

## Files at a glance

| File | Role |
|---|---|
| `rx_to_latest.py` | The receiver. Radio in, JSON files out. Run this during a flight. |
| `tm_schema.py` | The binary wire format. `pack()` / `unpack()`. **Must match flight exactly.** |
| `protocol_tm.py` | Framing: sync bytes, IDs, CRC16. **Must match flight exactly.** |
| `sx126x.py` | Driver for the Waveshare SX126x UART LoRa module. |
| `index.html` | The dashboard. Single file, no build step, no framework. |
| `chart_umd_min.js` | Chart.js, vendored locally so the dashboard works with no internet. |
| `fake_telemetry.py` | Writes pretend telemetry so you can test the dashboard with no radio. |

`tm_schema.py`, `protocol_tm.py`, and `sx126x.py` are shared with the flight Pi
and must be byte-identical copies. If you edit one, copy it to the other before
the next flight, or the ground will decode garbage.

---

## Running a flight

On the ground Pi, in the ground folder:

```
python3 rx_to_latest.py
```

Then open the dashboard in a browser (on the field hotspot, that is
`http://192.168.4.1/`). You should see the contact badge start ticking within a
few seconds of the payload powering up.

Console output is one line per decoded packet:

```
RX OK (TM) seq=412 rssi=-97
```

Stop with Ctrl+C.

---

## `rx_to_latest.py`

### Single-instance lock

The script grabs an exclusive lock on `/tmp/rx_to_latest.lock` at import time and
exits immediately if another copy already holds it. Two receivers fighting over
`/dev/serial0` and `latest.json` is a confusing failure to debug at a launch
site, so the script refuses to be that failure.

### `TelemetryAccumulator`

The decode pipeline lives in this class, deliberately separated from files and
the radio so it can be unit-tested with handcrafted byte strings. You hand it raw
bytes and it yields fully decoded records:

```python
acc = TelemetryAccumulator()
for record in acc.feed(raw_bytes, now=time.time()):
    ...
```

It holds three pieces of state:

- **`tm_stream`** — a rolling byte buffer that frames are parsed out of. Radio
  bytes arrive as a messy stream, not neat packets, so leftover bytes carry over
  between calls.
- **`pending`** — partial multi-fragment messages keyed by `msg_id`. Fragments
  that never complete are discarded after `REASSEMBLY_TTL_S` (60 s) so partial
  data does not accumulate forever.
- **`_seen`** — a bounded ring of recently decoded `msg_id`s. The flight side
  sends every packet twice, so the same packet usually arrives twice; the second
  copy is dropped here. The window is 256, which matters because `msg_id` is only
  16-bit and wraps.

**The subtle bit in `feed()`.** `try_parse_one` can return `None` *and still
consume bytes* — that is what happens when it slides past garbage or a failed
CRC. So the loop does not stop at the first `None`. It stops only when the buffer
stops shrinking, which means what remains is a genuine incomplete frame waiting
for more bytes.

Counters (`frames_parsed`, `records_decoded`, `duplicates_dropped`,
`unpack_failures`) are kept for status and debugging. When an unpack fails, the
raw hex is printed so you can inspect the bad bytes afterward.

### `flatten_for_dashboard()` — the contract

`tm_schema.unpack()` returns a nested record. The dashboard reads flat names.
This function is where that mapping is pinned, on purpose: the binary schema's
internal layout should not leak into the UI.

| Dashboard field | Source |
|---|---|
| `seq`, `ts` | `sequence_id`, `timestamp_utc` |
| `t_c`, `p_hpa`, `rh`, `altitude_m` | `environment.*` |
| `bv`, `bp` | `power.battery_v`, `power.battery_pct` |
| `uv_raw`, `ambient_raw` | `uv_light.*` (LTR390) |
| `lux`, `ir`, `vis` | `light.*` (TSL2591) |
| `accel`, `gyro`, `ax…gz` | `imu.*` (ICM20948), both array and flat-axis forms |
| `image_captured`, `alerts` | status flags |
| `_radio`, `_flags_raw`, `schema_version` | provenance |

A field whose sensor failed comes back as `None` and is emitted as `null`. The
key is always present; the dashboard renders null as an em dash on its own.

Old v1 (22-byte) packets have no light or IMU blocks, so those fields fall back
to `None` automatically. **If you rename a field here, rename it in `index.html`
too**, or that tile silently goes blank.

### Atomic writes

Every write to `latest.json` goes to a temp file, gets flushed and fsynced, then
renamed over the target. Rename is atomic, so the dashboard always reads either
the whole old file or the whole new file — never a half-written one.

### Status states

`latest.json` does not always contain telemetry. When nothing is decoding, the
receiver writes a status object instead, and the dashboard recognizes it as
"no telemetry yet":

| Status | Meaning |
|---|---|
| `NO DATA YET` | First run, placeholder so the dashboard loads cleanly. |
| `DEMO / NO RADIO` | `sx126x` failed to import. The dashboard still runs. |
| `CONTACT (RX BYTES, decode pending)` | Bytes are arriving but nothing has decoded yet. |
| `NO CONTACT` | No bytes for more than `NO_CONTACT_AFTER_S` (30 s). |

The contact-pending writes include `rx_len` and the first 16 bytes as hex, which
is often enough to tell "the radios are talking but the schema disagrees" from
"nothing is arriving at all."

### Config that must match flight

`LORA_FREQ_MHZ` 915, `LORA_AIR_SPEED` 2400, `LORA_NET_ID` 0, `LORA_ADDR` 1,
`LORA_BUFFER_SIZE` 240, `LORA_CRYPT` 0. Since `sx126x.py` never reconfigures the
module at runtime, these values must already be set on the physical radio with
M0/M1 jumpered LOW/LOW.

---

## `index.html` — the dashboard

One file. No build step, no framework, no backend. It fetches two static paths:

- `/latest.json` — polled every 250 ms
- `/history.jsonl` — read once on load, to seed the charts so they are not empty

Chart.js is loaded from a local copy (`/chart.umd.min.js`), so the dashboard works
on a field hotspot with no internet.

### What to change

Search for `CHANGE ME`. Two cosmetic spots: the browser tab `<title>` and the big
heading. That is all.

### What not to change

Everything else. There are no usernames, hostnames, IP addresses, or folder paths
in the file. It asks for `/latest.json` from whatever server handed it the page,
so the same file works on any Pi, any username, home Wi-Fi or field hotspot. **If
the dashboard is blank, the problem is nginx's root path or the receiver not
running — not this file.**

### How it decides a packet is new

It builds a key from `_rx_utc`, falling back to `ts`, falling back to `seq`. If
the key changed and the record actually has telemetry (`seq` present plus at
least one of `t_c`, `p_hpa`, `bv`), it pushes a chart point, prepends a table
row, and updates the tiles. Otherwise it only ticks the age badge.

The badge goes green under 26 seconds, amber under 55, red beyond that — sized
around the 2-second packet cadence with room for the losses a real link has.

Six charts: temperature, humidity, pressure, lux, battery voltage, battery
percent. A seventh for battery current is stubbed out in comments if you ever
want it. The table keeps the last 60 rows.

---

## Testing without a radio

```
python3 fake_telemetry.py
```

Writes the same two files the real receiver writes, filled with realistic drifting
values (gentle sine waves plus noise: temperature wobbles, pressure falls,
altitude climbs, battery drains). It seeds 50 past packets so the charts have a
curve on load, then writes one new packet every 2 seconds.

It touches **only** `latest.json` and `logs/history.jsonl`. No radio, no sudo, no
system settings, so it cannot harm the Pi or the setup.

Open the dashboard and watch. If the charts move, the dashboard and nginx are
both fine and any flight-day problem is on the radio side. If they stay blank,
check that nginx is running and that you opened the right address.

Its field names must stay in step with `flatten_for_dashboard()` and
`index.html`. Rename in one place, rename in all three.

> **Note:** `fake_telemetry.py` prints a warning if it does not find
> `static/index.html` next to it. That check dates from the Flask layout; the
> current nginx setup serves `index.html` from the ground folder directly. The
> warning is harmless, and the script writes the files either way.

---

## Files produced

```
ground/
  latest.json          newest reading, atomically replaced (or a status object)
  logs/
    history.jsonl      one flattened JSON record per line, appended forever
```

`history.jsonl` grows without bound and is never rotated. Clear or archive it
between flights so the dashboard is not seeding charts from last month's data.

---

## Troubleshooting

| Symptom | Where to look |
|---|---|
| Dashboard blank, no tiles at all | nginx root path, or the page was not served over HTTP. |
| `NO DATA YET` forever | Receiver not running, or it exited on the lock. |
| `DEMO / NO RADIO` | `sx126x` import failed — check `RPi.GPIO` and `pyserial` are installed. |
| `NO CONTACT` | Payload not transmitting, wrong frequency, or antenna problem. |
| `CONTACT` but never `RX OK` | Radios agree, framing does not. Check `protocol_tm.py` matches. |
| `unpack failed for msg_id=…` | `tm_schema.py` differs between flight and ground. Hex is printed. |
| One tile blank, rest fine | Field name mismatch between `flatten_for_dashboard()` and `index.html`. |
| Charts move with fake data, not real | Problem is radio side, not dashboard side. |
| "already running (lock held)" | An older receiver is still up. `pkill -f rx_to_latest` and retry. |

---

## Preflight checklist

- [ ] `tm_schema.py` and `protocol_tm.py` identical on flight and ground Pis
- [ ] M0/M1 jumpers LOW/LOW; both radios at 915 MHz, 2400 bps, net ID 0
- [ ] nginx running and serving the ground folder
- [ ] `chart.umd.min.js` reachable at `/chart.umd.min.js` (offline check: airplane mode)
- [ ] `fake_telemetry.py` makes the charts move
- [ ] Old `history.jsonl` archived or cleared
- [ ] Hotspot up (SSID `UNLVCUBE1-GS`, dashboard at `192.168.4.1`)
- [ ] Ground Pi clock is correct — a bad clock is what produced the 5 595-second
      offset on Launch 1, and it corrupts every receive timestamp
