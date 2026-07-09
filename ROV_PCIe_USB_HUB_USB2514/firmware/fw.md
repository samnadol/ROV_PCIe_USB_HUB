# µPD720201 Firmware Provisioning Guide

How to bring up the USB host controller (U1, µPD720201K8-701-BAC-A) from a
factory-blank SPI flash (U9) to a self-booting board, using a plain Ubuntu
PC and the PCIe x1 → FFC test adapter.

<details>
<summary><b>Background — how the chip boots, and why a blank flash is safe</b></summary>

The µPD720201 has **no internal firmware ROM**. At every power-on it either:

1. **Loads firmware from the external SPI flash** (U9) — fully autonomous,
   works on any host, no driver cooperation needed. This is the end state
   we want.
2. **Waits for a register-based firmware upload** from the host (BIOS or
   OS driver). Linux ≥ 5.9 does this automatically via `xhci-pci-renesas`
   if `renesas_usb_fw.mem` is present in `/lib/firmware`.

A blank or corrupt flash behaves like an absent flash — the chip falls
back to mode 2. You cannot brick the board with a bad flash write; worst
case you erase and try again.

<details>
<summary>How flash-type detection works (no persistent config on the chip)</summary>

The chip auto-detects the flash type at power-on (JEDEC ID → PCI config
register `0xEC`). Nothing about the flash is stored on the chip itself.
Write/erase operations need a per-vendor "ROM Parameter" written to config
register `0xF0` by the flashing tool for each session (volatile). For our
SST25VF512A: ROM Information = `00BF_0048`, ROM Parameter = `0001_0791`.

Reference: Renesas user's manual ISG-NK1-110027 (µPD720201/202), sections
5.5 (ROM connection), 6 (external ROM access), 7 (FW download interface).

</details>
</details>

## What you need

- The board, connected to a PC via the PCIe x1 → FFC adapter.
  Board logic power comes from the adapter's 5 V buck (camera VBUS supply
  not required for this procedure).
- Ubuntu (any recent release; kernel ≥ 5.9).
- Firmware image **v2.0.2.6** — included in this repo alongside this file
  as **`K2026090.mem`**.
  - SHA256: `177560c224c73d040836b17082348036430ecf59e8a32d7736ce2b80b2634f97`
  - Verify before use: `sha256sum K2026090.mem`
- The loader tool: https://github.com/markusj/upd72020x-load

```sh
sudo apt install build-essential git pciutils
git clone https://github.com/markusj/upd72020x-load
cd upd72020x-load && make
sha256sum ../K2026090.mem      # must match 177560c2...
```

<details>
<summary>If the bundled firmware file is ever lost</summary>

It circulates as `renesas_usb_fw.mem` / `K2026.mem` via the linux-firmware
tree, distro packages, and the chunkeey/renesas-fw and
denisandroid/uPD72020x-Firmware GitHub mirrors — Renesas does not host a
public download. Always check the hash.

</details>

## Step 1 — Verify enumeration

<details>
<summary><b>Expand</b></summary>

Plug the board into the PC (power off), boot, then:

```sh
lspci -nn -d 1912:0014
```

Expected: one line, `USB controller: Renesas Technology Corp. uPD720201`.

`dmesg | grep -i xhci` complaining about firmware is **normal and correct**
at this stage — it means the chip enumerated and is waiting in download
mode.

<details>
<summary>Troubleshooting: chip not listed</summary>

Check adapter seating, FFC orientation/seating, PERST wiring, and that
both bucks came up (test points: 3.3 V, 1.05 V, 3V3_PG/1V05_PG).
Intermittent PCIe errors in `dmesg` usually mean a badly seated or kinked
ribbon.

</details>
</details>

## Step 2 — Prove the silicon with a RAM upload (non-persistent)

<details>
<summary><b>Expand</b></summary>

This validates the whole board without touching the flash. Find the
bus/device/function from lspci (e.g. `03:00.0` → bus 0x03, dev 0x00,
fn 0x0), unbind the kernel driver if it claimed the device, and upload:

```sh
sudo sh -c 'echo 0000:03:00.0 > /sys/bus/pci/drivers/xhci_hcd/unbind' 2>/dev/null
sudo ./upd72020x-load -u -b 0x03 -d 0x00 -f 0x0 -i K2026090.mem
sudo sh -c 'echo 0000:03:00.0 > /sys/bus/pci/drivers/xhci_hcd/bind'
```

Then `lsusb` should show new root hubs, and a device plugged into any
board port should enumerate. This state is volatile — it lasts until
power-off.

<details>
<summary>Alternate route: zero-tool check via the kernel loader</summary>

`sudo cp K2026090.mem /lib/firmware/renesas_usb_fw.mem` and reboot;
`xhci-pci-renesas` uploads it automatically at probe.

</details>
</details>

## Step 3 — Confirm the chip sees the flash

<details>
<summary><b>Expand</b></summary>

```sh
sudo setpci -s 03:00.0 ec.l
```

Expected for the SST25VF512A: `00bf0048`. Any of the values from manual
Table 6-1 means detection works; `00000000` means the chip found no ROM —
check U9 soldering and the SPI pull-up straps (SPISO 10k is mandatory).

Optionally dump the (blank) flash — also confirms the tool knows the
part's ROM Parameter:

```sh
sudo ./upd72020x-load -r -b 0x03 -d 0x00 -f 0x0 -o blank_dump.bin
```

<details>
<summary>If the tool reports "unknown EEPROM, no parameters found"</summary>

Its internal table lacks our flash — add an entry for ROM info
`0x00BF0048` with parameter `0x00010791` in the source (or swap U9 to a
W25X20-family part, which the tool already knows).

</details>
</details>

## Step 4 — Write the flash

<details>
<summary><b>Expand</b></summary>

> The `-w` path is marked untested by the tool's author and writes the
> file verbatim (no knowledge of the chip's two-image-block flash layout).
> The chip itself CRC-checks every boot and falls back gracefully, so the
> risk is a non-booting flash, not a brick.

```sh
sudo ./upd72020x-load -w -b 0x03 -d 0x00 -f 0x0 -i K2026090.mem
sudo ./upd72020x-load -r -b 0x03 -d 0x00 -f 0x0 -o verify_dump.bin
```

<details>
<summary>What the write sequence does under the hood</summary>

Per the manual: the tool sets ROM Parameter (0xF0), writes magic
`53524F4D` ("SROM") to DATA0, sets External ROM Access Enable, streams
32-bit words through DATA0/DATA1 with handshake bits, clears Enable, then
the chip runs its own CRC16 check and posts Result Code `001b` on
success.

</details>

<details>
<summary>Alternate route: Renesas Windows updater (reference implementation)</summary>

If the Linux tool misbehaves, boot Windows once and run the vendor
updater (`k2026fwup1.exe`, SHA1
`44184f1379c061067ac23ac30055a2b04ddf3940`, or the `W201FW21.EXE` package
circulating via StarTech/station-drivers/Win-Raid). It handles the flash
layout and blank-ROM cases authoritatively. Note it refuses to run if it
detects more than one µPD72020x in the system.
`W201FW21.EXE /srom 0 /erase` performs a full ROM erase if a botched
write needs a clean slate.

</details>
</details>

## Step 5 — Prove autonomous boot

<details>
<summary><b>Expand</b></summary>

1. Full power cycle (not just reboot — the ROM load happens after
   PONRSTB/PERSTB deassert on cold start).
2. `lsusb` — root hubs present and ports functional. To confirm the
   firmware came from the flash rather than the kernel loader, check
   `dmesg | grep -i renesas` — no upload should be reported (on a host
   without `/lib/firmware/renesas_usb_fw.mem` installed, none can be).
   The chip read the flash itself: second image block first,
   CRC16-verified, first block as fallback.

The board is now host-independent: it will behave identically on the Pi 5
with no firmware file installed there.

</details>

## Maintenance notes

<details>
<summary><b>Expand</b></summary>

- **Field update:** either rerun Step 4 on the PC adapter, or run the same
  tool on the Pi — it's pure PCI config-space access and works anywhere
  the chip enumerates. The chip's "Reload" bit permits a reload without
  power-cycling, but only with the xHC halted; a power cycle is simpler.
- **Recovery:** if the flash image is ever corrupt, the chip silently
  falls back to download mode — copy `K2026090.mem` to
  `/lib/firmware/renesas_usb_fw.mem` on any host and the board works
  while you re-flash.
- **Do not** use firmware for other family members (µPD720200/200A use
  different firmware; 2.0.2.6 covers µPD720201 rev 03 silicon —
  `lspci -nn` shows the revision).

<details>
<summary>Out-of-circuit provisioning option</summary>

The flash can also be programmed with a CH341A + SOIC-8 clip using a
full-image dump from a working board/card — useful for provisioning bare
flash chips before assembly on future builds.

</details>
</details>
