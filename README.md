# C64 Beamrace Lab

Engineering reference for PAL Commodore 64 screen double buffering, fine scrolling,
and beam-raced Color RAM updates using the MOS 6510 and VIC-II. Assembly examples
use ACME syntax.

**[Read the full engineering reference](docs/ENGINEERING_REFERENCE.md).**

## What this repository contains

The supplied “BEAMRACE LAB — Final Engineering Edition” reference is preserved
verbatim in `docs/ENGINEERING_REFERENCE.md`. Its 123 numbered sections cover:

- Screen matrix selection through `$D018` and VIC bank selection.
- Fine scrolling with `$D016` and coarse scrolling in a hidden screen buffer.
- Two ordinary-RAM colour shadows feeding the single physical Color RAM.
- Raster interrupts, commit protocols, and buffer-generation ownership.
- Unrolled colour copiers, code size, and nominal 6510 instruction timing.
- PAL badlines, BA/AEC stalls, sprite DMA, IRQ jitter, and per-cell deadlines.
- Validation methods, failure modes, and production optimization alternatives.

This is a documentation repository. The assembly fragments illustrate techniques;
there is no standalone assembled demo, PRG, or executable build target included.

## Central idea

The VIC-II can switch the displayed screen matrix, but it cannot switch physical
Color RAM at `$D800–$DBE7`. Software maintains colour shadows in ordinary RAM and
copies the selected values before the VIC-II fetches them. Correctness depends on
each write meeting its fetch deadline, including bus stalls and interrupt latency.
An average CPU-cycle budget alone does not prove a tear-free display.

## Reading the reference

Start with sections 1–5 for the memory model, then follow the scrolling, raster
commit, copier, and deadline discussions. Sections 121–123 collect the final audit
checklist, equations, and conclusion. Use GitHub's document outline to navigate the
full reference; its opening table of contents is an overview rather than a complete
index of all 123 sections.

### Known source inconsistency

The original document alternates between Screen B at `$0800` and `$0C00`.
With VIC bank 0 and charset selection bits `$04`, `$D018=$24` selects a screen
at `$0800`, while `$D018=$34` selects `$0C00`. Section 2 pairs `$0800` with `$34`
and later diagrams show `$0C00`. Choose a consistent memory map before adapting
the examples. The supplied reference has not been silently rewritten or validated
as a complete, cycle-exact program.

## Files

| File | Purpose |
| --- | --- |
| [Engineering reference](docs/ENGINEERING_REFERENCE.md) | Original supplied document |
| [NOTICE](NOTICE) | Attribution and source provenance |
| [LICENSE](LICENSE) | GNU GPL version 3 license text |

## License

Copyright (C) 2026 Ulf Bertilsson. Distributed under the GNU General Public
License, version 3 or later; see [LICENSE](LICENSE).
