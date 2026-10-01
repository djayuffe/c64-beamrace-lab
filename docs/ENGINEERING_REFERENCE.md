---
title: "BEAMRACE LAB"
subtitle: "Cycle-Aware C64 PAL Screen Double Buffering, Fine Scrolling, and Color-RAM Beam Racing"
version: "Final Engineering Edition"
target: "Commodore 64 PAL / MOS 6510 / VIC-II"
assembler: "ACME"
---

# BEAMRACE LAB

## Complete Engineering Reference

This is a self-contained technical reference for the C64 technique commonly described as **racing the beam** while combining screen double buffering with software-shadowed Color RAM.

The document deliberately separates three levels of truth:

1. **Architectural facts** — what the C64 hardware can and cannot switch.
2. **Nominal CPU timing** — instruction-cycle accounting before VIC-II bus steals.
3. **Deadline proof** — whether every required write completes before the corresponding VIC-II fetch.

That distinction prevents the common mistake of treating a convenient average-cycle calculation as a cycle-exact guarantee.

## Table of contents

1. Hardware problem and memory architecture
2. Screen RAM versus physical Color RAM
3. VIC bank and `$D018`
4. `$D016` fine scrolling
5. Hidden-buffer coarse scrolling
6. Software Color RAM shadows
7. Raster IRQ and commit point
8. Fully unrolled Color RAM copier
9. PAL cycle accounting
10. Badlines, BA/AEC, and DMA
11. Beam-race deadline model
12. IRQ latency and jitter
13. Memory-map correctness
14. Failure modes
15. Production optimizations
16. Validation methodology
17. Visual tutorial architecture
18. Final invariants and audit checklist

> **Terminology:** “double-buffered color” in this guide always means two *ordinary-RAM shadow images*. The physical C64 Color RAM itself is not double-buffered.


# 1. The problem

A standard 40×25 C64 character screen needs two kinds of information.

### Screen matrix

The 1000 screen bytes specify which character appears in each cell.

```text
40 columns × 25 rows = 1000 bytes
```

Example:

```text
$0400 = character at x=0, y=0
$0401 = character at x=1, y=0
...
$0427 = character at x=39, y=0

$0428 = character at x=0, y=1
...
```

### Color RAM

The corresponding 1000 Color RAM locations provide the per-cell foreground colors:

```text
$D800-$DBE7
```

Conceptually:

```text
SCREEN MATRIX                         COLOR RAM

$0400  char 0  -------------------  $D800 color 0
$0401  char 1  -------------------  $D801 color 1
$0402  char 2  -------------------  $D802 color 2
 ...
$07E7  char 999 -------------------  $DBE7 color 999
```

The difficulty is that these two memory systems behave differently.

---

# 2. Screen RAM can be switched

The VIC-II can select different screen matrices through `$D018`.

For this demonstration:

```text
SCREEN A = $0400
SCREEN B = $0800
```

Both are inside VIC bank 0.

The active display can therefore be changed approximately like this:

```asm
lda #$14
sta $d018       ; SCREEN A

; later

lda #$34
sta $d018       ; SCREEN B
```

This gives us real screen double buffering:

```text
                 $D018

             +-----+-----+
             |           |
             v           v

       +-----------+  +-----------+
       | SCREEN A  |  | SCREEN B  |
       |   $0400   |  |   $0C00   |
       +-----------+  +-----------+

          visible        hidden

                  then

          hidden         visible
```

While A is visible, B can be prepared.

While B is visible, A can be prepared.

The VIC-II only sees the screen selected by `$D018`.

---

# 3. Color RAM cannot be switched

Color RAM is different.

The physical color memory remains:

```text
$D800-$DBE7
```

Changing `$D018` does **not** select another Color RAM page.

There is no hardware equivalent of:

```text
COLOR PAGE A
COLOR PAGE B
```

for normal C64 Color RAM.

Therefore this conceptual design does not exist:

```text
SCREEN A + COLOR A
        ↕
SCREEN B + COLOR B
```

at the hardware level.

Instead we create the missing buffers ourselves in ordinary RAM.

---

# 4. Software Color RAM shadows

Reserve two normal-RAM areas:

```text
COLOR_A = $C000
COLOR_B = $C400
```

Now each logical screen has a matching logical color image:

```text
+-------------------+             +-------------------+
| SCREEN A          |             | SCREEN B          |
| $0400             |             | $0C00             |
+---------+---------+             +---------+---------+
          |                                 |
          | logical pair                    | logical pair
          |                                 |
+---------v---------+             +---------v---------+
| COLOR_A           |             | COLOR_B           |
| $C000             |             | $C400             |
| normal RAM shadow |             | normal RAM shadow |
+---------+---------+             +---------+---------+
          \                                 /
           \                               /
            +-------------+---------------+
                          |
                          | CPU transfer
                          v
                +-------------------+
                | PHYSICAL COLOR RAM|
                | $D800-$DBE7       |
                +-------------------+
                          |
                          v
                        VIC-II
```

The important word is **shadow**.

`COLOR_A` and `COLOR_B` are not Color RAM from the VIC-II's perspective. They are ordinary RAM holding the color values that *should eventually appear* in `$D800-$DBE7`.

---

# 5. Complete memory model

A useful memory layout is:

| Address | Purpose |
|---|---|
| `$0400-$07FF` | Screen A |
| `$0801-$080C` | BASIC loader stub (needed only before machine-code entry) |
| `$0800-$0BFF` | Screen B after startup |
| `$2000...` | Main machine-code body, including generated copiers |
| `$C000-$C3FF` | Color shadow A |
| `$C400-$C7FF` | Color shadow B |
| `$D800-$DBE7` | Physical Color RAM |
| `$0314/$0315` | KERNAL IRQ vector |
| `$D011` | VIC control register 1 |
| `$D012` | raster counter / compare low byte |
| `$D016` | VIC control register 2 / X fine scroll |
| `$D018` | VIC memory pointers |
| `$D019` | VIC IRQ status/acknowledge |
| `$D01A` | VIC IRQ enable |
| `$DD00` | CIA2 VIC-bank selection |

Using `$0C00` for Screen B is deliberate.

A BASIC-startable machine-language program commonly begins around `$0801/$0810`. Putting Screen B at `$0800` would collide with the executable.

---

# 6. What `$D018` actually does

`$D018` selects the screen-matrix and character-data offsets inside the currently selected VIC bank.

Conceptually:

```text
$D018

bits 7..4  -> screen matrix
bits 3..1  -> character data
```

In this design:

```text
$14 -> screen $0400, character source at the selected $1000 position
$24 -> screen $0800, same character source selection
```

The crucial distinction is:

| `$D018` does | `$D018` does not |
|---|---|
| select a screen matrix | switch Color RAM |
| select character data | copy screen data |
| change VIC memory interpretation | create a second `$D800` |
| allow screen-buffer flipping | synchronize CPU writes automatically |

So this:

```asm
lda #$34
sta $d018
```

can switch the screen matrix immediately, but the Color RAM still contains whatever was previously in `$D800-$DBE7`.

That is the problem the beam-raced transfer solves.

---

# 7. VIC bank selection

The VIC-II sees one 16 KiB bank at a time.

CIA2 port A bits 0 and 1 control that bank, using inverted encoding.

For bank 0:

```asm
lda $dd00
ora #%00000011
sta $dd00
```

With VIC bank 0 selected, the addresses used by the example are accessible in the intended VIC address space.

`$D018` offsets are relative to the VIC bank. Therefore bank selection and `$D018` must always be considered together.

---

# 8. Horizontal scrolling has two layers

A smooth character-mode horizontal scroller normally combines:

1. **fine scrolling** — pixel movement inside an 8-pixel character width;
2. **coarse scrolling** — moving the character matrix by one whole character.

Fine scrolling alone can move only within the 0–7 pixel range.

Coarse scrolling alone jumps eight pixels at a time.

Together they produce continuous motion.

---

# 9. `$D016` fine scrolling

Bits 0–2 of `$D016` contain the horizontal fine-scroll value.

A safe update preserves the other bits:

```asm
lda $d016
and #%11111000
ora fine_scroll
sta $d016
```

A left-moving sequence can use:

```text
7
6
5
4
3
2
1
0
```

Visually:

```text
fine=7   [A][B][C][D][E][F]...
fine=6    [A][B][C][D][E][F]...
fine=5     [A][B][C][D][E][F]...
fine=4      [A][B][C][D][E][F]...
fine=3       [A][B][C][D][E][F]...
fine=2        [A][B][C][D][E][F]...
fine=1         [A][B][C][D][E][F]...
fine=0          [A][B][C][D][E][F]...
```

At the coarse boundary, the logical character row is changed:

```text
before:

[A][B][C][D][E][F]...

after:

[B][C][D][E][F][G]...
```

and fine scroll returns to 7.

---

# 10. Never coarse-scroll the visible buffer

Suppose Screen A is currently visible.

Do **not** shift Screen A while the VIC-II is displaying it.

Instead:

```text
VISIBLE
SCREEN A
COLOR currently in $D800

HIDDEN
SCREEN B
COLOR_B
```

The CPU constructs the next state entirely in B.

For example:

```text
SCREEN B before:

A B C D E F G H ...

shift left:

B C D E F G H ...

append new character:

B C D E F G H ... Q
```

The matching color shadow must undergo the same logical transformation:

```text
COLOR_B before:

cA cB cC cD cE ...

after:

cB cC cD cE ... cQ
```

Only after both are ready do we mark the buffer for display.

---

# 11. Hidden-buffer state machine

The basic state machine is:

```text
active = A
hidden = B

        |
        v

fine scroll 7 -> 0

        |
        v

coarse boundary reached

        |
        v

shift SCREEN B
shift COLOR_B
append next character
append next color

        |
        v

swap_pending = 1

        |
        v

lower-border raster IRQ

        |
        +---- switch $D018 to B
        |
        +---- copy COLOR_B -> $D800

        |
        v

active = B
hidden = A
```

Then the roles reverse.

---

# 12. Why copying all 1000 colors is difficult

A straightforward Color RAM transfer has to move:

```text
40 × 25 = 1000 cells
```

A maximally simple deterministic transfer for one cell is:

```asm
lda COLOR_A+n
sta $d800+n
```

Both use absolute addressing.

6510 timing:

```text
LDA absolute = 4 cycles
STA absolute = 4 cycles
```

Therefore:

```text
one cell = 8 cycles
```

One 40-column row:

```text
40 × 8 = 320 cycles
```

Full 1000-cell image:

```text
1000 × 8 = 8000 cycles
```

That is before counting the `JSR` into the copier and the final `RTS`.

---

# 13. Why the copier is unrolled

Instead of a runtime loop, ACME can generate every load/store pair at assembly time:

```asm
copy_color_a:

!for .i, 0, 999 {
    lda COLOR_A + .i
    sta $d800 + .i
}

    rts
```

This is an **assembler loop**, not a 6510 runtime loop.

ACME expands it into machine code equivalent to:

```asm
lda $c000
sta $d800

lda $c001
sta $d801

lda $c002
sta $d802

lda $c003
sta $d803

; ...

lda $c3e7
sta $dbe7

rts
```

There is no:

```asm
inx
cpx
bne
```

inside the transfer.

There is also no indexed-address page-crossing issue in the generated body.

The transfer order and instruction timing are therefore exceptionally easy to reason about.

---

# 14. Code-size cost of unrolling

Each instruction is three bytes:

```text
LDA absolute = 3 bytes
STA absolute = 3 bytes
```

Therefore one cell requires:

```text
6 bytes of code
```

For 1000 cells:

```text
1000 × 6 = 6000 bytes
```

plus `RTS`.

If separate A and B copiers are generated:

```text
~6001 bytes for A
~6001 bytes for B
```

so roughly 12 KiB is deliberately traded for simple deterministic timing.

That is not necessarily the best production implementation. It is an excellent teaching implementation because every transfer is explicit.

---

# 15. PAL raster timing

A PAL C64 frame has 312 raster lines.

A commonly used timing model for a standard PAL VIC-II is:

```text
63 CPU cycles per raster line
```

The 8000-cycle transfer therefore corresponds to:

```text
8000 / 63
= 126.984...
```

or about:

```text
127 raster lines
```

This immediately proves something important:

> The complete 1000-byte transfer cannot simply be squeezed into the lower border.

It is much too long.

---

# 16. The wrong mental model

The naive model is:

```text
WAIT FOR BORDER
     |
     v
COPY ALL 1000 COLORS
     |
     v
MUST FINISH BEFORE FRAME WRAPS
```

That imposes one huge deadline on the entire transfer.

It is unnecessary.

The VIC-II does not consume all 1000 displayed color cells simultaneously.

---

# 17. The correct deadline model

The display needs different color rows at different times.

Therefore the real requirements are approximately:

```text
color row 0 complete before VIC needs row 0
color row 1 complete before VIC needs row 1
color row 2 complete before VIC needs row 2
...
color row 24 complete before VIC needs row 24
```

Mathematically, for each row `r`:

```text
completion_time(copy_row[r])
<
VIC_fetch_deadline[r]
```

This is the key idea.

The entire 8000-cycle operation does **not** need to finish before the frame begins.

Each part merely has to finish before *its own* display deadline.

---

# 18. Why top-to-bottom order matters

The earliest display deadline belongs to the uppermost row.

The latest deadline belongs to the bottom rows.

Therefore transfer order should match deadline order:

```text
row 0
row 1
row 2
row 3
...
row 24
```

This is effectively an earliest-deadline-first schedule.

Memory layout conveniently already has that order:

```text
$D800-$D827    row 0
$D828-$D84F    row 1
$D850-$D877    row 2
...
```

So monotonically increasing addresses naturally copy the most urgent data first.

---

# 19. Why bottom-to-top is dangerous

Consider:

```text
row 24
row 23
...
row 1
row 0
```

The CPU begins in the lower border.

The frame wraps.

The VIC-II starts approaching the top of the new visible frame.

But row 0 has deliberately been postponed until the very end of the copy.

That creates exactly the wrong relationship:

```text
VIC needs row 0
        |
        v
-------------------- time -------------------->

                              CPU finally copies row 0
                                      |
                                      v
```

The display may fetch stale colors before the CPU replaces them.

Top-to-bottom fixes the scheduling order.

---

# 20. Racing across frame wrap

Assume the raster IRQ begins around PAL raster line 252.

Conceptually:

```text
RASTER 252
+---------------------------------------------------+
| IRQ                                               |
| switch screen                                    |
| copy color row 0                                 |
| copy color row 1                                 |
| copy color row 2                                 |
| ...                                               |
|                                                   |
RASTER 311
+---------------- FRAME WRAP -----------------------+
RASTER 000
|                                                   |
| copy continues                                   |
|                                                   |
| CPU has already dealt with the most urgent rows  |
|                                                   |
|             CPU COPY ----------------------->     |
|                                                   |
|                    VIC DISPLAY ------------->     |
|                                                   |
+---------------------------------------------------+
```

The fact that the copy crosses frame wrap is intentional.

The relevant question is not:

```text
"Did the copy finish before raster 0?"
```

It is:

```text
"Was every color cell ready before the VIC-II needed that cell?"
```

---

# 21. Character-row throughput

A character is eight raster lines high.

Therefore one character-row interval represents approximately:

```text
8 × 63 = 504 raw cycles
```

A normal badline consumes VIC-II bus time. A useful first-order model subtracts about 40 CPU cycles:

```text
504 - 40 = 464 cycles
```

The unrolled copier needs:

```text
40 cells × 8 cycles = 320 cycles
```

to advance one complete color row.

So the simplified throughput margin is:

```text
464 - 320 = 144 cycles
```

per character-row interval.

This demonstrates why the copier can advance through rows faster than the display progresses through character rows in this sprite-free teaching case.

However, this number should be treated as a **first-order engineering model**, not a complete cycle-exact proof.

---

# 22. Why 144 cycles is not the whole proof

Real VIC-II bus ownership is more detailed than:

```text
63 cycles every line, minus one 40-cycle block
```

A rigorous implementation must consider:

- exact raster line where the IRQ begins;
- exact cycle where the IRQ handler begins executing;
- IRQ entry overhead;
- badline BA/AEC behavior;
- the exact location of each memory access;
- sprite DMA;
- other interrupt sources;
- music-player execution;
- additional raster effects;
- PAL versus NTSC;
- VIC-II revision/timing assumptions.

The real correctness criterion remains:

```text
for every required cell or row:

CPU write completion
<
VIC read deadline
```

A production-quality timing audit should verify that relationship against the actual generated machine code.

---

# 23. Raster interrupt setup

For a raster number below 256, `$D011` bit 7 must be clear.

Example for raster 252:

```asm
lda $d011
and #$7f
sta $d011

lda #252
sta $d012
```

Then clear a pending raster interrupt and enable it:

```asm
lda #$01
sta $d019

lda #$01
sta $d01a
```

The VIC-II can now request an IRQ when the raster comparator matches.

---

# 24. `$D019` acknowledgement

Inside the IRQ:

```asm
lda #$01
sta $d019
```

acknowledges the raster interrupt.

Failing to acknowledge the VIC source correctly can cause immediate re-entry or an interrupt that appears stuck.

This operation should happen predictably near the beginning of the handler.

---

# 25. KERNAL IRQ vector

The demonstration installs its handler through:

```text
$0314/$0315
```

For example:

```asm
lda #<irq
sta $0314

lda #>irq
sta $0315
```

When using the normal KERNAL IRQ entry path, the KERNAL entry code has already performed register-saving work before dispatching through this vector.

The demo can eventually leave through the corresponding KERNAL restore/RTI path:

```asm
jmp $ea81
```

This detail matters: IRQ code must be designed according to the interrupt entry mechanism actually being used. A raw hardware vector implementation under ROM banking would have different responsibilities.

---

# 26. CIA interrupts

For a tightly controlled raster demonstration, unrelated CIA interrupt sources can be disabled:

```asm
lda #$7f
sta $dc0d
sta $dd0d

lda $dc0d
lda $dd0d
```

This keeps the VIC raster IRQ as the primary timing event.

In a larger application, CIA interrupts may be required. In that case their execution time and ordering become part of the timing budget.

---

# 27. Complete coarse-scroll transaction

Assume:

```text
active = A
hidden = B
fine   = 2
```

### Fine step

```text
frame N:

fine 2 -> 1
$D016 low bits = 1
```

Next update:

```text
fine 1 -> 0
$D016 low bits = 0
```

At the next boundary:

```text
hidden SCREEN B before:

A B C D E F G H I ...

hidden COLOR_B before:

1 2 3 4 5 6 7 8 9 ...
```

Shift both:

```text
SCREEN B:

B C D E F G H I ...

COLOR_B:

2 3 4 5 6 7 8 9 ...
```

Append a new character and its color:

```text
SCREEN B:

B C D E F G H I ... Q

COLOR_B:

2 3 4 5 6 7 8 9 ... 13
```

Then:

```text
fine = 7
swap_pending = 1
```

At the controlled swap point:

```text
$D018 -> SCREEN B
COLOR_B -> $D800
active = B
```

Now A becomes the next hidden work buffer.

---

# 28. Example shift routine

For one 40-column scrolling row:

```asm
shift_row_b:

    ldx #0

.loop:
    lda ROW_B+1,x
    sta ROW_B,x

    lda CROW_B+1,x
    sta CROW_B,x

    inx
    cpx #39
    bne .loop

    rts
```

Notice that screen code and color shadow are shifted together.

That preserves their logical relationship.

---

# 29. Appending a new character

After shifting 39 cells:

```asm
jsr next_char
sta ROW_B+39

jsr color_for_char
sta CROW_B+39
```

The newly exposed rightmost character and its corresponding color are now part of the same hidden-buffer state.

This is preferable to treating character and color updates as unrelated operations.

---

# 30. Buffer ownership invariant

At all times, there should be a clear owner/state relationship:

```text
VISIBLE BUFFER:
    VIC owns its presentation state.
    CPU should avoid destructive coarse reconstruction.

HIDDEN BUFFER:
    CPU owns it.
    CPU may shift, redraw, recolor, append, etc.
```

Then:

```text
hidden complete
      |
      v
commit / swap
      |
      v
old visible becomes new hidden
```

This is the same high-level idea used by double-buffered graphics systems on much newer hardware.

---

# 31. Logical atomicity

`$D018` switching itself is only one register write, but the *logical frame* consists of more than that register.

The state being committed is conceptually:

```text
{
    screen_matrix,
    color_image,
    fine_scroll_phase
}
```

Because Color RAM cannot switch atomically with `$D018`, the beam-raced transfer is effectively a scheduled completion of that logical commit.

The software must therefore guarantee that the partially transferred physical Color RAM is never observed by the VIC at a cell whose deadline has already passed.

---

# 32. Fine-scroll update cycle example

A typical non-boundary path might look like:

```asm
lda fine          ; 3
bne .fine         ; 3 when taken

.fine:
dec fine          ; 5

lda $d016         ; 4
and #$f8          ; 2
ora fine          ; 3
sta $d016         ; 4

rts               ; 6
```

Approximate total for this simplified path:

```text
3 + 3 + 5 + 4 + 2 + 3 + 4 + 6
= 30 cycles
```

The caller's `JSR` adds another six cycles.

The coarse boundary is substantially more expensive because it performs memory movement and text generation.

That is why a complete raster budget should distinguish fine-only frames from coarse-update frames.

---

# 33. The fully unrolled copier

The central demonstration routine is conceptually:

```asm
copy_color_a:

!for .i, 0, 999 {
    lda COLOR_A + .i
    sta COLOR_RAM + .i
}

    rts
```

Expanded beginning:

```asm
lda $c000       ; 4
sta $d800       ; 4

lda $c001       ; 4
sta $d801       ; 4

lda $c002       ; 4
sta $d802       ; 4
```

Expanded ending:

```asm
lda $c3e5
sta $dbe5

lda $c3e6
sta $dbe6

lda $c3e7
sta $dbe7

rts
```

The copy body itself is:

```text
8000 cycles
```

The final:

```asm
rts
```

costs six cycles.

The caller's:

```asm
jsr copy_color_a
```

costs another six.

So invocation plus body plus return is approximately:

```text
6 + 8000 + 6
= 8012 cycles
```

not including the surrounding IRQ logic.

---

# 34. Per-row copier deadlines

The transfer completes rows at regular 320-cycle intervals.

Ignoring surrounding overhead for illustration:

| Row | Bytes | Cumulative copy cycles |
|---:|---:|---:|
| 0 | 40 | 320 |
| 1 | 80 | 640 |
| 2 | 120 | 960 |
| 3 | 160 | 1280 |
| 4 | 200 | 1600 |
| 5 | 240 | 1920 |
| 6 | 280 | 2240 |
| 7 | 320 | 2560 |
| 8 | 360 | 2880 |
| 9 | 400 | 3200 |
| 10 | 440 | 3520 |
| 11 | 480 | 3840 |
| 12 | 520 | 4160 |
| 13 | 560 | 4480 |
| 14 | 600 | 4800 |
| 15 | 640 | 5120 |
| 16 | 680 | 5440 |
| 17 | 720 | 5760 |
| 18 | 760 | 6080 |
| 19 | 800 | 6400 |
| 20 | 840 | 6720 |
| 21 | 880 | 7040 |
| 22 | 920 | 7360 |
| 23 | 960 | 7680 |
| 24 | 1000 | 8000 |

This table makes the deadline nature of the algorithm much clearer.

Row 0 does not wait 8000 cycles. It is complete after only about 320 transfer cycles.

Row 1 is complete after 640.

And so on.

---

# 35. Visual deadline diagram

```text
time ------------------------------------------------------------>

CPU:
ROW0 ███
ROW1    ███
ROW2       ███
ROW3          ███
ROW4             ███
...

VIC:
                         ^ row 0 needed
                              ^ row 1 needed
                                   ^ row 2 needed
                                        ^ row 3 needed

Requirement:

end(ROW0 copy) < row0 VIC deadline
end(ROW1 copy) < row1 VIC deadline
end(ROW2 copy) < row2 VIC deadline
...
```

The safety property is local and monotonic.

If the CPU starts with enough initial lead and gains rather than loses lead per character-row interval, the transfer remains ahead.

---

# 36. What a badline changes

On a badline the VIC-II needs extra bus accesses to fetch display matrix/color information.

That means the CPU cannot assume unrestricted access to every nominal cycle.

This is why a statement such as:

```text
63 cycles × number of lines
```

is only a raw upper bound.

The VIC-II's BA/AEC behavior determines when the CPU actually loses bus access.

For a pedagogical no-sprite example, subtracting the normal badline cost gives a useful intuitive throughput model.

For cycle-critical production code, use an exact VIC timing model or validate the actual trace in an emulator/hardware setup.

---

# 37. Sprite DMA

Sprites make the analysis more demanding.

Enabled sprites consume additional VIC-II bus cycles on raster lines where sprite DMA is active.

The relevant budget becomes:

```text
available raster time

- badline DMA
- sprite DMA
- IRQ entry/exit
- scroll logic
- music player
- other effects
- color copy

>= required deadline margin
```

Therefore the no-sprite example should not be blindly reused inside an eight-sprite multiplexing routine without recalculating timing.

---

# 38. Music playback

A demo often calls music once per frame.

If music is called before the beam-raced copier:

```text
IRQ
 |
 +-- acknowledge
 |
 +-- music
 |
 +-- screen swap
 |
 +-- color copy
```

then the music routine delays every color deadline by its execution time.

If the music player has variable runtime, it also changes the starting phase of the copier.

A robust production design can:

- move music to another raster region;
- reserve a worst-case music budget;
- use a stable player;
- start the color transfer earlier;
- transfer only dirty color regions.

---

# 39. Dirty-row optimization

The demonstration copies all 1000 cells because it illustrates the general mechanism.

Many real effects do not need that.

If only one row's colors change:

```text
40 cells × 8 cycles
= 320 cycles
```

That is dramatically cheaper than:

```text
8000 cycles
```

A 25-bit dirty-row mask could represent which rows need transfer.

Conceptually:

```text
dirty_rows:

bit 0  -> row 0 changed
bit 1  -> row 1 changed
...
bit 24 -> row 24 changed
```

Only those rows would be copied.

---

# 40. Constant-color optimization

If the scroller always uses one foreground color, Color RAM may need no per-frame copying at all.

Initialize it once:

```asm
lda #LIGHT_BLUE
ldx #0

.loop:
    sta $d800,x
    sta $d900,x
    sta $da00,x
    ; handle final range appropriately

    inx
    bne .loop
```

Then leave it unchanged.

In that case the screen matrix alone needs double buffering.

This is far cheaper than solving a problem the visual effect does not actually have.

---

# 41. One-row scroller optimization

For a conventional single-line scroller, only the scrolling row may require dynamic colors.

Then maintain only:

```text
40 bytes of shadow color
```

and transfer that row at a controlled raster position.

The full-screen 1000-byte implementation is useful as a stress/general-case demonstration, not necessarily the optimal solution for a single-line production scroller.

---

# 42. One copier instead of two

The educational implementation may contain:

```text
copy_color_a
copy_color_b
```

each fully unrolled.

That gives extremely obvious code but costs around 12 KiB.

A production implementation could use one generated copy body with a self-modified source address.

Conceptually:

```asm
lda SOURCE+0
sta $d800+0

lda SOURCE+1
sta $d800+1

...
```

and patch the source-page operands according to the active shadow.

The tradeoff is:

```text
less code
vs.
more setup complexity
```

The best choice depends on the demo's memory and timing constraints.

---

# 43. PAL versus NTSC

This document's timing discussion is PAL-specific.

Do not assume:

```text
PAL raster number
PAL line count
PAL cycles/line
PAL visible geometry
```

are automatically valid for NTSC.

An NTSC implementation should independently derive:

- frame geometry;
- raster IRQ position;
- CPU/VIC timing;
- available lead;
- row deadlines.

The algorithmic principle remains valid, but the constants change.

---

# 44. Failure mode: visible-buffer modification

Bad:

```text
VIC displays A
CPU shifts A
```

Possible result:

```text
left part = new state
right part = old state
```

depending on when the VIC fetches cells relative to the CPU writes.

Correct:

```text
VIC displays A
CPU constructs B
commit B only after it is complete
```

---

# 45. Failure mode: screen/color mismatch

Bad:

```text
SCREEN B updated
COLOR_B still represents previous content
$D018 switched to B
```

Now the characters and colors describe different logical frames.

Treat them as one logical object:

```text
BUFFER B = {
    SCREEN_B,
    COLOR_B
}
```

Only commit after both components are ready.

---

# 46. Failure mode: bottom-to-top Color RAM transfer

Bad:

```text
$DBE7 -> $D800
```

This prioritizes the least urgent display rows.

Correct:

```text
$D800 -> $DBE7
```

This prioritizes the earliest display deadlines.

---

# 47. Failure mode: assuming border equals unlimited CPU time

The border is useful because the VIC is not displaying normal character rows there, but the amount of border time is finite.

A transfer that takes roughly 127 raster lines obviously cannot fit entirely inside a small lower-border region.

The beam-racing method works specifically because it does **not** require that.

---

# 48. Failure mode: ignoring code placement

A common example layout is:

```text
program around $0801
screen A $0400
screen B $0800
```

That is a collision.

If executable bytes occupy `$0800-$0BFF`, using the same area as a screen destroys either code or display data.

The example therefore uses:

```text
SCREEN A = $0400
SCREEN B = $0800
```

---

# 49. Failure mode: confusing assembler time with runtime

This:

```asm
!for .i,0,999 {
    ...
}
```

is processed by ACME on the development machine.

The C64 never executes `!for`.

The actual C64 receives thousands of generated `LDA` and `STA` instructions.

That distinction is important when calculating runtime cycles.

---

# 50. Failure mode: unbounded IRQ work

If an IRQ contains:

```text
music
animation
sprite sorting
multiplexer setup
screen generation
color copy
debug code
```

without a cycle budget, the color copier may start too late.

Raster code should be designed as a schedule, not merely as a list of functions.

---

# 51. A better timing mindset

Instead of asking:

> “How many things can I put in the IRQ?”

ask:

> “What is the deadline for every operation, and what is its worst-case execution time?”

For example:

```text
operation                 deadline
------------------------------------------------
set next raster           before handler exits
switch $D018              before target display
Color row 0               before row 0 fetch
Color row 1               before row 1 fetch
music                     before next audio tick
sprite pointers           before sprite DMA uses them
```

This turns raster programming into a scheduling problem.

---

# 52. Complete conceptual frame

```text
                         FRAME N

                   VIC DISPLAYING A
                          |
                          |
                 fine-scroll updates
                          |
                          v
                 coarse boundary?
                    /          \
                  no            yes
                  |              |
                  |       build hidden B
                  |              |
                  |       shift SCREEN_B
                  |       shift COLOR_B
                  |       append char
                  |       append color
                  |              |
                  |       swap_pending=1
                  |              |
                  +-------+------+
                          |
                          v
                    raster 252
                          |
                          v
                    raster IRQ
                          |
                    acknowledge
                          |
                          v
                     $D018=B
                          |
                          v
                  COLOR_B -> $D800
                    top to bottom
                          |
                          v
                     frame wraps
                          |
                  transfer continues
                          |
                          v
                  VIC reaches display
                          |
                  required rows ready
                          |
                          v
                     FRAME N+1

                   VIC DISPLAYING B
                          |
                    A is now hidden
```

---

# 53. Core code skeleton

A compact architecture looks like:

```asm
irq:
    lda #1
    sta $d019

    jsr scroll_tick

    lda swap_pending
    beq .done

    lda #0
    sta swap_pending

    lda active_buffer
    eor #1
    sta active_buffer

    beq .show_a

.show_b:
    lda #D018_B
    sta $d018

    jsr copy_color_b
    jmp .done

.show_a:
    lda #D018_A
    sta $d018

    jsr copy_color_a

.done:
    jmp $ea81
```

This is intentionally easy to understand.

A more advanced implementation may split duties across multiple raster IRQs or use more exact synchronization.

---

# 54. Scroll code skeleton

```asm
scroll_tick:

    lda fine_scroll
    bne .fine_only

    lda active_buffer
    beq .build_b

.build_a:
    jsr shift_row_a
    jsr append_char_a
    jmp .built

.build_b:
    jsr shift_row_b
    jsr append_char_b

.built:
    lda #7
    sta fine_scroll

    lda #1
    sta swap_pending

    jmp .write_d016

.fine_only:
    dec fine_scroll

.write_d016:
    lda $d016
    and #$f8
    ora fine_scroll
    sta $d016

    rts
```

The crucial point is that coarse reconstruction happens only in the hidden buffer.

---

# 55. Full Color RAM copier skeleton

```asm
copy_color_a:

!for .i,0,999 {
    lda COLOR_A+.i
    sta $d800+.i
}

    rts


copy_color_b:

!for .i,0,999 {
    lda COLOR_B+.i
    sta $d800+.i
}

    rts
```

The monotonically increasing `.i` is not incidental.

It defines the display-deadline order.

---

# 56. Why this is not tearing in the usual sense

With ordinary framebuffer tearing, a display scans a buffer while software replaces arbitrary portions of it, and the boundary between old/new frames becomes visible.

Here software intentionally modifies physical Color RAM while the frame is progressing.

The difference is that the writes are scheduled so that:

```text
CPU modifies future display data
not data whose display deadline has passed
```

The CPU is effectively painting the road immediately ahead of the VIC-II, while maintaining sufficient distance that the VIC never reaches unfinished data.

---

# 57. The beam-racing invariant

A concise invariant is:

```text
For the next cell the VIC will consume,
the corresponding Color RAM write must already be complete.
```

A stronger row-level version is:

```text
for r = 0..24:

Tcopy_complete(r) < Tvic_fetch(r)
```

If that inequality is true for every row, the fact that the complete transfer crosses frame wrap is irrelevant.

---

# 58. Initial lead

The copier does not begin at the same instant that the VIC needs row 0.

It begins in the lower border, creating an initial timing lead.

Then, in the simplified no-sprite model, the copier consumes fewer cycles per logical row than the display takes to progress one character row.

Thus:

```text
initial lead > 0

and

CPU row-transfer rate > VIC row-consumption rate
```

which is exactly the structure needed for a successful race.

---

# 59. Why raster 252 is useful

Raster 252 is in/around the lower-border region for the intended PAL setup, giving the CPU a head start before the next frame's upper display rows.

It is not a magical universal constant.

Changing:

- vertical scroll;
- display mode;
- border opening tricks;
- PAL/NTSC target;
- raster effects;

can change what position is most useful.

Treat it as part of the schedule, not as an intrinsic C64 rule.

---

# 60. Exact validation strategy

For a serious implementation, validation should go beyond hand calculations.

A strong workflow is:

```text
1. Assemble source.
2. Inspect generated machine code.
3. Record exact IRQ entry location.
4. Count cycles to $D018 write.
5. Count each generated Color RAM store.
6. Model VIC badline stalls.
7. Model sprite DMA if enabled.
8. Determine each VIC fetch deadline.
9. Assert every required CPU write precedes it.
10. Validate in VICE trace/raster debugging.
11. Test on PAL hardware.
```

The result can be expressed as a table:

```text
row | CPU completion | VIC deadline | margin
----+----------------+--------------+-------
 0  |       ...      |      ...     | ...
 1  |       ...      |      ...     | ...
...
24  |       ...      |      ...     | ...
```

That is substantially stronger than merely observing that the demo “looks okay”.

---

# 61. Production design hierarchy

Use the cheapest mechanism that satisfies the visual effect.

From cheapest to most general:

```text
constant Color RAM
       |
       v
update individual dirty cells
       |
       v
update one dirty row
       |
       v
update several dirty rows
       |
       v
full 1000-byte shadow transfer
```

Do not spend 8000 cycles every frame if only 40 bytes actually changed.

The full transfer is valuable because it demonstrates the hardest general case.

---

# 62. Practical engineering rules

1. **Double-buffer screen matrices, not physical Color RAM.**
2. Keep a software color shadow for every logical screen that needs independent colors.
3. Modify only the hidden logical buffer.
4. Keep character and color state synchronized.
5. Commit at a controlled raster position.
6. Copy physical Color RAM in upcoming display order.
7. Give earliest deadlines highest priority.
8. Count cycles, not source-code lines.
9. Include badlines and sprite DMA in the real budget.
10. Do not assume PAL timing on NTSC.
11. Avoid screen/program memory overlap.
12. Optimize dirty regions when possible.
13. Validate the assembled machine code, not merely the source.
14. Treat raster effects as real-time scheduling problems.

---

# 63. Final mental model

The whole technique can be reduced to this picture:

```text
                    PREPARE NEXT FRAME

               +-----------------------+
               | hidden SCREEN matrix  |
               +-----------+-----------+
                           |
                           | same logical frame
                           |
               +-----------v-----------+
               | hidden COLOR shadow   |
               +-----------+-----------+
                           |
                           v

                     COMMIT POINT
                     raster IRQ

                           |
              +------------+------------+
              |                         |
              v                         v

        switch $D018             begin $D800 copy
              |                   top -> bottom
              |                         |
              v                         v

      new screen visible       future rows prepared
              |                         |
              +------------+------------+
                           |
                           v

                        VIC-II
                           |
                           v

                  DISPLAY CONSUMPTION
```

The screen switch is instantaneous from the software's point of view.

The Color RAM transition is not.

Instead, it is transformed into a sequence of deadlines that the CPU satisfies ahead of the scanning VIC-II.

That is the essence of the technique.

---

# 64. Short version

If only five facts are remembered, remember these:

```text
1. $D018 can switch Screen RAM.

2. $D018 cannot switch $D800 Color RAM.

3. Keep matching colors in ordinary-RAM shadow buffers.

4. At the swap, copy the new shadow into $D800 in
   top-to-bottom display order.

5. The entire copy need not finish before raster 0;
   every row only needs to finish before the VIC-II
   reaches that row.
```

Or in one sentence:

> **Build the next screen and its colors off-screen, flip the screen at a controlled raster point, then copy the color shadow in earliest-display-deadline order fast enough to remain ahead of the VIC-II beam.**


---

# 65. Revision: make the memory map provably safe

A subtle but important issue in large examples is **assembled code size**.

A pair of fully unrolled color copiers consumes approximately:

```text
1000 cells
× 2 instructions/cell
× 3 bytes/instruction
× 2 source buffers
= 12000 bytes
```

That is before the IRQ, scroll code, tables, strings, and documentation/demo logic.

Therefore this is unsafe:

```text
$0810  program begins
$0C00  second screen
```

because only:

```text
$0C00 - $0810 = $03F0 = 1008 bytes
```

separate them.

The generated copier alone is many times larger than that.

The correct engineering approach is to treat the linker/assembler memory map as part of correctness.

A much safer arrangement is:

```text
$0400-$07FF   SCREEN A

$0800-$0BFF   SCREEN B
$0801...      temporary BASIC loader bytes before startup only

$1000-$1FFF   VIC-visible character-ROM window / leave appropriately arranged

$2000...      machine-code body
              IRQ
              scroll engine
              unrolled copier A
              unrolled copier B
              data/tables

$C000-$C3FF   COLOR SHADOW A
$C400-$C7FF   COLOR SHADOW B

$D800-$DBE7   physical Color RAM
```

The startup sequence is then:

```text
BASIC LOAD
   |
   v
tiny SYS stub around $0801
   |
   v
JMP $2000
   |
   v
machine code is now executing safely elsewhere
   |
   v
initialize/overwrite $0800-$0BFF as Screen B
```

This matters because a demo that is visually correct but silently overwrites its own machine code is not a working example.

---

# 66. Correct startup layout in ACME

A robust ACME layout can be written as:

```asm
; BASIC: 10 SYS8192
* = $0801

!byte $0b,$08,$0a,$00,$9e
!text "8192"
!byte 0,0,0

; Main code is deliberately outside both screen buffers.
* = $2000

start:
    sei
    ; initialization...
```

Now:

```text
SYS 8192 = $2000
```

and once execution has reached `$2000`, Screen B may safely occupy `$0800`.

This is preferable to putting the complete program at `$0810`.

---

# 67. `$D018` values with the safe layout

With VIC bank 0:

```text
Screen A = $0400
Screen B = $0800
character source selection = $1000
```

the intended `$D018` values are:

```text
A: $14
B: $24
```

Breakdown:

```text
$14 = %00010100
       ^^^^
       screen selector = 1 -> $0400

$24 = %00100100
       ^^^^
       screen selector = 2 -> $0800
```

Both retain the same character-data selector.

The exact interpretation of the character source also depends on VIC bank and the special character-ROM visibility behavior, so `$D018` should never be documented independently of `$DD00`.

---

# 68. Character ROM is not ordinary RAM

The C64's built-in character generator deserves explicit treatment.

From the CPU's perspective, the character ROM is not simply normal writable memory at `$1000`.

From the VIC-II's perspective, character ROM can appear in particular VIC address windows depending on the selected VIC bank.

This means three concepts must be kept separate:

```text
CPU address map
VIC-II address map
physical ROM/RAM selected on the bus
```

A statement such as:

```text
charset = $1000
```

is shorthand for the VIC-II's character-data selection, not permission for the CPU to treat that region as an ordinary writable character bitmap.

If custom characters are required, place character bitmap data in VIC-visible RAM and choose the corresponding `$D018` character offset.

---

# 69. `$D016`: fine scroll and column mode are coupled

`$D016` contains more than the three fine-scroll bits.

Important fields include:

```text
bits 0-2   XSCROLL
bit 3      CSEL
bit 4      multicolor mode control
```

For scrolling, edge handling matters.

A 38-column display mode is commonly useful because it creates side-border cover for the pixel motion. If the screen is left in full 40-column presentation while horizontally fine-scrolling, newly exposed edge pixels/cells can become visible in an undesirable way.

Therefore code such as:

```asm
lda $d016
and #%11111000
ora fine
sta $d016
```

preserves the current non-scroll mode bits; it does **not** decide whether 38- or 40-column mode is correct.

The mode should be configured intentionally during initialization.

This is an important distinction between:

```text
logical screen width = 40 cells in memory
```

and:

```text
visible horizontal presentation during fine scrolling
```

which may intentionally use the VIC's reduced-column mode to hide the scrolling edge.

---

# 70. The screen is 1000 useful bytes, but the page is 1024 bytes

A screen matrix occupies:

```text
40 × 25 = 1000 = $03E8 bytes
```

but VIC screen locations are naturally aligned on `$0400` boundaries.

Thus a reserved screen block is normally 1024 bytes:

```text
$0400-$07FF
```

while the displayed matrix uses only:

```text
$0400-$07E7
```

The final 24 bytes:

```text
$07E8-$07FF
```

are outside the 40×25 displayed matrix.

Likewise:

```text
Screen B page: $0800-$0BFF
Displayed B:   $0800-$0BE7
```

This distinction is useful when auditing memory maps.

---

# 71. Color RAM is physically unusual

Color RAM is not just another 1 KiB RAM page.

For normal character-cell color use, each location effectively supplies a 4-bit color value:

```text
0..15
```

So software should think of:

```text
$D800[n] & $0F
```

as the meaningful color.

The shadow buffers should therefore store normalized values:

```asm
lda new_color
and #$0f
sta COLOR_A,x
```

even if a particular code path already guarantees a valid palette index.

Keeping shadows normalized makes debugging and comparison much easier.

---

# 72. Raster IRQ latency: raster compare is not instruction zero

When the VIC raster comparator triggers, the CPU does not magically execute the first instruction of `irq:` at the exact same conceptual instant.

There is interrupt-recognition and entry overhead, and if the KERNAL IRQ entry is used there is additional work before dispatch through `$0314/$0315`.

Therefore:

```text
raster compare = 252
```

does not mean:

```text
first STA $D800 happens at cycle 0 of line 252
```

A cycle-exact proof must derive:

```text
raster compare
   +
CPU interrupt entry
   +
KERNAL dispatch overhead
   +
custom IRQ preamble
   +
screen-swap logic
   +
JSR copier
   =
first actual color write
```

The correct timing origin is the actual first relevant bus access, not merely the programmed raster number.

---

# 73. IRQ jitter

A raster interrupt can be recognized at an instruction boundary.

If arbitrary foreground code is running, the exact instruction being completed when IRQ is asserted can affect entry phase.

For many ordinary effects this small variation is harmless.

For cycle-exact effects it matters.

Classic C64 techniques for eliminating or controlling raster jitter include stabilized raster interrupts, often using a second interrupt or carefully arranged synchronization.

The beam-raced color-copy design does not automatically require a fully stabilized IRQ if it has ample margin, but the uncertainty must be included in the worst-case start-time budget.

---

# 74. Badline condition

A badline is not simply “every eighth line forever”.

Its occurrence depends on VIC state, including display enable and vertical scroll relationship.

At a high level, during the active display region a badline occurs when the raster line's low three bits match the vertical scroll setting in `$D011`, subject to the VIC's display-state conditions.

This is why changing `$D011` vertical scroll can move the badline phase.

For an exact row-deadline table, use the actual configured `$D011` value rather than assuming a generic fixed pattern.

---

# 75. What the VIC fetches on a badline

During a badline the VIC-II fetches character matrix and color information needed for the next character row.

This is precisely why Color RAM has a **deadline**.

The displayed color is not necessarily read continuously from `$D800` for every pixel at arbitrary times. The VIC performs scheduled fetch activity associated with character-row preparation.

Therefore the safe-write question is tied to the relevant VIC fetch window.

This makes the stronger formulation:

```text
Color RAM cell must be updated before the VIC fetch that latches/uses it
```

more accurate than merely saying:

```text
before the pixels become visible
```

The fetch deadline precedes the eventual visible pixels.

---

# 76. BA and AEC: why the CPU loses cycles

Two VIC/CPU bus-control concepts matter:

```text
BA   = Bus Available
AEC  = Address Enable Control
```

The VIC can warn the CPU and then take control of the bus for its own memory accesses.

From the programmer's perspective, the practical consequence is that some CPU memory operations are stretched/stalled around VIC DMA activity.

Thus a cycle table for ordinary 6510 instructions is necessary but not sufficient for raster timing.

You need both:

```text
nominal instruction cycles
+
VIC-induced stalls
```

to derive wall-clock raster position.

---

# 77. Exact copy-body instruction stream

For one cell:

```asm
lda $c000
sta $d800
```

the nominal sequence is eight CPU cycles.

For the next cell:

```asm
lda $c001
sta $d801
```

another eight.

Therefore the nominal store completion pattern advances by eight cycles per cell:

```text
cell 0   store completes around transfer cycle 8
cell 1   store completes around transfer cycle 16
cell 2   store completes around transfer cycle 24
...
cell 39  store completes around transfer cycle 320
```

For row `r`, with zero-based indexing, the nominal end of that row is:

```text
C_nominal(r) = 320 × (r + 1)
```

before VIC stalls and surrounding handler overhead are applied.

That gives a convenient base formula for automated validation.

---

# 78. Formal deadline test

Let:

```text
S = actual cycle/time at which the copy body begins
D(r) = VIC deadline for row r
W(r) = nominal CPU work through row r
V(r) = accumulated VIC stall time before row r completes
```

Then:

```text
Tcopy(r) = S + W(r) + V(r)
```

with:

```text
W(r) = 320 × (r + 1)
```

The correctness requirement is:

```text
S + W(r) + V(r) < D(r)
```

for every:

```text
r = 0..24
```

Define margin:

```text
M(r) = D(r) - Tcopy(r)
```

Then:

```text
M(r) > 0
```

must hold for every row.

The smallest margin is the important result:

```text
Mmin = min(M(0), M(1), ... M(24))
```

A rigorous implementation should report `Mmin`, not just an average throughput number.

---

# 79. Why average throughput can mislead

Suppose average copy throughput is faster than average VIC consumption.

That alone does not prove correctness.

A temporary stall could still cause:

```text
M(7) <= 0
```

even if later rows would otherwise recover.

Real-time systems fail on the missed deadline, not on the average.

Therefore:

```text
average CPU rate > average VIC rate
```

is useful intuition, while:

```text
every individual deadline has positive margin
```

is the actual proof target.

---

# 80. Frame wrap arithmetic

When the transfer crosses raster 311 to raster 0, naïve raster-number subtraction becomes awkward:

```text
311 -> 0
```

For analysis, use a monotonically increasing cycle timeline.

For example:

```text
frame N raster 252  = time origin
frame N raster 311  = later
frame N+1 raster 0  = still later
frame N+1 raster 50 = still later
```

Represent positions as absolute cycles from the IRQ event rather than repeatedly wrapping raster numbers.

This avoids mistakes in deadline calculations.

---

# 81. Double buffering as producer/consumer synchronization

The architecture can be understood as a producer/consumer system.

CPU:

```text
producer of future display state
```

VIC-II:

```text
consumer of display state
```

Screen double buffering gives the producer private workspace.

Color shadowing gives the producer private color workspace.

The raster commit establishes when the produced state becomes eligible for consumption.

The beam race then guarantees that the non-switchable physical resource is populated before each consumer deadline.

This framing is useful because the same principles appear in modern graphics, audio, networking, and real-time systems.

---

# 82. Three different notions of “ready”

It helps to distinguish:

### Logically ready

The hidden screen and shadow contain a complete next state.

```text
SCREEN_B complete
COLOR_B complete
```

### Selected

`$D018` points the VIC at the new screen matrix.

```text
active = B
```

### Physically committed

The required future Color RAM cells have actually reached `$D800`.

These events are not simultaneous.

A robust state machine never confuses them.

---

# 83. Swap-pending is a commit request

`swap_pending` should be understood as:

```text
"The producer has completed the hidden logical frame.
 Commit it at the next legal presentation point."
```

It should not mean:

```text
"Start modifying whatever happens to be visible."
```

That distinction keeps rendering work and presentation timing separated.

---

# 84. Why a raster-controlled commit is useful

Without a controlled commit, `$D018` could change at an arbitrary display position.

That could expose a new screen matrix in the middle of a frame.

A raster IRQ lets software choose a predictable presentation boundary.

The color transfer then begins from a known region of the frame, giving the deadline analysis a stable reference.

---

# 85. Edge hiding during horizontal scroll

When a 40-column character matrix is shifted by pixel increments, the edge of the matrix can reveal unwanted data because fine scrolling exposes pixels at the side.

A classic solution is to use the VIC's reduced-width display mode during scrolling so the border hides the entering/leaving edge.

This is why `$D016` bit 3 must be documented alongside bits 0–2.

The logical row may still contain 40 cells even if fewer are exposed at the exact scrolling boundary.

---

# 86. Coarse-shift direction

For leftward visual motion, the matrix generally shifts content toward lower column indices:

```text
source column 1 -> destination column 0
source column 2 -> destination column 1
...
source column 39 -> destination column 38
new character    -> destination column 39
```

That is why code uses:

```asm
lda ROW+1,x
sta ROW,x
```

with increasing `x`.

Using the opposite copy direction would duplicate/overwrite source values incorrectly for an in-place shift.

---

# 87. In-place copy overlap reasoning

The shift:

```text
src = x+1
dst = x
```

is safe when iterating low-to-high because each destination is behind the source:

```text
read 1 -> write 0
read 2 -> write 1
read 3 -> write 2
```

The next source has not yet been destroyed.

This is the same overlap rule that determines whether a `memmove`-style operation should run forward or backward.

---

# 88. Screen code versus PETSCII

Text stored in source code is not automatically identical to VIC screen codes.

For uppercase letters, a simple demonstration may convert values using logic such as:

```asm
cmp #$41
bcc .not_upper

cmp #$5b
bcs .not_upper

and #$1f
```

This maps uppercase alphabetic values into the corresponding screen-code range for the intended character set.

A complete text engine should define exactly:

```text
source encoding
PETSCII interpretation
screen-code conversion
active character set
```

rather than assuming all byte values are interchangeable.

---

# 89. Color generation

A demonstration can assign colors algorithmically:

```asm
and #$0f
tax
lda palette,x
```

with a 16-entry table.

A production scroller might instead encode color commands in the text stream:

```text
character bytes
control codes
color changes
speed changes
pause commands
effect commands
```

The important architectural rule remains that the resulting color belongs in the hidden buffer's shadow until commit.

---

# 90. Extending the text stream

A richer scroller format could be:

```text
$01-$1A     screen characters
$80-$8F     set color 0-15
$90         pause
$91         speed up
$92         slow down
$93         reverse
$94         trigger effect
$FF         end / loop
```

The parser runs while constructing hidden state, not while the VIC is consuming the visible row.

This preserves deterministic presentation.

---

# 91. A practical cycle-budget worksheet

For each raster-controlled phase, record:

| Component | Worst-case cycles |
|---|---:|
| IRQ recognition/entry | measure/model |
| KERNAL dispatch | measure/model |
| `$D019` acknowledge | known instruction cost |
| frame counter logic | count |
| fine-scroll logic | count |
| coarse shift | worst-case count |
| text parser | worst-case count |
| `$D018` swap | count |
| `JSR` copier | 6 nominal |
| color body | 8000 nominal + stalls |
| `RTS` | 6 nominal |
| IRQ exit | measure/model |

The table should contain **worst case**, not typical case.

---

# 92. Coarse-frame versus fine-frame budget

There are at least two execution classes.

### Fine-only update

```text
update fine value
write $D016
no matrix shift
no new character
```

### Coarse update

```text
shift hidden screen row
shift hidden color row
parse next character
calculate/store new color
set swap_pending
write $D016
```

The coarse path is more expensive.

A timing proof must use the path that actually precedes a swap, not the cheapest frame.

---

# 93. Debugging with border colors

A classic C64 timing-debug technique is to write colors to `$D020` around code regions.

Example:

```asm
lda #2
sta $d020

jsr copy_color_a

lda #6
sta $d020
```

On a real display or emulator raster view, the border-color region visually reveals how long the code occupies the frame.

For more phases:

```text
red     = IRQ entry
yellow  = coarse update
green   = $D018 commit
cyan    = Color RAM copy
blue    = idle
```

This converts timing into a visible oscilloscope-like display.

The instrumentation itself costs cycles, so remove it or include its cost in final measurements.

---

# 94. Debugging the active buffer

A useful debug overlay can show:

```text
ACTIVE: A
HIDDEN: B
FINE:   5
SWAP:   0
RASTER: 252
```

When the coarse boundary occurs:

```text
ACTIVE: A
HIDDEN: B
FINE:   7
SWAP:   1
```

After commit:

```text
ACTIVE: B
HIDDEN: A
FINE:   7
SWAP:   0
```

This makes state-machine errors immediately visible.

---

# 95. Guard bytes

During development, reserve sentinel values around buffers:

```text
before SCREEN A
after SCREEN A
before COLOR_A
after COLOR_A
```

Periodically verify that the sentinels remain unchanged.

This catches off-by-one errors such as accidentally copying 1024 bytes into the 1000-byte physical Color RAM region.

---

# 96. Why exactly 1000 Color RAM bytes

The displayed matrix is:

```text
40 × 25 = 1000
```

Therefore the useful physical range is:

```text
$D800
through
$D800 + 999
=
$DBE7
```

The shadow may occupy a convenient 1024-byte page-aligned block, but the full-screen copier should stop at 999 unless the extra addresses are deliberately understood and handled.

---

# 97. Compile-time assertions

A polished ACME project should use build-time checks where practical.

Conceptually assert:

```text
SCREEN_A aligned to $400
SCREEN_B aligned to $400
SCREEN_A != SCREEN_B
code does not overlap screens
color shadows do not overlap code
color shadows contain >=1000 bytes
```

Even if some checks are performed by a separate build script, memory-map validation should be automated rather than left to visual inspection.

---

# 98. Generate a symbol/map file

A release build should produce assembler symbols.

For example, use ACME's symbol/report facilities as appropriate for the installed version and inspect:

```text
start
irq
copy_color_a
copy_color_b
message
end_of_program
```

Then verify:

```text
end_of_program < next reserved region
```

This is particularly important with generated code because source length does not visually reveal binary size.

---

# 99. Validate the actual PRG

A robust build pipeline can parse the PRG's two-byte load address and payload size.

If:

```text
load = $2000
payload = N bytes
```

then:

```text
end = $2000 + N - 1
```

Check this against every reserved range.

The build should fail on overlap.

This turns a memory-layout assumption into an enforceable invariant.

---

# 100. Cycle-exact trace validation

The strongest demonstration would generate or collect a trace containing:

```text
absolute cycle
raster
cycle-in-line
PC
opcode
memory address
memory value
```

For Color RAM stores, derive:

```text
row = (address - $D800) / 40
column = (address - $D800) % 40
```

Then compare each store against the modeled VIC deadline for that cell/row.

A validator can report:

```text
ROW  CPU_COMPLETE  VIC_DEADLINE  MARGIN  RESULT
 0       ...           ...        ...    PASS
 1       ...           ...        ...    PASS
...
24       ...           ...        ...    PASS
```

This is the ideal bridge between source-code reasoning and empirical emulator validation.

---

# 101. Why emulator validation is not the whole story

An emulator trace is extremely useful, but final confidence benefits from hardware testing because the project is explicitly about bus timing.

A good hierarchy is:

```text
static cycle analysis
        +
assembled-binary audit
        +
VICE trace
        +
real PAL C64 observation
```

Each layer catches different classes of mistakes.

---

# 102. Recommended final demo architecture

For a polished teaching demo:

```text
BOOT
 |
 +-- initialize VIC bank
 +-- initialize both screens
 +-- initialize both color shadows
 +-- initialize physical Color RAM
 +-- install IRQ
 |
 v
DEMO LOOP / IRQ SYSTEM
 |
 +-- animated explanatory dashboard
 |
 +-- live fine-scroll value
 |
 +-- active/hidden buffer indicator
 |
 +-- raster indicator
 |
 +-- colored scrolling text
 |
 +-- optional border timing bars
 |
 +-- beam-raced full Color RAM transfer
 |
 v
VISUAL PROOF
```

The demo should not merely tell the user that the algorithm works; it should expose enough state that the user can see what the engine is doing.

---

# 103. Suggested on-screen tutorial pages

A complete educational demo can cycle through pages.

### Page 1 — The hardware problem

```text
SCREEN RAM CAN SWITCH
COLOR RAM CANNOT
```

### Page 2 — The shadows

```text
A: $0400 + $C000
B: $0800 + $C400
```

### Page 3 — Fine scroll

Display the live `$D016 & 7` value.

### Page 4 — Coarse commit

Highlight which buffer is active.

### Page 5 — Beam race

Use border colors to visualize the copy duration.

### Page 6 — Cycle math

```text
40 * 8 = 320
1000 * 8 = 8000
```

### Page 7 — Deadline rule

```text
WRITE BEFORE VIC FETCH
```

This transforms the project from a code sample into a self-explaining C64 technical demo.

---

# 104. Final corrected architecture

The strongest form of the design is:

```text
                          CPU PRODUCER

          +--------------------------------------+
          |                                      |
          v                                      v
 +------------------+                   +------------------+
 | SCREEN A $0400   |                   | SCREEN B $0800   |
 +--------+---------+                   +--------+---------+
          |                                      |
          v                                      v
 +------------------+                   +------------------+
 | SHADOW A $C000   |                   | SHADOW B $C400   |
 +--------+---------+                   +--------+---------+
          \                                      /
           \                                    /
            +----------------+-----------------+
                             |
                       raster commit
                             |
                   +---------+---------+
                   |                   |
                   v                   v
                $D018              CPU transfer
                   |                   |
                   v                   v
             selected screen     $D800-$DBE7
                   |                   |
                   +---------+---------+
                             |
                             v
                           VIC-II
                             |
                             v
                       DISPLAY CONSUMER


MAIN MACHINE CODE: $2000...
```

This layout separates:

```text
display buffers
loader
large generated machine code
software color shadows
physical Color RAM
```

and makes the memory architecture auditable.

---

# 105. Deepest takeaway

The interesting part of this technique is not the 6502 copy itself.

It is the conversion of a hardware limitation:

```text
"There is only one Color RAM."
```

into a real-time scheduling problem:

```text
"I do not need a second physical Color RAM if I can guarantee
that every future color value reaches the single Color RAM
before the VIC-II consumes it."
```

That is the deeper engineering principle.

The software creates temporal separation where the hardware does not provide spatial separation.

Screen RAM gets **spatial double buffering**:

```text
A memory region
B memory region
```

Color RAM gets **temporal buffering**:

```text
future state in ordinary RAM
+
deadline-scheduled commit to one physical resource
```

The two techniques combine into one visually coherent frame.

That is why the method works, and it is the key idea the demo should teach.


---

# 106. Reference implementation invariants

A finished implementation should make these invariants explicit in source comments and, where possible, build-time checks.

```text
I1  Screen A and Screen B never overlap executable code.

I2  COLOR_A and COLOR_B are ordinary RAM and each provide
    at least 1000 usable bytes.

I3  The visible screen is never destructively coarse-shifted.

I4  A screen buffer and its color shadow represent the same
    logical generation.

I5  $D018 changes only after the hidden logical generation
    is complete.

I6  Physical Color RAM is copied in monotonically increasing
    display/deadline order.

I7  Every required $D800-$DBE7 write precedes the VIC-II
    fetch that consumes that value.

I8  The worst-case timing proof includes IRQ latency,
    VIC-II stalls, sprite DMA if enabled, and all other work
    executed before/during the transfer.

I9  PAL constants are not silently reused for NTSC.

I10 Debug instrumentation is either included in the cycle
    budget or removed from the release build.
```

These are more useful than a statement such as “works on my emulator,” because they define what must remain true after the demo is extended.

---

# 107. Recommended final source memory map

For the large, fully unrolled teaching build:

```text
$0000-$00FF   zero page / system
$0100-$01FF   hardware stack
$0200-$03FF   system / vectors / workspace

$0400-$07FF   SCREEN A

$0800-$0BFF   SCREEN B
              ($0801 loader exists only as startup content)

$0C00-$1FFF   reserve according to charset/system needs

$2000...      MAIN CODE
              initialization
              raster IRQ
              scroll engine
              text parser
              generated color copier A
              generated color copier B
              tables / messages

$C000-$C3FF   COLOR SHADOW A
$C400-$C7FF   COLOR SHADOW B

$D000-$DFFF   I/O when mapped
$D800-$DBE7   PHYSICAL COLOR RAM
```

The exact end address of main code must be obtained from the assembled output rather than guessed.

---

# 108. Recommended build-time memory audit

A polished build should fail if any interval overlaps.

Treat every allocation as:

```text
[start, end]
```

and test every relevant pair.

For two regions A and B, overlap exists when:

```text
A.start <= B.end
AND
B.start <= A.end
```

The build should specifically test:

```text
CODE vs SCREEN_A
CODE vs SCREEN_B
CODE vs COLOR_A
CODE vs COLOR_B

SCREEN_A vs SCREEN_B

COLOR_A vs COLOR_B
```

It should also verify:

```text
SCREEN_A % $400 == 0
SCREEN_B % $400 == 0

size(SCREEN_A) >= 1000
size(SCREEN_B) >= 1000
size(COLOR_A)  >= 1000
size(COLOR_B)  >= 1000
```

This converts the memory map from documentation into an enforceable contract.

---

# 109. Recommended timing audit output

The ideal validator emits a row table resembling:

```text
+-----+---------------+---------------+---------+--------+
| ROW | CPU COMPLETE  | VIC DEADLINE  | MARGIN  | RESULT |
+-----+---------------+---------------+---------+--------+
|  0  |      ...      |      ...      |   ...   | PASS   |
|  1  |      ...      |      ...      |   ...   | PASS   |
|  2  |      ...      |      ...      |   ...   | PASS   |
| ... |      ...      |      ...      |   ...   | ...    |
| 24  |      ...      |      ...      |   ...   | PASS   |
+-----+---------------+---------------+---------+--------+

minimum margin = ...
```

A release criterion can then be stated objectively:

```text
PASS iff minimum margin > required safety margin
```

The required margin may intentionally be greater than zero to absorb modeled uncertainty.

---

# 110. Instruction-level color-copy ledger

Ignoring VIC stalls, cell `n` uses:

```text
instruction 1:
    LDA COLOR+n
    opcode bytes = 3
    nominal time = 4 cycles

instruction 2:
    STA $D800+n
    opcode bytes = 3
    nominal time = 4 cycles
```

Thus:

```text
code offset for cell n = n × 6 bytes

nominal start cycle:
    8n

nominal completion after store:
    8(n+1)
```

For row `r` and column `c`:

```text
n = 40r + c
```

therefore:

```text
nominal completion(r,c)
    = 8 × (40r + c + 1)
```

and row completion is:

```text
nominal row completion(r)
    = 8 × 40 × (r+1)
    = 320(r+1)
```

This formula is useful for a trace checker because the expected nominal ordering can be generated without parsing source text.

---

# 111. Cell-level deadline formulation

The row-level proof is convenient, but the strongest formulation is cell-level.

For cell:

```text
(r,c)
```

define:

```text
Tw(r,c) = actual completion time of CPU store
Tf(r,c) = VIC-II fetch deadline
```

Then require:

```text
Tw(r,c) < Tf(r,c)
```

for all:

```text
0 <= r < 25
0 <= c < 40
```

The row-level test is sufficient only if the row deadline chosen is conservative for every cell in that row.

A validator can therefore operate at either:

```text
1000 cell deadlines
```

or:

```text
25 conservative row deadlines
```

depending on the fidelity of the VIC model.

---

# 112. Generation numbers prevent stale-buffer bugs

A production implementation can associate a generation counter with each logical buffer:

```text
screen_generation_A
color_generation_A

screen_generation_B
color_generation_B
```

Before commit:

```text
screen_generation_hidden
==
color_generation_hidden
```

must hold.

This detects a class of bugs where the hidden screen has been rebuilt but its color shadow belongs to an older logical frame.

Conceptually:

```text
A = generation 17
B = generation 18

commit B only if:

SCREEN_B.gen == COLOR_B.gen == 18
```

The idea costs little in a debug build and makes buffer ownership auditable.

---

# 113. Commit protocol

A disciplined commit can be described as:

```text
PREPARE
    write hidden screen
    write hidden color shadow

PUBLISH READY
    hidden_ready = true

RASTER COMMIT
    assert hidden_ready
    select hidden screen
    begin corresponding color transfer
    active = hidden
    hidden = old active
    hidden_ready = false
```

This avoids letting rendering code and presentation code independently manipulate `active_buffer`.

One piece of code owns the commit.

---

# 114. Do not swap twice

A common state-machine error is allowing multiple code paths to toggle:

```asm
active_buffer
```

If both coarse-scroll code and raster code perform the toggle, the logical and hardware states can diverge.

Prefer:

```text
producer:
    requests swap

presenter:
    performs swap
```

In this design the raster IRQ is the presenter.

---

# 115. Hardware state versus software state

Debug builds should periodically verify that software's `active_buffer` agrees with the `$D018` value it expects.

Conceptually:

```text
if active == A:
    expected D018 screen bits = A

if active == B:
    expected D018 screen bits = B
```

A mismatch means the software model and VIC hardware model have split.

This is particularly useful when later effects also modify `$D018`.

---

# 116. Self-modifying single-copier alternative

Two unrolled copiers are easy to audit but expensive.

An alternative is one unrolled destination stream whose source operands are patched.

Conceptually:

```asm
copy:
s0:
    lda $c000
    sta $d800
s1:
    lda $c001
    sta $d801
...
```

To switch from shadow A to B, patch the high-byte operands of the `LDA`s:

```text
$C0/$C1/$C2/$C3
        ->
$C4/$C5/$C6/$C7
```

depending on the source address.

This reduces duplicated copy code but adds:

```text
patch generation
patch-time cost
self-modifying-code complexity
more difficult auditing
```

For teaching, two explicit copiers remain clearer.

---

# 117. Page-specialized copier alternative

Because each shadow occupies four pages, a compromise is to generate four specialized 256-byte sections and choose source pages.

This can reduce some duplication while keeping destinations explicit.

Any such optimization should be evaluated against three resources:

```text
CPU time
code size
complexity / proof burden
```

On a C64, the best solution is often the one that optimizes the resource the current effect is actually short of.

---

# 118. Dirty-cell queue alternative

If only sparse colors change, maintain a queue:

```text
(address offset, color)
```

Example:

```text
(17,  RED)
(18,  RED)
(481, CYAN)
```

At the safe raster phase:

```asm
; conceptual
load offset
load color
store $d800+offset
```

This can be vastly cheaper than 1000 cells, but indirect/indexed dispatch introduces its own timing and ordering considerations.

The queue should be sorted by display deadline if it can contain cells from multiple rows.

---

# 119. Earliest-deadline-first generalization

The top-to-bottom full copier is a special case of a more general scheduler.

For any set of pending display writes:

```text
sort by VIC consumption deadline
execute earliest deadline first
```

For a normal character matrix, memory order and deadline order align sufficiently well that monotonically increasing Color RAM addresses are natural.

For more complex raster effects, deadline order may not equal address order.

This is the deeper generalization of “race the beam”.

---

# 120. Why this resembles scanline rendering

Modern systems often prepare complete frames and present them atomically.

The C64 technique described here is closer to a scanline/deadline renderer:

```text
prepare enough future state
before the display scanner consumes it
```

The hardware limitation forces software to reason about **when** data is consumed, not just **where** it is stored.

That temporal reasoning is one of the defining skills of advanced VIC-II programming.

---

# 121. Final audit checklist

Before calling the implementation complete, verify all of the following:

```text
MEMORY
[ ] Main code does not overlap either screen.
[ ] Main code does not overlap color shadows.
[ ] Screen pages are correctly aligned.
[ ] Screen B is not overwritten while executing loader code.
[ ] The assembled end address is checked.
[ ] Only 1000 useful Color RAM cells are assumed.

VIC CONFIGURATION
[ ] Correct VIC bank selected through $DD00.
[ ] $D018 values match the actual screen addresses.
[ ] Character source is understood for the selected bank.
[ ] $D016 CSEL/fine-scroll policy is intentional.
[ ] $D011 vertical-scroll value is known.

BUFFERING
[ ] Visible coarse buffer is never reconstructed in place.
[ ] Hidden screen and hidden color shadow are synchronized.
[ ] Only raster/presentation code performs the commit.
[ ] Software active-buffer state matches hardware $D018.

IRQ
[ ] Raster high bit in $D011 is correct.
[ ] $D012 contains the intended line.
[ ] $D019 is acknowledged correctly.
[ ] $D01A enables the intended VIC source.
[ ] CIA IRQ policy is intentional.
[ ] IRQ entry/exit mechanism matches KERNAL/raw-vector design.

TIMING
[ ] Copy body nominal cost is known.
[ ] IRQ preamble cost is included.
[ ] Coarse-update worst case is included.
[ ] Badline stalls are included.
[ ] Sprite DMA is included if sprites are active.
[ ] Music/effects are included.
[ ] IRQ jitter is bounded or stabilized.
[ ] Every row/cell has positive deadline margin.
[ ] PAL/NTSC assumptions are explicit.

VALIDATION
[ ] Assembler symbols/map inspected.
[ ] PRG memory range audited.
[ ] Border timing instrumentation tested.
[ ] VICE trace checked.
[ ] Real PAL hardware tested when available.
```

---

# 122. Final reference equations

### Geometry

```text
columns = 40
rows    = 25
cells   = 40 × 25 = 1000
```

### Nominal copier

```text
LDA abs = 4 cycles
STA abs = 4 cycles

cell = 8 cycles
row  = 40 × 8 = 320 cycles
full = 1000 × 8 = 8000 cycles
```

### Code size

```text
LDA abs = 3 bytes
STA abs = 3 bytes

cell code = 6 bytes
one full copier = 6000 bytes + RTS
two full copiers ≈ 12002 bytes
```

### PAL first-order conversion

```text
63 cycles/raster line

8000 / 63 ≈ 126.98 raster lines
```

### Nominal row completion

```text
W(r) = 320(r+1)
```

### Actual completion model

```text
Tcopy(r) = S + W(r) + V(r)
```

where:

```text
S    = actual transfer start time
V(r) = accumulated VIC/other stalls through row r
```

### Margin

```text
M(r) = D(r) - Tcopy(r)
```

Correctness:

```text
M(r) > 0
```

for every required row.

Strongest result:

```text
Mmin = min_r M(r) > 0
```

---

# 123. Final conclusion

The C64 does not provide two switchable Color RAM pages.

The solution is not to pretend that it does.

Instead:

```text
SCREEN:
    spatial buffering
    A / B memory pages

COLOR:
    software shadow buffering
    A / B ordinary-RAM images

PRESENTATION:
    raster-controlled screen selection

PHYSICAL COLOR COMMIT:
    deadline-ordered writes into the one $D800 page
```

The complete technique is therefore a combination of:

```text
double buffering
+
shadow state
+
raster synchronization
+
deadline scheduling
+
cycle accounting
```

The essential proof is not:

```text
"The whole copy finishes in the border."
```

It does not.

Nor is it merely:

```text
"The CPU is faster on average."
```

The actual proof obligation is:

```text
For every color value the VIC-II will consume,
the corresponding CPU write has completed before
the VIC-II fetch deadline for that value.
```

Once that is understood, the apparent limitation of a single physical Color RAM becomes manageable: the program substitutes **time** for the missing second hardware buffer.

That is the core of beam-raced Color RAM on the Commodore 64.
