# Flight Pi Software — ATLAS-1 / MOSS Kit

This folder holds everything that runs **on the payload** during a flight: the
Raspberry Pi that rides the balloon, reads the sensors, writes to the SD card,
and transmits a compact copy of each reading to the ground over LoRa.

If you are setting up a Pi from scratch (flashing the SD card, enabling I2C and
serial, wiring the sensors), start with the MOSS Kit documentation site
(`MOSS-docs.html`) and come back here once the Pi boots and `i2cdetect` sees
your sensors.

---

## Overview of flight

`flight.py` runs a loop every 2 seconds. Each pass it reads every sensor,
assembles one JSON record, appends that record to an SD card log, then squeezes
the important fields into a 50-byte binary packet and transmits it twice. The
camera runs on its own background thread so a slow photo never stalls the loop.
Two rules govern the whole design:

1. **Nothing crashes the flight.** Every sensor and radio call is wrapped. A
   failure is recorded in the record's `errors` field and the loop keeps going.
   A dead sensor must not silence the working ones.
2. **The SD card log is the source of truth.** The radio is best effort — most
   packets are lost in flight. The JSONL file is what you actually analyze
   afterward.

---

## Files at a glance

| File | Role |
|---|---|
| `flight.py` | The flight program. Sensors, logging, camera, transmit, main loop. |
| `tm_schema.py` | The binary wire format. `pack()` and `unpack()`. **Must match ground exactly.** |
| `protocol_tm.py` | Framing: sync bytes, IDs, CRC16. **Must match ground exactly.** |
| `sx126x.py` | Driver for the Waveshare SX126x UART LoRa module. |
| `test_flight_smoke.py` | Preflight check. Run at a desk before launch day. |
| `test_camera_worker_real.py` | Camera check on real hardware, writes real .jpg files. |
| `MOSS-docs.html` | Setup and teaching documentation for the whole kit. |

`tm_schema.py` and `protocol_tm.py` are shared with the ground station. They are
byte-identical copies on both Pis. If you edit one, you must copy it to the other
before the next flight, or the ground will decode garbage.

---

## `flight.py`

The flight program. Run it directly:

```
python3 flight.py
```

On start it creates `runs/YYYYMMDD_HHMMSS/` with `logs/` and `images/`
subfolders, so every flight's data stays separate.

### The main loop, in order

1. Stamp the record with UTC time and a sequence number.
2. Read the **BME280** (temperature, pressure, humidity) and derive altitude
   from pressure.
3. Read the **LTR390** (UV, ambient), **TSL2591** (lux, IR, visible), and
   **ICM20948** IMU (accel, gyro).
4. Read the **UPS HAT** over I2C for battery voltage, current, and percent, then
   compute battery alerts.
5. Read Pi health (CPU temperature, free disk). Logged locally only, never
   transmitted.
6. Ask the camera worker for a photo if enough time has passed, and read back
   whatever the worker last finished.
7. Pack, frame, and transmit the record over LoRa.
8. Append the full record to `telemetry.jsonl`.
9. Every 60 seconds, delete oldest files if the logs or images folders exceed
   their size caps.
10. Print a one-line status heartbeat and sleep until the next tick.

### The `(value, error)` pattern

Almost every helper returns a pair — a result and an error string, one of which
is always `None`. That is how rule #1 is enforced: the caller records the error
in `record["errors"]` and moves on instead of raising. You will see this in
`read_bme280`, `read_ups_status`, `read_pi_health`, `init_lora`, and others.

### `CameraWorker`

A photo takes about a second, sometimes longer. If the main loop waited for it,
telemetry would stall on every capture. So captures run on a background thread:

```python
worker = CameraWorker(images_dir)
worker.start()
worker.request_capture(seq)       # returns immediately
status = worker.snapshot_status() # safe to call every loop
worker.stop()
```

Requests go into a queue with `maxsize=4`. If the camera falls behind, new
requests are dropped rather than piling up. The capture itself runs
`rpicam-still` as a subprocess with `timeout=CAMERA_TIMEOUT_S` — that timeout
exists because a hung capture is exactly what lost all of ATLAS-1 Launch 1's
photos.

### Constants worth knowing

| Constant | Value | Notes |
|---|---|---|
| `TELEMETRY_HZ` | 0.5 | One packet every 2 seconds. |
| `IMAGE_PERIOD_S` | 25 | Capture cadence; drops to 60 s when the battery is critical. |
| `IMG_W`, `IMG_H`, `IMG_Q` | 3280 × 2464, q85 | Full Camera v2 sensor. Images stay on the SD card — they are never radioed. |
| `TX_COPIES` | 2 | Each frame is sent twice back to back. |
| `LORA_FREQ_MHZ` | 915 | Offset is `freq - LORA_BASE_FREQ_MHZ` (850), validated before the radio is touched. |
| `MAX_LOG_MB` / `MAX_IMAGE_MB` | 200 / 500 | Oldest files deleted past these caps. |

---

## `tm_schema.py` — the wire format

Converts a JSON record into a fixed-size binary blob and back. `SCHEMA_VERSION`
is currently **2** (50 bytes). Version 1 was 22 bytes and is still decodable,
because `unpack()` reads the leading version byte and dispatches on it — old
captures do not go stale when you bump the schema.

v2 is a strict superset of v1: the first ten fields are byte-for-byte identical,
with new light and IMU fields appended. You can read a v2 packet as "a v1 packet
with a tail."

**Encoding.** Everything is big-endian and integer-scaled. Temperature is
`°C × 100` in a signed int16, pressure is `hPa × 10`, lux is `lux × 10`, and so
on. That is why the round-trip test compares rounded values — a tiny loss of
precision is expected.

**Sentinels.** A missing reading still has to occupy its bytes, so each field has
a "no data" value chosen to be obviously impossible: temperature unpacks to
−327.68 °C, humidity to 255 %, battery to 0.00 V. If you see those on the ground,
the sensor failed, it is not a real reading.

**Flags.** One byte carries five booleans: image captured, battery low percent,
battery critical percent, battery low voltage, and camera fault. Bits 5–7 are
reserved.

### Changing the schema

1. Add your fields to the format string and the field table comment.
2. Bump `SCHEMA_VERSION` and add a matching `_unpack_vN`.
3. Copy the file to the ground Pi.
4. Run `test_flight_smoke.py` on both sides.

Skipping step 3 is the classic way to lose a flight's telemetry.

---

## `protocol_tm.py` — the framing layer

Wraps a payload so the ground can find it in a noisy byte stream and confirm it
arrived intact. Ten bytes of header:

```
[MAGIC 'TM'][VER][msg_id u16][frag_idx u8][frag_tot u8][paylen u8][crc16 u16][payload]
```

- **MAGIC** is the sync marker. The ground scans for it to lock onto a packet.
- **CRC16-CCITT** covers header plus payload. Mismatch means the packet was
  corrupted, and it is dropped.
- **Fragmentation** fields exist but are unused: the 50-byte payload fits in one
  frame, so `flight.py` always sends `frag_idx=0, frag_tot=1`.

`try_parse_one(buf)` is the ground-side stream parser. It pulls at most one valid
frame off the front of a buffer and hands back the leftovers. `(None, buf)` means
"not enough bytes yet, call me again." When MAGIC matches by chance in noise, or
the CRC fails, it slides forward one byte and keeps hunting.

---

## `sx126x.py` — the radio driver

A patched, demo-safe driver for the Waveshare SX126x UART LoRa module. Two things
make it different from the stock driver:

**It never reconfigures the radio.** `self.set()` is skipped at init. The build
assumes both radios were already configured identically by hand and the M0/M1
jumpers are strapped, so it just forces Normal Mode (M0=0, M1=0), opens the
serial port, and moves bytes.

**`recv_packet()` reads bursts, not slices.** A naive UART read grabs whatever
happens to be in the buffer right now, which is frequently half a packet.
Instead this waits for the first byte, then keeps accumulating until the line has
been quiet for 30 ms, treating that gap as the packet boundary. That is what
keeps whole frames from arriving in pieces.

`parse_packet()` strips the module's own 3-byte header (source address, frequency
offset) and the trailing RSSI byte when RSSI reporting is on, returning the TM
frame that `protocol_tm.try_parse_one()` then decodes.

Transmit side: `flight.py` calls `build_addressed_frame()` to prepend
`[DEST_H][DEST_L][FREQ_OFF]` before handing bytes to `send()`.

---

## Testing

### `test_flight_smoke.py` — run this before launch day

```
python3 test_flight_smoke.py
```

Exits 0 if everything passed, 1 if anything failed. Five tests, cheapest first:

1. **imports** — catches a missing library on this Pi. If this fails, everything
   after it is meaningless, so the runner stops here.
2. **pack_size** — confirms `pack()` returns the length the schema declares. It
   asks `tm_schema` for the expected value rather than hard-coding it, so the
   test survives a schema bump.
3. **round_trip** — the most valuable one. Simulates the entire radio path in
   memory with no radio: record → pack → frame → parse → unpack. Uses deliberately
   extreme values (−45 °C, 250 hPa, 15 800 m) because a schema that only works at
   room temperature is a schema that fails at 30 km.
4. **jsonl_log** — writes and re-reads records to prove the log format holds.
5. **main_loop** — runs `main()` on a background thread for six seconds and asks
   one question: did it crash? Printed errors are **expected** if no hardware is
   attached. A crash is not.

It does **not** check whether readings are sensible, whether frames physically
transmit, or whether JPEGs are valid.

### `test_camera_worker_real.py` — run this on the flight Pi

```
python3 test_camera_worker_real.py
```

Uses the real camera and writes real .jpg files to `/tmp/cam_real_test/`, so it
never touches flight data. It requests three captures five seconds apart while
ticking a 0.5 s loop, which demonstrates the point of the threading: the loop
keeps running the whole time.

Before running it, confirm the camera works at all:

```
rpicam-still -n -t 300 -o /tmp/manual_test.jpg
```

If that fails, fix the camera first. This test cannot help you.

---

## Output layout

```
runs/
  20260520_183000/
    logs/
      telemetry.jsonl     one JSON object per line, one line per 2 s
    images/
      img_000042.jpg      named by the sequence number that requested it
```

Each JSONL line contains the full record: `environment`, `uv_light`, `light`,
`imu`, `power`, `alerts`, `pi_health`, `image`, `radio`, and — when something
went wrong — `errors`. The `radio` block records transmit success, copies sent,
payload bytes, and frame bytes, so link performance is visible per packet.

---

## Preflight checklist

- [ ] `i2cdetect -y 1` shows the BME280, LTR390, TSL2591, IMU, and UPS
- [ ] `rpicam-still` produces a valid photo
- [ ] `tm_schema.py` and `protocol_tm.py` are identical on flight and ground Pis
- [ ] M0/M1 jumpers strapped LOW/LOW on both radios; both set to 915 MHz, 2400 bps
- [ ] `python3 test_flight_smoke.py` exits 0
- [ ] `python3 test_camera_worker_real.py` reports PASS
- [ ] SD card has room for the run
- [ ] Pi clock is correct (a bad clock produced the 5 595-second offset on Launch 1)
