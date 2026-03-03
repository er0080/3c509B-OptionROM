# 3Com 3C5X9B TriROM v1.7 — 32KB Contiguous ROM Deployment

This repository contains the original 3Com TriROM v1.7 BIOS Option ROM image for
the **3Com EtherLink III (3C509B)** ISA network adapter, along with a patched image
for deployment on a generic 32KB parallel ROM chip mapped to a contiguous 32KB block
of upper memory, rather than the card's original two-page bank-switched ROM scheme.

---

## Files

```
3C5-TriROMv1.7.BIN          Original 32KB ROM image (unmodified)
3C5-TriROMv1.7-patched.BIN  Patched image for contiguous 32KB ROM deployment
README.md                   This file
```

---

## The ROM Image

| Property | Value |
|---|---|
| File | `3C5-TriROMv1.7.BIN` |
| Size | 32,768 bytes (32KB) |
| MD5 | `a09f3a5d5861a39bcf74d7e64e6b1bb9` |
| Product | 3Com 3C5X9 EtherLink III Ethernet 802.3 |
| Version | 1.7.0 |
| Build date | 1995-05-17 |
| Boot agent | Lanworks BootWare |

The ROM image is an x86 real-mode Option ROM containing Lanworks BootWare network
boot firmware. It provides remote boot capability (pre-PXE era) for ISA Ethernet
adapters in the 3C509/3C529 family.

---

## Original Card Architecture — Two 16KB Pages

On the 3C509B card, the onboard ROM socket is wired with a bank-switching scheme.
The card's I/O registers control which of the two 16KB pages is visible in upper
memory at any given time. The ROM image contains **two independent 16KB segments**,
each with its own standard Option ROM header:

| Page | Image offset | Declared size | Content |
|---|---|---|---|
| Page 0 | `0x0000` – `0x3FFF` | 8KB (`10h` blocks) | Boot loader / initializer stub |
| Page 1 | `0x4000` – `0x7FFF` | 8KB (`10h` blocks) | BootWare expansion image |

Both pages begin with the canonical x86 Option ROM signature `55 AA`, followed by
a size field of `10h` (16 × 512 = 8KB declared) and an `EB 4B` short-jump entry
point. Page 1 is a near-duplicate of page 0 with minor differences in bytes
`0x10–0x11` and `0x3A–0x49` (page identity fields).

---

## Initialization Sequence (Disassembly Notes)

The page 0 entry point is at offset `0x0050`. The full initialization sequence:

1. **Double-init guard** (`0x005E–0x006C`): checks a sentinel word at `[0x0308]`
   in low memory. If already set (`0x6B67` or `0x7967`), returns immediately. Sets
   the sentinel before proceeding, so the page 1 entry point (also executed by the
   BIOS when it finds the `55 AA` at offset `0x4000`) silently skips re-init.

2. **2KB stub copy** (`0x009D–0x00A4`): copies the first 2,048 bytes of page 0
   (offsets `0x000`–`0x7FF`) into conventional RAM at segment `0x5000` (linear
   `0x50000`). All subsequent init code runs from this RAM copy so that the ROM
   window can be reconfigured without losing execution context.

3. **Card detection** (`0x00DA` → `call 0x0354`): sends the 3Com 255-byte LFSR
   ID activation sequence to ISA port `0x110`, then reads the card's product ID
   register. The expected value is `0x6D50` (3C509B). On success, the card's
   EEPROM is read to obtain the I/O base address (`[0x003E]`) and address
   configuration word (`[0x023B]`), which encodes the ROM base segment and ROM
   size.

4. **ROM socket disable** (`0x0117–0x011F`): clears bits 8–9 of the card's
   Internal Configuration register (Window 3, `IO+4`), disabling the 3C509B's
   own ROM decode. After this, the card's onboard ROM socket no longer drives
   the memory bus regardless of whether a chip is installed in it.

5. **ROM base computation** (`0x0127–0x0135`): derives the ROM base segment from
   `[0x023B]` bits 11:8 using the formula `C000h + (bits × 2)`. For a card
   configured at `C800h`: bits = 4 → `C800h`. Result stored in `[0x0248]`.

6. **Expansion scan** (`0x019C` → `call 0x0744`): scans upper memory to verify
   the second ROM page is accessible. See the bug analysis below.

7. **Final jump** (`0x020D–0x0212`): computes the target ROM segment from
   `[0x023B]` bits 11:10, stores it in `[0x0248]`, then does a far `RETF` to
   `{[0x0248]}:0x0213`. Code at `ROM_seg:0x0213` installs `INT 19h` and `INT 18h`
   handlers pointing to `ROM_seg:0x0600` (the BootWare boot agent entry), then
   returns to the BIOS. The BIOS later invokes `INT 19h` to begin network boot.

---

## Bug Analysis — Contiguous ROM Incompatibilities

Bring-up testing revealed two bugs that prevent the ROM from initializing correctly
when deployed as a contiguous 32KB image.

### Bug 1 — "Unable to expand BootWare to 32K" (card detection, no patch needed)

**Symptom**: Error message displayed immediately on boot; ROM returns to BIOS after
`F1` is pressed.

**Cause**: The card detection routine at `0x0354` uses the ISA ID activation protocol
to find the 3C509B card at I/O port `0x110`. If no card is present (or the card does
not respond), the routine returns `AX = 0x0261`. Any non-zero return from `call 0x354`
at `0x00DA` causes the generic error message to be displayed, regardless of the
specific error code.

```
0x00DA  call 0x0354         ; detect NIC via ISA ID port 0x110
0x00DD  cmp  ax, 0x0000     ; AX=0 = success
0x00E0  jz   0x010D         ; skip error on success
0x00E2  test [cs:0x023B], 0x2000  ; check bypass flag
0x00E9  jnz  0x010D         ; skip error if bypass flag set
0x00EB  mov  bx, 0x028C     ; -> print "Unable to expand BootWare to 32K"
```

**Resolution**: Not a patchable issue. The 3C509B NIC **must be physically present**
in the system. The ROM directly drives 3C509B hardware registers for all Ethernet
operations; it cannot function without the card. See hardware requirements below.

---

### Bug 2 — Expansion scan misidentifies page 1 location (patch applied)

**Symptom**: If card detection succeeds, the ROM initializes but executes at the
wrong address (`D000:0213`) and crashes.

**Cause**: After disabling the card's ROM socket at `0x0117`, the expansion scan
function at `0x0744` searches upper memory to locate the second ROM page. It does
this by scanning segments in 32KB increments, looking for the first segment whose
second half (at `DS:0x4000`) is **blank** (`0xFF`). On the original bank-switched
card, disabling the ROM socket makes the entire C800:0000–7FFF window read `0xFF`,
so the first segment (C800h) passes the blank check — scan returns successfully with
`[0x0248] = C800h`.

With a contiguous external ROM chip, the card's ROM-disable has no effect on the
external chip. `C800:4000` contains page 1 content (`55 AA ...`) rather than `0xFF`,
causing the scan to advance to the next candidate segment (`D000h`). `D000h` is blank
RAM, so the scan incorrectly writes `D000h`-derived bits into `[0x023B]`, and the
final segment computation at `0x01BD` resolves `[0x0248]` to `D000h`. The far jump
goes to `D000:0213` — blank memory — crash.

```
; Inside function 0x0744 (called from 0x019C):
0x0785  call 0x07DA         ; search: scan DS:4000 for blank (0xFF) area
0x0788  jnz  0x07AA         ; ZF=0 = content found -> advance DS by 32KB, retry
                            ; ZF=1 = blank found   -> success (original card case)
; With contiguous ROM: C800:4000 = 55 AA... -> ZF=0 -> scans to D000h -> wrong
```

**Fix**: NOP out the `call 0x0744` at offset `0x019C`. Without the scan, `[0x0248]`
retains the correct value computed at `0x0132` directly from the card's EEPROM
(e.g., `C800h`). The `[0x023B]` configuration word is also left unmodified, so the
recomputation of `[0x0248]` at `0x01BD` produces the correct ROM base segment and
the final far jump lands at the correct address.

---

## The Patch

| Offset | Original bytes | Patched bytes | Change |
|---|---|---|---|
| `0x019C–0x019E` | `E8 A5 05` | `90 90 90` | `CALL 0x0744` → `NOP NOP NOP` |
| `0x001FFF` | `A2` | `84` | Page 0 checksum corrected |

The page 1 checksum at `0x5FFF` is unchanged (page 1 is not modified). The page 1
entry point is not patched because the double-init guard cookie set by page 0 causes
page 1's init to return silently before reaching any problematic code.

| File | MD5 |
|---|---|
| `3C5-TriROMv1.7.BIN` (original) | `a09f3a5d5861a39bcf74d7e64e6b1bb9` |
| `3C5-TriROMv1.7-patched.BIN` | `306bb9f7de6ce703b2b69ad3b140cd10` |

---

## Hardware Requirements

The following conditions must be met for the patched ROM to initialize and boot
correctly:

1. **3C509B NIC installed**: The card must be present and responding to the ISA ID
   activation sequence on port `0x110`. The ROM is a hardware driver for this
   specific card; it cannot boot without it.

2. **Card ROM socket empty or disabled**: The card's onboard ROM socket must either
   be unpopulated or have the ROM enable bit cleared in the card's EEPROM. If a chip
   is installed in the card's socket and enabled at the same address as the external
   chip, a bus conflict will occur before initialization can begin.

3. **EEPROM ROM base configured to match the external chip's decode address**: The
   card's EEPROM ROM base field (bits 11:8 of the Address Configuration word) must
   encode the segment where the external 32KB chip is mapped. For a chip at `C800h`:
   set ROM base = `C800h` in the EEPROM. The card's ROM size field should be set to
   32KB. Use `3C5X9CFG.EXE` (3Com DOS configuration utility) to set these values.

4. **Card's ROM socket disabled after EEPROM configuration**: The initialization
   code at `0x0117` disables the card's ROM socket as part of its normal sequence,
   so this is self-correcting at runtime — but the EEPROM must still encode a valid
   base address for the jump target computation to work correctly.

---

## ROM Header Details

Bytes `0x00–0x04` of each page follow the standard x86 Option ROM layout:

```
Offset  Len  Description
------  ---  -----------
0x00     2   Signature: 55 AA
0x02     1   Size in 512-byte blocks (10h = 16 blocks = 8KB per page)
0x03     2   Entry point jump: EB 4B (JMP SHORT +0x4D -> 0x0050)
```

Additional fields beginning at offset `0x05` contain BootWare-specific metadata:
the `BWA` identifier at `0x06`, version flags, and the `ELNK3.SYS` driver name
at `0x20`.

---

## Programming the ROM Chip

Flash the **patched** image to the target 32KB ROM chip:

```sh
# Example using minipro with a 27C256 (32KB OTP EPROM)
minipro -p AT27C256R -w 3C5-TriROMv1.7-patched.BIN

# Example using flashrom for a parallel flash device
flashrom -p <programmer> -w 3C5-TriROMv1.7-patched.BIN
```

Verify the written contents before installing:

```sh
minipro -p AT27C256R -r readback.bin && md5sum readback.bin 3C5-TriROMv1.7-patched.BIN
```

---

## References

- 3Com EtherLink III 3C509B adapter technical documentation
- ISA Option ROM specification (x86 real-mode, `55 AA` signature, 512-byte granularity)
- Lanworks BootWare remote boot agent (acquired by 3Com, ~1995)
- `ndisasm` (NASM package) used for 16-bit x86 disassembly of ROM image
