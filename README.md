# MOSS Kit

A hands-on kit for building, flying, and recovering a real telemetry payload — the hardware, the code, and the docs that tie them together. Built by students, for students, as part of UNLV's ATLAS-1 high-altitude balloon platform.

## What's here

The folders in this directory are already arranged in the structure each Raspberry Pi expects. Keep them together — the scripts read and write files relative to their own location, so moving individual files out of a folder will break them.

- `hab/` — Flight Pi (payload). The main flight program, shared pipeline modules, LoRa radio driver, and preflight tests. A `runs/` folder gets created at runtime, one timestamped folder per launch.
- `ground/` — Ground Pi. The receiver, the dashboard it serves, the shared modules, and optional test scripts for checking things work with no radio attached.
- `'Docs/MOSS-docs.html` — Full setup documentation. Open it in any browser.

No web server configuration ships with the kit. The docs walk you through installing nginx and writing its config yourself, one command at a time — and the only path you type is wherever you actually put `ground/`. If your folder lives somewhere else, or you named it something else, that's fine; the config points at whatever you tell it.

## Getting the files onto your Pis

There are two ways to get the MOSS Kit files:

1. **Clone the repository (recommended):**
```bash
git clone https://github.com/unlv-atlas-1/mosskit.git
```

2. **Download as a ZIP:**
   Click the green "Code" button on the repository page, then select "Download ZIP".

Cloning lets you pull updates later. Downloading the ZIP is the simplest way to start, since it hands both Pis an identical, unmodified copy of every file.

## The most important rule

`tm_schema.py` and `protocol_tm.py` must be **byte-identical copies** on both Pis. They aren't shared over the network — each Pi keeps its own copy. If the two drift apart, the ground station can't decode anything the flight Pi sends.

Because a fresh download gives both Pis the same starting copies, pulling the files down again is the simplest way to guarantee they match. To verify:

```bash
sha256sum ~/hab/tm_schema.py ~/hab/protocol_tm.py       # flight Pi
sha256sum ~/ground/tm_schema.py ~/ground/protocol_tm.py # ground Pi
```

If the fingerprints match, the files are the same.

## Step-by-step guide

**Open `MOSS-docs.html` in a browser** for the full walkthrough. It covers everything below in detail, with commands you can copy.

The recommended order:

1. **Main setup** — do this on both Pis. OS, connection, interfaces, Python. Four steps, once per Pi. Nothing else works until this is done.
2. **Ground station** — build and test this first. Install nginx, write the config so it points at your `ground/` folder, then prove the dashboard works using fake data. No radio needed, nothing to wire.
3. **Flight Pi** — the payload. Wire the sensors, confirm the bus sees each one, install the drivers, then run it. With the ground station already working, you can watch real telemetry arrive immediately.
4. **Field hotspot** — last of all. Only once both Pis work on ordinary Wi-Fi. This is purely for the launch site, where no network exists.

Ground before flight is deliberate: the ground station can be fully tested on a desk, so when you power up the payload, anything that goes wrong is on the flight side rather than a mystery split between the two.

## Hardware

| Component | Role |
|-----------|------|
| Raspberry Pi 3B+ | Flight computer |
| Sensor suite (BME280, LTR390, TSL2591, ICM20948) | Environmental + motion data |
| Raspberry Pi Camera v2 | Imaging |

## Notes

- Paths in the docs use `unlvcube1` as the username (`/home/unlvcube1/hab/`). Yours will read `/home/<your-username>/` with whatever name you chose in Raspberry Pi Imager.
- Folder names (`hab`, `ground`) are examples. The structure is what matters — just make sure the path you type into the nginx config matches the folder you actually have.
- Wi-Fi never reaches the balloon in flight. It's only for the ground: starting scripts before launch and pulling data after recovery.

For questions, bug reports, or ideas, please see our [organization's contact information](https://github.com/unlv-atlas-1/.github/blob/main/CONTACT.md)
