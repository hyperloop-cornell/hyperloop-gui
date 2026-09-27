# Raspberry Pi Hub Setup (rpi-hub-server)

How to set up a Raspberry Pi as a telemetry hub from a blank SD card, verify it, and (for the one
hub with a cellular HAT) add Wi-Fi-to-cellular failover.

Written for humans and agents. Values in `<ANGLE_BRACKETS>` are placeholders. Every step ends
with an **Expected result**; stop and troubleshoot if it does not match. Commands run on the Pi as
user `pi` unless they start with `sudo`.

## Rules (read first)

- A hub's `DEVICE_TOKEN` is a secret: do not print it, paste it into chat, or commit it. It lives
  only in `/opt/rpi-hub-server/.env` (mode 600).
- Each hub needs a `HUB_ID` + `DEVICE_TOKEN` pair that is also listed in the cloud's
  `DEVICE_TOKENS` (see `.claude/auth-setup.md`, step 5). Get the pair before starting.
- All hubs run the same code. The only per-hub differences are `.env` and the profile:

| Hub | `HUB_PROFILE` | Extra software |
|---|---|---|
| Lab hubs (Wi-Fi only) | `lab-hub` | none |
| The hub with the cellular HAT | `cellular-hub` | uplink manager (section "Cellular hub") |

## 1. Prepare the SD card

Use Raspberry Pi Imager with **Raspberry Pi OS Lite (64-bit)**. In the Imager settings:

- Hostname: the hub ID, e.g. `rpi-lab-01`
- Username `pi` with a password of your choice
- Wi-Fi: the lab network (SSID, password, country)
- Enable SSH

Boot the Pi and connect: `ssh pi@<HOSTNAME>.local`

**Expected result:** a shell on the Pi. `ping -c 3 api.cornellhyperloop.com` gets replies.

## 2. Base packages and permissions

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git python3-venv python3-pip openocd usbutils
sudo usermod -aG dialout,plugdev pi
sudo reboot
```

After the reboot, reconnect and check:

```bash
groups
ls /lib/udev/rules.d/ | grep -i openocd
python3 --version
```

**Expected result:** `groups` includes `dialout` and `plugdev`; the OpenOCD udev rule file is
listed (it gives `plugdev` access to ST-LINK probes); Python is 3.11 or newer.

## 3. arduino-cli and board cores

Install arduino-cli system-wide so the systemd service finds it:

```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sudo BINDIR=/usr/local/bin sh
arduino-cli version
```

Install the cores as user `pi` (the service runs as `pi` and uses `~/.arduino15`):

```bash
arduino-cli config init
arduino-cli config add board_manager.additional_urls https://github.com/stm32duino/BoardManagerFiles/raw/main/package_stmicroelectronics_index.json
arduino-cli core update-index
arduino-cli core install arduino:avr
arduino-cli core install arduino:renesas_uno
arduino-cli core install STMicroelectronics:stm32
```

The Uno R4 core ships udev rules for its bootloader; install them once:

```bash
sudo ~/.arduino15/packages/arduino/hardware/renesas_uno/*/post_install.sh
```

**Expected result:** `arduino-cli core list` shows `arduino:avr`, `arduino:renesas_uno` and
`STMicroelectronics:stm32`. The STM32 core is large; the install can take 10+ minutes. If the
STM32 core fails to install on the Pi, STM32 boards can still be flashed with precompiled
`.bin`/`.elf` files from the GUI (only `.ino` compilation needs the core).

## 4. Install rpi-hub-server

```bash
sudo mkdir -p /opt/rpi-hub-server
sudo chown pi:pi /opt/rpi-hub-server
git clone https://github.com/hyperloop-cornell/rpi-hub-server.git /opt/rpi-hub-server
cd /opt/rpi-hub-server
python3 -m venv venv
venv/bin/pip install -r requirements.txt
```

**Expected result:** `venv/bin/python -c "import fastapi, serial, websockets; print('ok')"`
prints `ok`.

## 5. Configure the hub

```bash
cd /opt/rpi-hub-server
cp .env.example .env
chmod 600 .env
nano .env
```

Set these keys (leave the others at their defaults):

```bash
HUB_PROFILE=<lab-hub or cellular-hub>
HUB_ID=<HUB_ID>
DEVICE_TOKEN=<DEVICE_TOKEN>
SERVER_ENDPOINT=wss://api.cornellhyperloop.com/hub
```

**Expected result:** `grep -c DEVICE_TOKEN= .env` prints `1`, and `HUB_ID` matches the entry in
the cloud's `DEVICE_TOKENS` exactly.

## 6. Check the boards are recognized

Plug in each board the hub will serve, then:

```bash
lsusb
arduino-cli board list
```

Compare the `ID vvvv:pppp` values from `lsusb` with `config/boards.yaml`:

| Board | Expected `lsusb` ID |
|---|---|
| Arduino Uno R3 | `2341:0043` (or `2341:0001`, `2a03:0043`) |
| Arduino Mega 2560 | `2341:0042` |
| Nano clone (CH340) | `1a86:7523` |
| Arduino Uno R4 Minima | `2341:0069` |
| Arduino Uno R4 WiFi | `2341:1002` |
| STM32F407G-DISC1 (ST-LINK) | `0483:374b` or `0483:3752` (older firmware: `0483:3748`) |

**Expected result:** every board's ID appears in `config/boards.yaml`. If one does not, add its
ID to the matching board entry (or a new entry) in `config/boards.yaml`. Unknown devices are
deliberately **not** opened, which is what keeps a cellular modem's serial ports untouched.

STM32F407G-DISC1 serial output: the hub reads the ST-LINK virtual COM port. Confirm the board
revision's ST-LINK exposes one (`ls /dev/ttyACM*` shows a new device when the board is plugged in)
and that the sketch prints on the UART wired to it; otherwise serial data will not appear even
though flashing works.

## 7. First run in the foreground

```bash
cd /opt/rpi-hub-server
venv/bin/python -m src.main
```

Watch the log for about 20 seconds, then stop it with Ctrl+C.

**Expected result:** lines containing `Connected to server` and `Server acknowledged hub
connection`, and one `connection_opened` per plugged-in board. If you see `ws_rejected`, the
`HUB_ID`/`DEVICE_TOKEN` pair does not match the cloud. If you see `Device detected but not opened`,
that device's ID is missing from `config/boards.yaml` (step 6).

## 8. Run as a service

```bash
sudo cp /opt/rpi-hub-server/deploy/systemd/rpi-hub-server.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now rpi-hub-server
sleep 5
systemctl is-active rpi-hub-server
journalctl -u rpi-hub-server -n 30 --no-pager
```

**Expected result:** `active`, and the same connection lines as in step 7.

## 9. Verify end to end

On the Pi (the local API listens on loopback only):

```bash
curl -s 127.0.0.1:8000/health
curl -s 127.0.0.1:8000/ports
curl -s 127.0.0.1:8000/connections
```

**Expected result:** `"uplink_connected":true`; every board is listed under `/ports` with a
`board_profile`, and has a session under `/connections`.

In the GUI (https://gui.cornellhyperloop.com):

1. Dashboard: the hub shows **Connected** with its profile name.
2. Device Manager: its ports are listed; subscribe to one and open Live Telemetry; data streams.
3. Firmware Flash (signed in with your NetID, not View Only), one test per board type present:

| Board | Test | Expected |
|---|---|---|
| Uno R3 / Mega / Nano | "Blink LED" preset, board "Detected" | Flash completed; LED blinks; serial resumes |
| Uno R4 Minima / WiFi | "Blink LED" preset | Flash completed (the board briefly drops off USB and comes back) |
| Uno R4 | Upload a `.bin` built with `arduino-cli compile --fqbn arduino:renesas_uno:minima --output-dir out <sketch>` | Flash completed |
| STM32F407G-DISC1 | "Blink LED" preset (first compile can take several minutes) or a `.bin`/`.elf` | Flash completed; board runs the new program |

4. Restart (circular arrow) on a port: task completes and serial data resumes.

## Cellular hub (one Pi only)

Do steps 1 to 9 with `HUB_PROFILE=cellular-hub`, then add the uplink manager, which prefers
Wi-Fi, switches to the cellular HAT when Wi-Fi fails, and reports the active link to the GUI.

### C1. Modem and NetworkManager

```bash
sudo apt install -y modemmanager network-manager
sudo systemctl enable --now ModemManager
mmcli -L
nmcli device status
```

**Expected result:** `mmcli -L` lists the modem; `nmcli device status` shows `wlan0` (wifi) and
the modem's data interface (often `wwan0`, sometimes `usb0`/`ppp0`; note the name,
`<CELLULAR_INTERFACE>`).

Create the cellular connection (APN from the SIM provider):

```bash
sudo nmcli connection add type gsm ifname '*' con-name cellular apn <APN> connection.autoconnect yes
sudo nmcli connection up cellular
```

**Expected result:** `nmcli device status` shows the cellular interface `connected`, and
`ping -I <CELLULAR_INTERFACE> -c 3 1.1.1.1` gets replies.

### C2. Install the uplink manager

```bash
sudo git clone https://github.com/hyperloop-cornell/wifi-fallback-relay.git /opt/hyperloop-uplink
sudo python3 -m venv /opt/hyperloop-uplink/.venv
sudo /opt/hyperloop-uplink/.venv/bin/pip install -r /opt/hyperloop-uplink/requirements.txt
sudo mkdir -p /etc/hyperloop-uplink
sudo cp /opt/hyperloop-uplink/.env.example /etc/hyperloop-uplink/config.env
sudo nano /etc/hyperloop-uplink/config.env
```

Set:

```bash
WIFI_INTERFACE=wlan0
CELLULAR_INTERFACE=<CELLULAR_INTERFACE>
CELLULAR_CONNECTION=cellular
DRY_RUN=true
```

Start it in dry-run mode first (it logs routing commands without running them):

```bash
sudo cp /opt/hyperloop-uplink/systemd/hyperloop-uplink.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now hyperloop-uplink
sleep 15
journalctl -u hyperloop-uplink -n 20 --no-pager
cat /run/hyperloop-uplink/status.json
```

**Expected result:** the log shows `DRY_RUN: nmcli ...` lines; `status.json` has
`"active": "wifi"` and both links with `"probe_ok": true`.

Then set `DRY_RUN=false` in `/etc/hyperloop-uplink/config.env` and
`sudo systemctl restart hyperloop-uplink`.

### C3. Failover test

```bash
systemctl show rpi-hub-server -p NRestarts
sudo nmcli radio wifi off
sleep 30
cat /run/hyperloop-uplink/status.json | grep '"active"'
journalctl -u rpi-hub-server -n 20 --no-pager | grep -i connected
sudo nmcli radio wifi on
sleep 90
cat /run/hyperloop-uplink/status.json | grep '"active"'
systemctl show rpi-hub-server -p NRestarts
```

If you are connected over Wi-Fi SSH, the session drops when Wi-Fi goes off; run the block from a
local console, or wrap it in `sudo systemd-run --unit=failover-test bash -c '...'` and read the
results afterwards.

**Expected result:** `"active": "cellular"` within about 30 seconds of Wi-Fi going off, and the
hub log shows it reconnected; the GUI dashboard shows a **Cellular** badge. About 60 seconds after
Wi-Fi returns, `"active": "wifi"`. `NRestarts` is the same before and after (the uplink manager
never restarts the hub).

## Updating a hub

```bash
cd /opt/rpi-hub-server
git pull
venv/bin/pip install -r requirements.txt
sudo systemctl restart rpi-hub-server
journalctl -u rpi-hub-server -n 20 --no-pager
```

**Expected result:** `Connected to server` in the log; the hub shows Connected in the GUI.

## Migrating a hub that ran the previous software

Older installs may live in another directory (for example `/opt/rpi-hub-service`) and may run the
old relay daemons. On such a Pi:

```bash
sudo systemctl disable --now rpi-master-relay rpi-slave-fallback 2>/dev/null
sudo rm -f /etc/systemd/system/rpi-master-relay.service /etc/systemd/system/rpi-slave-fallback.service
sudo systemctl daemon-reload
systemctl list-units --type=service | grep -i -E "hub|relay|fallback"
```

Stop and disable any old hub service shown by the last command, then follow steps 4 to 9. Reuse
the old `HUB_ID` and `DEVICE_TOKEN` if the cloud still lists them.

**Expected result:** only `rpi-hub-server` (and on the cellular hub, `hyperloop-uplink`) remain.

## Troubleshooting

| Symptom | Check |
|---|---|
| `ws_rejected` in the hub log | `HUB_ID` or `DEVICE_TOKEN` does not match the cloud's `DEVICE_TOKENS` |
| Hub keeps logging `Reconnecting in ...` | Network or `SERVER_ENDPOINT`; `curl -sI https://api.cornellhyperloop.com/health` from the Pi |
| Board not in `/ports` | USB cable (charge-only cables have no data lines); `lsusb`; `dmesg | tail` |
| Board in `/ports` but no session | ID missing from `config/boards.yaml`, or permissions (`groups` must include `dialout`) |
| Flash fails with "arduino-cli not found" | Step 3 install; `which arduino-cli` must print `/usr/local/bin/arduino-cli` |
| Flash fails with a missing core or platform | `arduino-cli core list` as user `pi`; install the missing core |
| STM32 flash fails in OpenOCD (`LIBUSB_ERROR_ACCESS`) | User not in `plugdev`, or udev rule missing (step 2); unplug/replug after fixing |
| Uno R4 flash hangs or the board stays in bootloader | Run the core's `post_install.sh` (step 3); double-tap reset and flash again |
| GUI shows no Wi-Fi/Cellular badge on the cellular hub | `systemctl status hyperloop-uplink`; `/run/hyperloop-uplink/status.json` must be updating |
