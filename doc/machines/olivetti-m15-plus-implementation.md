# Olivetti M15 Plus engineering notes

## Outcome and evidence boundary

BluMach's first M15 Plus implementation is an **experimental dual-floppy
pilot**. It gives BIOS 1.10 a distinct 32 KB ROM map and reuses only those M15
family interfaces for which the Plus firmware shows the same guest-visible
contract. The 20 MB HDU configuration remains unavailable.

Here, *documented* means stated by contemporary Olivetti material, *observed*
means visible in static firmware analysis or a reproducible emulator test,
*inferred* means a design conclusion supported by several observations, and
*approximate* identifies a compatible substitute for unrecovered circuitry.
Reaching POST will not by itself prove the original motherboard has been
reconstructed.

## Why the machine is difficult

The operations guide describes user-facing hardware but provides no schematic,
service-level I/O map or controller identities. The surviving 32 KB BIOS has
unverified physical provenance and cannot be distributed. The Plus also has a
proprietary 20 MB HDU path, a backlit panel that differs physically from the
M15 display, and a brochure claim of 640×320 that conflicts with the guide's
320×200 and 640×200 BIOS graphics modes.

Treating the Plus as an M15 alias would hide those differences. Treating it as
a generic XT would lose the firmware's unusual memory-switch, RTC and video
contracts. The pilot therefore has a separate machine identity while sharing
small, evidence-backed behavioral components.

## From firmware to the component map

The 32 KB image ends with a far jump to `F000:8050h`, so it maps at
`F8000h-FFFFFh`. Its 128-glyph, 8×8 ASCII table begins at file offset `7A6Eh`,
which resolves to physical `FFA6Eh`: exactly the address used by the M15 BIOS
font loader. This permits direct font reads from mapped firmware with no copied
or generated font asset.

Disassembly shows immediate access to CGA/V6355D-compatible ports `03D4h`,
`03D8h`, `03D9h`, `03DDh` and `03DFh`, including the extended register path.
It also repeats the M15 sequence that toggles port `61h`, reads multiplexed
configuration nibbles at `62h`, and obtains startup display bits through
`60h`. Those observations justify the shared board-switch behavior and the
configurable 40/80-column startup selector.

The clock code asserts HOLD at `010Dh`, transfers BCD digits through
`0100h-010Ch`, and accesses the remaining control registers through `010Fh`.
That is the same guest-visible MSM6242 contract already implemented for M15.
The floppy path is provisionally supplied by the compatible XT FDC used by the
working M15 model.

By contrast, the HDU routines poll and command ports `0320h-0323h`. No current
evidence identifies their controller, DMA behavior, geometry translation or
interrupt routing. A generic XT-IDE or MFM card would make software run under a
fictional interface, so the pilot omits the HDU.

## Compromise ledger

| Subsystem | Pilot treatment | Replacement evidence |
|---|---|---|
| 80C88 and timing | Fixed 4.77 MHz 8088 core plus the calibrated M15-family PIT ratio. | Physical timer trace or a dedicated CMOS 80C88 timing model. |
| RAM and switches | Fixed documented 512 KB; firmware-observed multiplexed switch encoding. | Board schematic or switch table confirming electrical wiring. |
| BIOS and font | Exact local 32 KB mapping; ASCII glyphs read at `FFA6Eh`. No firmware is distributed. | A provenance-confirmed, redistributable dump would change packaging, not the map. |
| V6355D and LCD | Existing register-compatible controller with fixed green four-level presentation. | Panel/controller identification and traces for backlight, contrast, timings and the 640×320 claim. |
| RTC | Existing MSM6242 BCD/HOLD model at `0100h-010Fh`. | Chip identification or board trace if Plus control side effects differ. |
| Keyboard and board probe | Shared M15-family port and switch subset with separate Plus device identity. | Keyboard firmware, protocol capture or schematic. |
| Floppy, UART and LPT | Compatible XT-era devices on firmware-visible routes. | Controller identification and timing traces. |
| 20 MB HDU | Deliberately unavailable. | Controller identity, port semantics, IRQ/DMA routing and a safe disk geometry test. |

## Validation ladder

The static gate verifies the ROM size and reset address, font location,
switch-reading sequences, video ports, RTC range and HDU port boundary. A clean
Qt 6 build then cold-booted BIOS 1.10, passed every displayed Resident
Diagnostics item, reported `RAM 512/512 Pass`, loaded the original 720 KB
System Test disk into MS-DOS 3.20 and opened `M15PLUS SYSTEM TEST` version 1.00.
This validates the vertical slice through firmware, LCD text and floppy I/O;
it does not yet validate every test-menu subsystem.

The remaining runtime gates are the complete System Test menu, soft and hard
reset, the Keyboard Drivers & Utilities disk, and separate 40×25, 80×25,
320×200 and 640×200 display tests.

The HDU tests are intentionally excluded: running a generic controller would
validate the substitute, not the M15 Plus. No original media is mounted
writable, and proprietary firmware, manuals and disks remain outside Git.

## Rejected shortcuts and replacement criteria

The implementation rejects an M15 alias because it would erase the Plus ROM,
display and storage identity. It rejects a C000h video option ROM because the
system BIOS directly owns the video path. It rejects a generic 20 MB XT-IDE or
MFM template because firmware accesses a different register block. Finally, it
does not turn the brochure's 640×320 wording into a mode without a trace that
explains how software selects it.

A later revision should replace an approximation only when a schematic,
readable board photograph, physical trace, controller dump or repeatable
machine/software test establishes a stronger contract. Until then the narrow
dual-floppy model is easier to audit and less likely to fossilize a convenient
but false design.

## Implementation map

- `src/machine/m_xt.c`: 32 KB firmware mapping and shared M15-family setup;
- `src/machine/machine_table.c`: fixed CPU, RAM, video and floppy identity;
- `src/device/kbc_xt.c`: separate Plus device identity over the observed
  keyboard/switch behavior;
- `src/video/vid_cga_v6355.c`: fixed green V6355D-compatible LCD rendering;
- `src/qt/catalog/source/machines/olivetti/olivetti-m15-plus/`: multilingual
  sheet and declarative dual-floppy creation template.
