# How I Unbricked My HG680P After a Failed armbian-install (DDR_ENC.USB Error)

I spent weeks trying to fix my bricked HG680P. Most guides online didn't work for my case, so I'm writing this down in case someone else hits the same wall.

---

## What Happened

I was moving my home server storage from a 30GB drive to a 256GB one. The data migration went fine, but when I tried booting from the new HDD, Armbian wouldn't load. Plugging the old HDD back in worked, so I figured the issue was that the new drive had no bootloader.

Natural next step: run `armbian-install` and write the bootloader to eMMC. Done. Rebooted.

Black screen.

No Android, no Armbian, nothing. Just a black screen every time I powered it on.

---

## The USB Burning Tool Rabbit Hole

My first instinct was to reflash the firmware using Amlogic USB Burning Tool. I downloaded what looked like the correct stock Android firmware for the HG680P, loaded it up, put the device in burn mode (hold reset, plug power), and hit Start.

It got to 1% and died with this:

```
ulValue = 0xbdfd21bc
File change to DDR_ENC.USB
[Err]--DDR_ENC.USB
[Err]--Read item data error, code -1
[0x10103005]Romcode/Initialize DDR/Download buffer/Read item data failed
```

I had no idea what `DDR_ENC.USB` meant. After some digging, I found out that certain HG680P units use an encrypted DDR initialization file — and the value `0xbdfd21bc` is basically a fingerprint of the specific DDR chip on my board. Most firmware images floating around the internet only include the unencrypted `DDR.USB`, which doesn't work for my unit.

I tried every firmware I could find:
- Original HG680P stock firmware → same error
- atvXperience Android 7 for S905X → same error  
- Aidan's ROM v7.5 for S905X → same error
- Random firmwares from Indonesian STB communities → same error

All of them hit the same wall at exactly 1%.

---

## Other Things I Tried (That Didn't Work)

**SD card recovery** — I tried making a recovery SD card with BootcardMaker and uboot.bin. The problem was my SD card (a cheap Vgen 4GB) wasn't even recognized by my laptop, let alone the STB. Ordered a Sandisk but even with that, the STB kept ignoring the SD card and trying to boot from the broken eMMC first.

**ADB** — Since Android wasn't booting, there was nothing to connect to.

**PC desktop** — Someone suggested the "hub" issue with laptops was causing the USB Burning Tool to fail. Tried it on a desktop with direct motherboard USB. Same `DDR_ENC.USB` error. It wasn't a power issue — the firmware just didn't have the right file.

**Recovery mode tricks** — Various button combinations, timing tricks. Nothing.

At this point I was pretty convinced the device was dead.

---

## What Actually Worked — USB TTL and U-Boot

A few people mentioned using a USB TTL adapter to access the U-Boot console via UART. I'd never done this before and it sounded intimidating, but I ordered a CP2102 module (cost about $2) and gave it a shot.

The idea is simple: even though the eMMC bootloader is broken, the chip itself is fine. U-Boot is still running — you just can't see it because there's no display output. The UART pins on the PCB give you direct serial access to U-Boot before it tries (and fails) to boot from eMMC.

**Hardware needed:**
- USB TTL adapter (CP2102 or CH340)
- 3x female-to-female jumper wires
- Soldering iron — the HG680P PCB has UART holes but no pin headers installed, so you need to solder them yourself
- A good SD card (Sandisk Class 10 minimum)

**Finding the UART pins:**

Open the case and look for a row of 3-4 holes near the main chip, usually labeled TX, RX, GND on the PCB silkscreen. Solder pin headers into them.

**Wiring:**
```
USB TTL → HG680P
TXD     → RXD
RXD     → TXD  
GND     → GND
(don't connect VCC)
```

TX and RX have to be crossed — this tripped me up the first time.

**Putty settings:**
```
Connection type: Serial
Speed: 115200
Data bits: 8, Stop bits: 1, Parity: None, Flow control: None
```

Power on the STB with Putty open. If everything is wired correctly, you'll see boot output scrolling, and eventually:

```
gxl_p212_v1#
```

That's the U-Boot prompt. I actually laughed out loud when I saw it for the first time.

---

## From Here, Two Options

### Option 1 — Write the Correct U-Boot Directly to eMMC

This is the cleaner method. Flash an Armbian image to your SD card using Balena Etcher (use the S905X image from [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)). That image includes `u-boot-p212.bin`, which is the correct bootloader for HG680P.

With the SD card inserted and U-Boot console open:

```bash
# See what's on the SD card
fatls mmc 1

# Load u-boot into memory
fatload mmc 1 0x1000000 u-boot-p212.bin

# You should see something like:
# 606670 bytes read in 19 ms (30.5 MiB/s)

# Switch to eMMC
mmc dev 0

# Write to eMMC bootloader area
mmc write 0x1000000 0 0x1000

# Should say: 4096 blocks written: OK

# Reboot
reset
```

After this, the STB booted from SD card normally. The broken u-boot in eMMC was replaced with the correct one.

### Option 2 — U-Boot + USB Burning Tool Timing Trick

This is what I stumbled onto before I figured out Option 1. It's less reliable but it worked.

With USB Burning Tool open and firmware loaded on PC, USB A-to-A cable connected:

1. Click Start in USB Burning Tool
2. In U-Boot console, type `update`
3. The trick is timing — you need the chip to enter USB burn mode at the exact moment the tool is scanning

Most attempts still failed at 1%. But after maybe 15-20 tries with different timing, one attempt jumped to 7% and kept going. It eventually flashed successfully.

Honestly I'm not 100% sure why this worked when nothing else did. My best guess is that entering `update` via U-Boot bypasses whatever check normally triggers the DDR_ENC.USB lookup, at least in some timing window. If anyone has a better explanation I'd love to know.

---

## Installing Armbian Properly

Once the STB was booting again (via Android TV after the successful flash), I did the Armbian install properly this time:

1. Flashed Armbian to SD card with Balena Etcher
2. In Android TV terminal emulator: `reboot update`
3. STB booted into Armbian from SD card
4. Ran `armbian-install`
5. **Critical step:** selected ID 105 (HG680P) when asked for device model

```
Model Name    : HG680P
FDTFILE       : meson-gxl-s905x-p212.dtb
UBOOT_OVERLOAD: u-boot-p212.bin
```

6. Chose ext4 filesystem
7. Waited ~15 minutes
8. Shutdown, pulled SD card, powered on — booted from eMMC perfectly

---

## Things I Wish I'd Known Earlier

The `DDR_ENC.USB` error is not a death sentence. It just means the firmware you're using doesn't have the right DDR initialization file for your specific chip. The chip is fine.

If you can get to U-Boot via UART, you're not actually bricked — you just need the right approach.

The `u-boot-p212.bin` from ophub's Armbian image is the correct bootloader for HG680P. Don't use generic u-boot files.

Cheap SD cards will waste your time. I spent days troubleshooting what turned out to be a bad SD card.

---

## Specs for Reference

```
Device  : Fiberhome HG680P
SoC     : Amlogic S905X
Board   : gxl_p212
RAM     : 2GB
Storage : 8GB eMMC
DDR enc : ulValue = 0xbdfd21bc
OS      : Armbian 25.05 (Ubuntu Noble, kernel 6.1.137-ophub)
```

---

## Links That Helped

- https://github.com/ophub/amlogic-s9xxx-armbian
- https://github.com/ophub/amlogic-s9xxx-armbian/discussions/2704
- https://xdaforums.com/t/bricked-s905x-box.4043053/
- https://xdaforums.com/t/s905x-unbrick-xiaomi-mi-box-3-mdz-16-aa.3502992/page-9

---

If you're hitting the same `DDR_ENC.USB` error with `ulValue = 0xbdfd21bc`, hopefully this saves you a few weeks of frustration. Feel free to open an issue or leave a comment if something here doesn't work for your setup.
