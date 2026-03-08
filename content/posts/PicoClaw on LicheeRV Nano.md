---
title: PicoClaw on LicheeRV Nano
date: 2026-03-03
draft: false
categories: [AI, hardware]
---

> PicoClaw is an ultra-lightweight AI personal assistant written in Go. Running on a $10 RISC-V board like the [[LicheeRV Nano]], it boots in under 1 second and uses less than 10MB of RAM.

[PicoClaw Repo](https://github.com/sipeed/picoclaw) · [LicheeRV Nano Build](https://github.com/sipeed/LicheeRV-Nano-Build)

---

## What is this setup?

I wanted to run a local AI assistant on the smallest and cheapest hardware possible. The **LicheeRV Nano** caught my eye — a RISC-V 64-bit SBC powered by the SG2002 chip that costs around $10. PicoClaw is built exactly for this: a single Go binary, no containers, no cloud dependency, under 10MB of RAM.

**Why it interested me:**
- Runs on sub-$15 hardware
- RISC-V architecture — feels like betting on the future
- Single self-contained binary
- Built-in WiFi, onboard mic, and PA amp for a speaker

---

## Hardware used

- [[LicheeRV Nano]] (SG2002, RISC-V 64-bit)
- 32GB SD card
- USB-C data cable
- 8Ω 0.2W small speaker
- Mac Mini M4 32GB (runs Ollama for inference)

---

## Step 1 — SD Card Recovery

I had an old SD card that previously had [[DietPi]] installed. When I plugged it in, macOS only showed 134MB — the Linux root partition was invisible since it was ext4. One `diskutil` command recovered everything:

```bash
diskutil eraseDisk ExFAT picoClaw /dev/disk4
```

Full 32GB back, clean ExFAT — then immediately overwritten with the OS image.

---

## Step 2 — Flashing the OS

Used the `20251230` release from LicheeRV-Nano-Build — a raw `.img.xz` pipeable directly into `dd`. `xz` isn't on macOS by default:

```bash
brew install xz
xz -dc ~/Downloads/licheervnano.img.xz | sudo dd of=/dev/rdisk4 bs=4m status=progress
```

> Use `/dev/rdisk4` not `/dev/disk4` — the raw device is significantly faster on macOS.

The image writes two partitions:
- `boot` — 16.8MB FAT32 (bootloader + kernel)
- `Linux` — 1.7GB root (auto-expands on first boot)
- ~29GB free

---

## Step 3 — Connecting to the Board

### The long detour (UART)

My first instinct was UART via pins A16 (TX) and A17 (RX). I didn't have a USB-to-TTL adapter so I tried using an Arduino Nano as a passthrough by shorting RESET to GND. Got output from the board but couldn't send input — only one direction worked. Switched to an Arduino Leonardo with a serial bridge sketch, still hit character corruption. Time lost: significant.

**Bottom line:** Don't use an Arduino as a serial adapter. Get a dedicated [[USB-to-TTL adapter]] (CP2102 or CH340, ~$3).

### What actually worked

While fighting the UART setup I had the board plugged in via USB-C the whole time. I ran `ifconfig` on my Mac and spotted a new interface `en9` with a `10.x.x.x` IP — the LicheeRV Nano exposes itself as a **USB ethernet gadget** automatically. Found the board's IP and SSH'd straight in:

```bash
ssh root@10.154.190.1
# password: root
```

No drivers, no adapters — it was right there the whole time.

---

## Step 4 — WiFi Setup

The board uses an AIC8800 WiFi chip. The `/opt/wifi.sh` referenced in the official docs doesn't exist in this image. Community workaround using `wpa_supplicant` directly:

```bash
killall wpa_supplicant
wpa_passphrase YourSSID YourPassword > /etc/wpa_supplicant.conf
awk '!/#psk/' /etc/wpa_supplicant.conf > temp && mv temp /etc/wpa_supplicant.conf
wpa_supplicant -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf -B
udhcpc -i wlan0
```

The `nl80211: kernel reports: Registration to specific type not supported` flood is non-fatal — the board connects anyway.

### Making WiFi persistent

The `S30wifi` init script already handles this cleanly. Just drop two files in `/boot` (the FAT32 partition, survives reboots):

```bash
touch /boot/wifi.sta
cp /etc/wpa_supplicant.conf /boot/wpa_supplicant.conf
```

To add multiple networks:

```bash
wpa_passphrase SecondSSID password >> /boot/wpa_supplicant.conf
awk '!/#psk/' /boot/wpa_supplicant.conf > /tmp/wpa.tmp && mv /tmp/wpa.tmp /boot/wpa_supplicant.conf
```

---

## Step 5 — Installing PicoClaw

Downloaded the riscv64 binary on the Mac (board had no internet yet) and copied it over:

```bash
# On Mac
curl -L -o ~/Downloads/picoclaw.tar.gz \
  "https://github.com/sipeed/picoclaw/releases/download/v0.2.0/picoclaw_Linux_riscv64.tar.gz"
scp ~/Downloads/picoclaw.tar.gz root@10.154.190.1:/root/
```

On the board, `tar -xzf` fails — the `-z` flag isn't supported in this busybox build:

```bash
gunzip picoclaw.tar.gz
tar -xf picoclaw.tar
```

---

## Step 6 — Onboarding

```bash
./picoclaw onboard
```

Confirmed picoclaw was ready and pointed to `/root/.picoclaw/config.json` for API key setup.

![[picoclaw-working.png]]

---

## Step 7 — Running a Local Model with Ollama

Instead of a cloud API, I set up Ollama on my **Mac Mini M4 (32GB RAM)** and pointed picoclaw to it. The Mac handles inference, the board just sends and receives text.

**On the Mac Mini:**

```bash
brew install ollama
OLLAMA_HOST=0.0.0.0 ollama serve

# In a new terminal
ollama pull qwen3:8b
```

> The model is `qwen3:8b` (~5.2GB). There is no `qwen3.5` variant on Ollama.

**Find the Mac Mini's LAN IP:**

```bash
ipconfig getifaddr en0
```

**Edit `~/.picoclaw/config.json` on the board:**

Add to `model_list`:
```json
{
  "model_name": "qwen3",
  "model": "qwen3:8b",
  "api_key": "ollama",
  "api_base": "http://<mac-mini-ip>:11434/v1"
}
```

Change default in `agents.defaults`:
```json
"model_name": "qwen3"
```

**Test:**
```bash
./picoclaw agent -m "hello, are you there?"
```

It responded. Full stack working — RISC-V board → local LLM on Mac Mini, no cloud.

---

## Step 8 — Audio (Onboard Mic + Speaker)

The board has an onboard analog silicon mic and a PA amplifier (up to 1W). Both confirmed working.

**Test mic (record 5 seconds):**
```bash
arecord -D hw:0,0 -f S16_LE -r 16000 -c 1 -d 5 /tmp/test.wav
```

**Test speaker:**
```bash
aplay -D hw:1,0 /tmp/test.wav
```

**Transfer recording to Mac to listen:**
```bash
# On Mac
scp root@10.154.190.1:/tmp/test.wav ~/Desktop/test.wav
```

Tested with an 8Ω 0.2W speaker — works well. Amp can drive up to 1W, so there's room to go louder.

**Whisper-cpp installed on Mac Mini** for speech transcription (Metal accelerated on M4):
```bash
brew install whisper-cpp
```

> Full speech pipeline (mic → whisper → picoclaw → TTS → speaker) pending.

---

## My Takeaways

1. **The USB-C ethernet gadget is the hidden gem** — plug in, run `ifconfig`, SSH in. No setup needed.
2. **The official docs are outdated** — scripts referenced don't exist. GitHub issues from other users are more reliable than the wiki.
3. **Arduino-as-serial-adapter is not worth it** — wastes hours. Get a real [[USB-to-TTL adapter]].
4. **WiFi persistence is already handled** — `S30wifi` just needs flag files in `/boot`.
5. **Can't run LLMs locally** — 256MB RAM rules out even the smallest models. This board is always a client.
6. **Ollama on a local Mac beats cloud APIs** — free, private, fast enough on M4.
7. **Audio works out of the box** — onboard mic and PA amp, no extra hardware needed to get started.

---

## To Do

- [x] Make WiFi persistent on boot
- [x] Confirm onboard mic works (`arecord -D hw:0,0 -f S16_LE -r 16000 -c 1`)
- [x] Confirm onboard PA amp works with 8Ω 0.2W speaker (`aplay -D hw:1,0`)
- [x] Install whisper-cpp on Mac Mini (Metal accelerated, M4)
- [ ] Download whisper model and test transcription with board recording
- [ ] Build full speech pipeline: mic → wav → Mac → whisper → picoclaw → TTS → speaker
- [ ] Explore `picoclaw-launcher-tui` for a local UI
- [ ] Research display options similar to Whisplay HAT compatible with the board
- [ ] Find best screen for use with [[LOPAKA]] pixel art UI tool
- [ ] Build character interface and animations using [LOPAKA](https://github.com/sbrin/lopaka)
- [ ] Animation ideas: idle breathing, thinking (eyes shifting), happy/confused/sleepy expressions, reaction to time of day
- [ ] Add a small screen and build a tamagotchi-like companion/agent UI
- [ ] Add a battery, USB-C charger, and power on/off switch to make it portable
- [ ] Design and 3D print a case
