# Zephyr in Production - Workshop

A hands-on Packt workshop: taking Zephyr RTOS firmware from a prototype blinky to a
production-ready, signed, CI-built image.
Target board: **NXP FRDM-MCXA156**.

This repo holds the starting firmware in `app/`. We'll build on it together during the
workshop.

## Before you join

Please complete the steps below **before the workshop starts**.
Pulling the Zephyr tree and SDK takes time.

If something doesn't work, check [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

The goal is to be able to **build and flash blinky to your board and see the LED toggle**.

### What to bring / have ready

- **FRDM-MCXA156 board** + a **USB-C cable**
- A **Linux or macOS host**, or **Windows with WSL2**
- **Python 3.10+** and **Git** installed
- A **GitHub account** (used later in the CI portion)

### 1. Get the code and a Python virtualenv

```bash
git clone <workshop-repo-url> packt-training
cd packt-training
python3 -m venv .venv
source .venv/bin/activate      # run this in every new terminal
pip install west
```

### 2. Initialize west and pull dependencies

```bash
west init -l .
west update
west zephyr-export
west packages pip --install
```

`west.yml` pins **Zephyr v4.4.0** and only pulls `cmsis_6` and `hal_nxp`
into `deps/` - a small checkout, not the full Zephyr tree.

### 3. Install the Zephyr SDK (one-time, large download)

```bash
west sdk install
```

### 4. Install the flash and signing tools

```bash
pip install -r requirements.txt
pyocd pack install MCXA156
pyocd list                      # should list the MCU-Link probe when the board is plugged in
imgtool -h                      # should print imgtool help message
```

**Linux users:** you almost certainly need the CMSIS-DAP udev rules and `dialout` group
membership for the probe and serial port to work without `sudo`.
See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#linux-usb-permissions).

### 5. Install Go and the `mcumgr` CLI

[`mcumgr`](https://github.com/apache/mynewt-mcumgr-cli) is the SMP host tool used to push
a new signed image to the board over serial. It is written in Go, so you install Go first,
then `go install` the client.

Install Go (1.21+) following the [official instructions](https://go.dev/doc/install).

Then install `mcumgr` and put Go's bin dir on your PATH:

```bash
go install github.com/apache/mynewt-mcumgr-cli/mcumgr@latest

# the binary lands in $(go env GOPATH)/bin - add it to PATH if it isn't already
export PATH="$PATH:$(go env GOPATH)/bin"   # add to ~/.bashrc / ~/.zshrc to persist
mcumgr version                             # confirm it runs
```

### 6. Verify your setup

Run these and confirm each works:

```bash
west --version
west list                                   # shows zephyr, cmsis_6, hal_nxp under deps/
west build -b frdm_mcxa156 app --pristine   # builds to completion
```

Then plug in the board and flash blinky:

```bash
west flash --runner pyocd
```

**Success:** the onboard LED toggling at 1 Hz, and serial output at 115200 baud:

```
[00:00:01.000,000] <inf> main: LED state: ON
[00:00:02.000,000] <inf> main: LED state: OFF
```

To watch the serial log, in a second terminal:

```bash
minicom -D /dev/ttyACM0 -b 115200
# or: picocom -b 115200 /dev/ttyACM0
# or: tio /dev/ttyACM0
```

---

## What we'll cover

Starting from the blinky in `app/`, we layer on production concerns:

- **Build** - board anatomy, custom board overlays, multi-target builds
- **Secure** - MCUboot bootloader, partition layout, sysbuild
- **Signing** - key generation, build-time signing, and what a wrong key does
- **CI** - a GitHub Actions build skeleton
- **Harden** - a release config and image-size analysis

You don't need to prepare anything for these beyond finishing the setup above.
