# 3Com 3C5X9B TriROM v1.7 — 32KB Contiguous ROM Deployment

This repository contains the original 3Com TriROM v1.7 BIOS Option ROM image for
the **3Com EtherLink III (3C509B)** ISA network adapter. The goal of this project
is to deploy this image onto a generic 32KB parallel ROM chip mapped to a single
contiguous 32KB block of upper memory, rather than the card's original two-page
bank-switched ROM scheme.

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
memory at any given time. The ROM image therefore contains **two independent 16KB
segments**, each with its own standard Option ROM header:

| Page | Image offset | Content |
|---|---|---|
| Page 0 | `0x0000` – `0x3FFF` | Boot loader / initializer |
| Page 1 | `0x4000` – `0x7FFF` | BootWare expansion image |

Both pages begin with the canonical x86 Option ROM signature `55 AA`, followed by
a size field of `10h` (16 × 512 = 8KB declared, used as a page-size marker) and an
`EB 4B` short-jump entry point. The page 0 initializer locates page 1 by toggling
a card register, copies it into RAM, validates its checksum, and then executes it.
Internal error strings confirm this mechanism:

```
Unable to expand BootWare to 32K
32K BootWare checksum is invalid
ROMShield requires 32K ROMSize
```

---

## Target Architecture — Contiguous 32KB ROM

The target platform maps a single generic 32KB parallel ROM chip to a contiguous
32KB window in upper memory (e.g. `C000:0000` – `C000:7FFF` or similar). No bank
switching hardware is present or needed.

Because the full 32KB image is already laid out contiguously in the binary — page 0
at offset 0 and page 1 immediately following at offset 0x4000 — **no re-ordering or
interleaving of the image data is required**. The binary can be written to the ROM
chip as-is.

The key consideration for this deployment is the page-switching initialization
sequence in page 0. On the original card, the initializer toggles a 3C509B I/O
register to expose page 1. In the contiguous-ROM scenario:

- Page 1 is already visible alongside page 0 at a fixed address offset.
- The I/O register toggle targets the 3C509B card and is irrelevant here.
- Depending on how the initializer locates page 1 (fixed segment address vs.
  register-derived address), it may either find page 1 correctly at the expected
  offset or attempt a no-op register access that fails silently.

This behavior should be verified during bring-up. If the initializer fails to
locate or validate page 1 in the contiguous map, a targeted patch to the page 0
code will be needed to supply the correct segment address directly.

---

## ROM Header Details

Bytes 0–3 of each page follow the standard x86 Option ROM layout:

```
Offset  Len  Description
------  ---  -----------
0x00     2   Signature: 55 AA
0x02     1   Size in 512-byte blocks (10h = 16 blocks = 8KB per page)
0x03     2   Entry point jump: EB 4B (JMP SHORT +0x4D)
```

Additional fields beginning at offset `0x05` contain BootWare-specific metadata
including the `BWA` identifier, version flags, and the `ELNK3.SYS` driver name.

---

## Files

```
3C5-TriROMv1.7.BIN    Original 32KB ROM image, binary, ready to program
README.md             This file
```

---

## Programming the ROM Chip

Write the image verbatim to the target 32KB ROM chip:

```sh
# Example using minipro with a 27C256 (32KB OTP EPROM)
minipro -p AT27C256R -w 3C5-TriROMv1.7.BIN

# Example using flashrom for a parallel flash device
flashrom -p <programmer> -w 3C5-TriROMv1.7.BIN
```

Verify the written contents match the original image before installing:

```sh
minipro -p AT27C256R -r readback.bin && md5sum readback.bin 3C5-TriROMv1.7.BIN
```

---

## References

- 3Com EtherLink III 3C509B adapter technical documentation
- ISA Option ROM specification (x86 real-mode, `55 AA` signature, 512-byte granularity)
- Lanworks BootWare remote boot agent (acquired by 3Com, ~1995)
