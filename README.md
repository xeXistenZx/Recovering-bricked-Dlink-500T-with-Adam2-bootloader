# Recovering a Bricked D-Link DSL-500T with ADAM2 Bootloader
## Complete Guide to Flashing OpenWrt via Serial + FTP
 
---

## Background

This guide documents the complete recovery process for a **D-Link DSL-500T EU Hardware Revision A1** (and similar AR7-based routers) that has been bricked by a bad firmware flash. The router uses the **ADAM2 bootloader** (Texas Instruments, v0.22.02) and has a 4MB flash chip.

### How it got bricked

This particular router was bricked approximately **15-20 years ago** during an attempt to upgrade the D-Link firmware. The wrong firmware was flashed — believed to be a **Russian (RU) regional firmware** instead of the correct **EU (European) version**. After the flash the router stopped booting completely, and was declared dead and put in a drawer.

Two decades later it was successfully recovered using nothing but a CP2102 serial adapter, a Windows PC, and a lot of patience — and is now running OpenWrt!

The key discoveries in this guide are **not documented anywhere else** and were reverse-engineered through extensive trial and error.

**Hardware:**
- Router: D-Link DSL-500T **EU version, Hardware Revision A1**
- CPU: TI AR7DB (MIPS 4KEc, 150MHz)
- RAM: 16MB
- Flash: 4MB
- Bootloader: ADAM2 v0.22.02
- Region: Europe (Annex A ADSL)

---

## What You Need

- **USB to Serial adapter** (CP2102 or similar as CH340xx, 3.3V)
- **Ethernet cable** (direct connection, no switch needed)
- **Windows PC** (the Windows FTP client behaves correctly with ADAM2)
- **PuTTY** for serial console
- **Firmware files** (see below)

---

## Step 1 — Serial Connection

Connect your CP2102 to the router's serial header pins:

| CP2102 | Router Pin |
|--------|-----------|
| TX     | RX        |
| RX     | TX        |   
| GND    | GND       |  
| **Do NOT connect 3.3V** | — |

TX & RX for for the router are pin 1 & 5. 
Router's pin 2 &  4 (tested using a multimeter)


**PuTTY settings:**
- Speed: `38400` (reads rubish wih other speeds)
- Data bits: `8`
- Stop bits: `1`
- Parity: `None`
- Flow control: `Hardware`

Power on the router. You should see:

```
ADAM2 Revision 0.22.02
Adam2_AR7DB >
Press any key to abort OS load, or wait 15 seconds for OS to boot...
```

Press any key to stay in the ADAM2 prompt.

---

## Step 2 — Network Setup

Set your PC to a static IP on the same subnet as the router:

- PC IP: `10.8.8.2` (or any `10.x.x.x`)
- Subnet: `255.0.0.0`
- Router IP: `10.8.8.8` (hardcoded in ADAM2)

Verify connectivity — **ADAM2 does not respond to ping**, but FTP works.

---

## Step 3 — Understanding ADAM2 Flash Layout

The 4MB flash chip is memory-mapped starting at `0x90000000`:

```
0x90000000 - 0x90010000  →  mtd2  Bootloader (ADAM2) — NEVER TOUCH
0x90010000 - 0x900e0000  →  mtd1  Kernel (~512KB)
0x900e0000 - 0x903f0000  →  mtd0  Filesystem (~3MB)
0x903f0000 - 0x90400000  →  mtd3  Config/NVRAM
```

> **Important:** `0x90xxxxxx` (cached) and `0xb0xxxxxx` (uncached) both point to the **same physical flash chip**. ADAM2 uses `0xb0` for writes.

---

## Step 4 — Understanding ADAM2 FTP Commands

This is the most critical section. The naming is counterintuitive:

| Command | What it actually does |
|---------|----------------------|
| `MEDIA FLSH` | **Writes to flash chip** — erases region then writes file |
| `MEDIA FLASH` | **Loads to RAM** — uploads file to RAM for execution, does NOT write to flash |

> **MEDIA FLSH** = FLaSH chip writing ✅  
> **MEDIA FLASH** = RAM upload (confusingly named) ✅

### The 21-Second Timeout Rule 

ADAM2 erases flash at approximately **1 second per 64KB block**. During erase, the FTP connection drops. If the erase takes too long the write never happens.

**Critical limits:**
- Kernel (~851KB = 13 blocks) → ~13 seconds → ✅ fits
- Filesystem (~1.7MB = 27 blocks) → ~20 seconds → ✅ just fits
- Full firmware (2.5MB = 40 blocks) → ~40 seconds → ❌ too long

> The FTP connection drops during erase but the erase continues and completes. ADAM2 then reconnects and writes. The total window is approximately **21 seconds**.

### Partition Size Rule

**The partition must be slightly larger than the file.** If the partition is smaller than the file, ADAM2 returns:
```
550 Store to media failed
```
This is NOT a checksum error — it simply means the file doesn't fit!

---

## Step 5 — Get the Firmware

Download OpenWrt Kamikaze 8.09.2 for AR7:

```
https://archive.openwrt.org/kamikaze/8.09.2/ar7/openwrt-ar7-squashfs.bin
```

This is a combined kernel+filesystem image (2.5MB). Split it into two files:

**On Windows PowerShell:**
```powershell
$data = [System.IO.File]::ReadAllBytes("C:\dlink\firmware.bin")
$kernel = $data[0..851967]
$fs = $data[851968..($data.Length-1)]
[System.IO.File]::WriteAllBytes("C:\dlink\kernel.bin", $kernel)
[System.IO.File]::WriteAllBytes("C:\dlink\fs.bin", $fs)
Write-Host "Kernel: $($kernel.Length) bytes"
Write-Host "FS: $($fs.Length) bytes"
```

You should get:
- `kernel.bin` — ~851KB
- `fs.bin` — ~1.7MB

---

## Step 6 — Set Up ADAM2 Environment Variables

From the serial console, set custom partition variables for flashing:

```
setenv zz 0x90010000,0x900e0000
setenv fs1 0x900e0000,0x902a0000
fixenv
printenv
```

Verify they appear in `printenv` with comma-separated format.

> **Why custom variables?** The built-in `mtd0`/`mtd1` variables may have corrupt duplicate entries from previous attempts. Using fresh variable names (`zz`, `fs1`) avoids parsing issues.

---

## Step 7 — Flash the Kernel

Create `C:\dlink\flash_kernel.txt`:
```
adam2
adam2
bin
quote MEDIA FLSH
put kernel.bin "fw zz"
quit
```

Run it:
```
ftp -s:C:\dlink\flash_kernel.txt 10.8.8.8
```

**Watch serial console** — you should see:
```
Erasing from 0xb0010000 to 0xb00e0000.
FlashEraseBlock(b0010000,b00dffff);
.............
Erase Successful.
```

Then FTP should complete with:
```
226 Transfer complete.
ftp: 851968 bytes sent in 10.02Seconds
```

**Verify it wrote:**
From serial console:
```
dm 0x90010000
```
You should see `feedfa42` as the first bytes — NOT `ffffffff`.

---

## Step 8 — Flash the Filesystem

Create `C:\dlink\flash_fs.txt`:
```
adam2
adam2
bin
quote MEDIA FLSH
put fs.bin "fw fs1"
quit
```

Run it:
```
ftp -s:C:\dlink\flash_fs.txt 10.8.8.8
```

This takes about 20 seconds. You should see:
```
226 Transfer complete.
ftp: 1769476 bytes sent in 20.61Seconds
```

**Verify:**
```
dm 0x900e0000
```
Should show `68737173` (`hsqs` = SquashFS magic).

---

## Step 9 — Boot!

From serial console, set autoload and boot:

```
setenv autoload 1
setenv autoload_timeout 5
fixenv
go
```

Watch the serial console. You should see Linux booting:

```
Linux version 2.6.26.8 ...
VFS: Mounted root (squashfs filesystem) readonly.
Please be patient, while OpenWrt loads ...
- preinit -
switching to jffs2
- init -

Please press Enter to activate this console.
```

**Press Enter** to get the OpenWrt shell:

```
root@OpenWrt:/#
```

🎉 **Success!**

---

## Step 10 — First Configuration

Set a password immediately:
```
passwd
```

Access via web browser:
```
http://192.168.2.1
```

Or via telnet (before setting password):
```
telnet 192.168.2.1
```

---

## Troubleshooting

### `550 <blockname> environment variable not set`
The variable name used in `"fw varname"` doesn't exist or has a corrupt duplicate. Use a fresh variable name (`setenv zz ...`).

### `550 Flash erase failed` / `Invalid flash address fffffff3`
The partition start address is not 64KB aligned. All addresses must end in `0000` (e.g. `0x90010000` not `0x9001fff4`).

### `550 Store to media failed`
The partition is slightly too small for the file. Increase the end address by 1-2 blocks (`0x10000` each).

### Connection drops during erase (no write happens)
Normal behaviour — ADAM2 drops FTP during erase. The erase continues. If your file fits within the ~21 second window, the write will complete after erase. If not, reduce partition size to reduce erase time.

### `File for wrong Endian!` at boot
Check `endian` variable — should match your hardware:
```
dm 0x90010000
```
If first bytes are `feedfa42` — kernel is little-endian. Set:
```
setenv endian le
fixenv
```

### Flash stays `ffffffff` after transfer
You used `MEDIA FLASH` (RAM mode) instead of `MEDIA FLSH` (flash mode). Redo with `quote MEDIA FLSH`.

### Kernel boots but panics (no filesystem)
The filesystem wasn't written, or was written to the wrong address. The kernel looks for SquashFS at flash offset `0xe0000` = address `0x900e0000`. Verify with `dm 0x900e0000`.

---

## Key Discoveries (not documented elsewhere)

1. **`MEDIA FLSH` ≠ `MEDIA FLASH`** — completely different operations
2. **~21 second erase timeout** — files must erase within this window
3. **Partition must be larger than file** — `Store to media failed` = file too big, not checksum error
4. **Do not split the filesystem** — gaps between chunks corrupt the filesystem
5. **Use custom env variables** — avoids corrupt duplicate `mtd` entries
6. **No checksum manipulation needed** — ADAM2 accepts files with valid `feedfa42` header
7. **Flash is memory-mapped** — `dm 0x90010000` reads flash directly, not RAM

---

## Flash Layout Reference

```
Physical Address    Size    Name        Contents
0x90000000         64KB    loader      ADAM2 bootloader (READ ONLY)
0x90010000         ~512KB  linux       Kernel (feedfa42 magic)
0x900e0000         ~3MB    rootfs      SquashFS filesystem (hsqs magic)
0x903f0000         64KB    config      NVRAM config
0x90400000         —       End of flash
```

---

## Environment Variables Reference

```
# Kernel partition
setenv zz 0x90010000,0x900e0000

# Filesystem partition (slightly larger than fs.bin)
setenv fs1 0x900e0000,0x902a0000

# Fix endianness
setenv endian le

# Enable autoboot
setenv autoload 1
setenv autoload_timeout 5

# Apply changes
fixenv
```

---

## Credits

This guide was written based on a real hands-on recovery session on a bricked DSL-500T. Every error message, timeout, and workaround was discovered through direct testing — not theory.

### The real story behind this guide

This router sat bricked for years after an accidental wrong firmware flash. The recovery took many hours of systematic debugging, reverse engineering ADAM2's behaviour, reading kernel source code, and pure persistence.

**AI assistance played a significant role in this recovery:**

- **Claude (Anthropic)** — Primary assistant throughout the entire session. Helped interpret every error message, suggested approaches, read the OpenWrt kernel source (`ar7part.c`) to understand partition detection, identified the `MEDIA FLSH` vs `MEDIA FLASH` distinction, and helped document the findings. Without Claude's persistent help across hundreds of attempts this guide would not exist.

- **ChatGPT (OpenAI)** — Consulted during various stages for additional perspectives on ADAM2 behaviour and AR7 architecture.

- **Google AI** — Used for quick lookups on specific error codes and flash protocol details.

However — and this is important — **the actual breakthroughs were all human insights**:

- Realising the filesystem partition just needed to be slightly larger than the file (not a checksum problem)
- Figuring out the 21-second erase timeout by observation
- Deciding to limit the partition size to reduce erase time
- Correctly diagnosing why the split filesystem was causing boot failures
- Persistent trial and error across dozens of attempts without giving up


**Tested on:** D-Link DSL-500T, ADAM2 v0.22.02, AR7DB chipset  
**OpenWrt version:** Kamikaze 8.09.2 (r18961)  
**Tools used:** Windows FTP client, PuTTY, CP2102 serial adapter, PowerShell, Python3

### Special mention

A huge thank you to **AliExpress** for making USB-to-serial adapters available for under £1 each. 

Some may call that being scammed. We call it the best £0.86 ever spent. 😄

Without these incredibly cheap little devices, hardware recovery projects like this would require expensive professional equipment. The democratisation of embedded hardware hacking is real — anyone can now talk to a bootloader for less than the price of a coffee! ☕

---

*If this guide saved your router, pay it forward — share it, star it, or help someone else on the forums!* 🎉
