# HF32 — a 32-bit Forth CPU with floating point for the Arty Z7-20

HF32 is a dual-stack machine in the PL whose instructions are Forth primitives, with 32-bit
cells, a separate float stack and an IEEE single-precision floating-point unit. A Forth
kernel, cross-compiled from `forth/kernel.fs` by `tools/hf32.py`, gives an interactive
Forth over the USB UART. The Cortex-A9 (`sw/src`) loads the kernel image and moves
characters between the PS UART and FIFOs in the PL; it also answers the app's service
requests ([Protocol.md](Protocol.md)) and, for the HASE laboratory, MCNF requests itself.

It is the 32-bit successor of the 16-bit HF16 in ZYNQ-FORTH and keeps everything around
the CPU: the same board I/O, shield pins, PWM, XADC and pattern generator, the same ARM
program structure and AXI registers, and the same Forth word names, so sources written for
the 16-bit Forth run here.

```
 ZYNQ-FPGA-FORTH32 app ── USB-UART ── PS UART0 ── ARM program ──── AXI-Lite ── forth_system (PL)
 (or PuTTY, or a HASE host)
                                                                    ├─ HF32 CPU (rtl/hf32_cpu.v)
                                                                    │   └─ multiply + FPU (rtl/hf32_mcu.v)
                                                                    ├─ 128 KB RAM (BRAM)
                                                                    ├─ RX/TX FIFOs
                                                                    ├─ LEDs, RGB, buttons, switches, ms and µs timers
                                                                    ├─ shield pins: 36 GPIO
                                                                    ├─ PWM: 4 channels on any shield pin (rtl/pwm.v)
                                                                    ├─ XADC scanner: A0..A11, temperature, supplies
                                                                    │   (rtl/xadc_scan.v)
                                                                    ├─ pattern generator: 16 channels on the Pmods
                                                                    │   (rtl/patgen.v, 200 MHz)
                                                                    ├─ 2 UARTs, 2 SPI masters, 1 I2C master on any pin
                                                                    │   (rtl/uart.v, spi.v, i2c.v)
                                                                    ├─ DDS on any pin (rtl/dds.v, 200 MHz)
                                                                    └─ tone generator on the audio jack (rtl/tone.v)
```

## Machine model

- 32-bit cells. Byte addresses; cells are little-endian and must be 4-aligned.
- 128 KB RAM at 0x00000–0x1FFFF holds code and data (code can be compiled at run time).
  Addresses with bit 20 set (from 0x100000) are I/O; below that, the RAM repeats every 128 KB.
- Instructions are 32 bits, one cell each; the PC is a cell (word) address of 15 bits.
- Data stack: T register plus 32 entries (N is the top entry). Return stack: 32 entries.
  Float stack: FT register plus 32 entries (FN is the top entry).
- The return stack holds **byte** addresses (`pc = R >> 2`).
- One instruction per clock at 50 MHz, except the multiply and the floating-point operations,
  which run in the multi-cycle unit while the CPU waits (table below).

## Instruction encoding

| Bits 31..29 | Meaning |
|---|---|
| `1xx` | literal: push `insn[30:0]` (0..2147483647); a negative number is a literal and `invert` |
| `000` | jump to cell address `insn[14:0]` |
| `001` | 0branch: pop T, jump to `insn[14:0]` if it was zero |
| `010` | call `insn[14:0]`, pushing the byte address of the next instruction on R |
| `011` | ALU instruction, fields below |

ALU instruction fields:

| Bits | Field | Effect |
|---|---|---|
| 28 | R2P | return: `pc = R >> 2` (combine with rd = -1) |
| 27..22 | op | new T, table below |
| 21 | T2N | write old T into N (used by SWAP; implied on a data push) |
| 20 | N2M | store N at address T (RAM or I/O) |
| 19 | BYTE | the store writes one byte instead of a cell |
| 18..17 | rd | return stack delta: 00 = 0, 01 = +1 (pushes T), 11 = -1, 10 = -2 |
| 16..15 | dd | data stack delta: 00 = 0, 01 = +1 (pushes T into N), 11 = -1, 10 = -2 |
| 14 | F2M | store FT at address T (F!) |
| 13 | FT2FN | write old FT into FN (used by FSWAP; implied on a float push) |
| 12..11 | fd | float stack delta, coded like dd |
| 10..6 | fop | new FT, table below |
| 5..0 | – | 0 |

New T (`op`):

| op | new T | op | new T |
|---|---|---|---|
| 0 | T | 17 | T + 1 |
| 1 | N | 18 | T − 1 |
| 2 | N + T | 19 | T = 0 (−1/0) |
| 3 | N and T | 20 | T < 0 (−1/0) |
| 4 | N or T | 21 | N × T, low cell (multi-cycle) |
| 5 | N xor T | 22 | T >> 1 arithmetic |
| 6 | not T | 23 | −T |
| 7 | N = T (−1/0) | 24 | N × T unsigned, high cell (multi-cycle, for `um*`) |
| 8 | N < T signed | 25 | N >> T arithmetic |
| 9 | N < T unsigned | 26 | FT's bits |
| 10 | N >> T logical | 27 | FN < FT (−1/0) |
| 11 | N << T | 28 | FN = FT (−1/0) |
| 12 | R | 29 | FT < 0 (−1/0) |
| 13 | cell at [T] | 30 | FT = 0 (−1/0) |
| 14 | byte at [T] | 31 | FT to integer, toward zero (multi-cycle) |
| 15 | data stack depth | 32 | float stack depth |
| 16 | N − T | | |

New FT (`fop`):

| fop | new FT | fop | new FT |
|---|---|---|---|
| 0 | FT | 7 | FN / FT (multi-cycle) |
| 1 | FN | 8 | √FT (multi-cycle) |
| 2 | T's bits | 9 | T converted to float (multi-cycle) |
| 3 | cell at [T] | 10 | −FT |
| 4 | FN + FT (multi-cycle) | 11 | \|FT\| |
| 5 | FN − FT (multi-cycle) | 12 | FT truncated toward zero (multi-cycle) |
| 6 | FN × FT (multi-cycle) | | |

One instruction may change all three stacks: `s>f` is `T = N, dd = -1, FT = float(T), fd = +1`,
`f<` is `T = FN < FT, dd = +1, FT = FN, fd = -1` followed by an `fdrop`. Shifts by 32 or more
give 0 (the arithmetic shift: all sign bits). Compare ops treat zero and subnormals as +0
and are false when either operand is a NaN.

**Store hazard:** a store cycle uses the RAM data port for the write, so the instruction
directly after a store must not read memory (ops 13/14, fop 3). The compilers always follow
a store with an instruction that does not: `!` and `c!` end with a plain DROP, `f!` with a
NOP.

## Floating point

`rtl/hf32_mcu.v` holds the 32 × 32 multiplier and the floating-point unit. The CPU starts an
operation when it decodes one, waits (the same freeze as an ARM pause: no register changes,
the code and data ports re-read the current instruction and [T]) until the result is valid,
and takes it when it commits. An ARM pause during the wait leaves the result waiting.

| Operation | Cycles at 50 MHz |
|---|---|
| `*`, `(um*)` | 3 |
| `f+`, `f-` | 6 |
| `f*` | 5 |
| `f/`, `fsqrt` | 31 (one quotient or root bit per clock) |
| `s>f` | 4 |
| `f>s`, `ftrunc` | 2 |

Semantics (IEEE 754 single precision):

- Round to nearest, ties to even, for every operation.
- Subnormal inputs count as zero; a result whose magnitude is below 2^-126 after rounding
  becomes zero with its sign (flush to zero). Overflow gives ±infinity.
- Every NaN result is the quiet NaN 0x7FC00000; 0/0, ∞−∞, 0×∞ and √(negative) give it.
  x/0 is ±∞, −0 + −0 = −0, x − x = +0, √−0 = −0.
- `f>s` truncates toward zero and saturates at −2147483648 and 2147483647; NaN gives 0.

The value being normalized is `m × 2^(e − 127) / 2^48` with a 50-bit `m`; bit 0 collects
the bits shifted out (sticky). Addition aligns the smaller operand, adds or subtracts and
normalizes with a leading-zero count and one shift; multiplication uses a 24 × 24 product;
division and square root are restoring and take one bit per clock (27 bits, the last
remainder's non-zero test is the sticky bit). `tools/hf32.py` contains the same algorithms
step by step; `tools/test_fpu.py` checks that model against IEEE results (Python's doubles
rounded to single, which is exact for these operations) and writes vectors from which
`sim/tb_mcu.v` checks the RTL bit for bit (`sim/run_mcu.ps1`).

**Conversions.** The kernel's `>float` (also used for float literals typed at the
interpreter) and `represent` (behind `f.`, `fs.`, `fe.`) scale by exact powers of ten and
carry the rounding error of each product with Dekker's exact product (a 12-bit split), so the
result is rounded once: `>float` is correctly rounded and the digits `represent` prints are
the correctly rounded decimal value of the float, across the normal range.
`tools/test_float_words.py` checks both, and the elementary functions, in the simulator.

## I/O map

Register k is at `0x100000 + 4k`; only address bits 9..2 are decoded, so the map repeats
every 1 KB above 0x100000. Registers are 32 bits wide; most use only their low bits.

| Address | Read | Write |
|---|---|---|
| 0x100000 | next received byte (pops the RX FIFO) | send byte (pushes the TX FIFO) |
| 0x100004 | bit 0 = RX byte available, bit 1 = TX FIFO has space | – |
| 0x100008 | LD0..LD3 | LD0..LD3 |
| 0x10000C | bits 3..0 = BTN3..BTN0 (debounced), bits 5..4 = SW1..SW0 | – |
| 0x100010 | RGB LEDs: bits 2..0 = LD4 B,G,R; bits 5..3 = LD5 B,G,R | same |
| 0x100014 | millisecond counter (32 bits, free running) | – |
| 0x100018 | microsecond counter (32 bits, free running) | – |
| 0x100020 + 16p | port p (0..2) pin levels, synchronized; pin n = bit n%16 of port n/16 | – |
| 0x100024 + 16p | port p output latch | same |
| 0x100028 + 16p | port p direction, 1 = output (all 0 after reset) | same |
| 0x100080 + 4r | XADC status register r (0..31), result in bits 15..4 | – |
| 0x100100 + 16c | PWM channel c (0..3): period, low 16 bits | same (held until the high half is written) |
| 0x100104 + 16c | PWM channel c: period, high 16 bits (clocks per cycle, at least 2) | same (takes both halves) |
| 0x100108 + 16c | PWM channel c: duty in 1/65536 of the period, 0xFFFF = always high | same |
| 0x10010C + 16c | PWM channel c: bit 15 on, bits 5..0 pin number | same |
| 0x100180..0x1001A8 | pattern generator registers 0..10 at 0x100180 + 4r, see [PatternGenerator.md](PatternGenerator.md#forth-interface) | |
| 0x100200 + 16u | UART u (0, 1): +0 data (read pops a received byte, bit 8 = one was there; write sends), +4 status (bit 0 received, 1 room to send, 2 idle, 3 lost byte, 4 framing error; a write clears 3 and 4), +8 clocks per bit, +12 bit 15 on, bits 13..8 RX pin, 5..0 TX pin | |
| 0x100220 + 32s | SPI master s (0, 1): +0 data (write starts a transfer, read gives the last answer), +4 bit 0 busy, +8 control (bit 0 CPHA, 1 CPOL, 2 LSB first, 5..4 width 8/16/32), +12 clocks per half SCK period (at least 2), +16 bit 31 on, bits 21..16 MISO, 13..8 MOSI, 5..0 SCK pin | |
| 0x100260 | I2C master: +0 command (write: bits 10..8 1 start, 2 stop, 3 write byte, 4 read + ACK, 5 read + NACK, bits 7..0 the byte; read: bit 31 busy, 9 timeout, 8 ACK received, 7..0 byte read), +4 clocks per quarter SCL period, +8 bit 15 on, bits 13..8 SDA, 5..0 SCL pin | |
| 0x100280 | DDS: +0 phase step per 200 MHz tick, +4 bit 0 on, bits 13..8 pin (read: bit 31 not yet taken over) | |
| 0x1002A0 | tone generator: +0 phase step per 50 MHz clock, +4 bit 0 on, bits 2..1 waveform (sine, square, triangle, sawtooth), +8 volume 0..65535, +12 bit 0 sounding, bits 31..16 current volume | |

The PWM and pattern generator modules are those of the 16-bit design; each 16-bit register
of theirs is one cell here (its low half).

### Shield pins

| Pin numbers | Port bits | Shield | FPGA port | Pull-up |
|---|---|---|---|---|
| 0..13 | port 0 bits 0..13 | IO0..IO13 | `ck_io[13:0]` | yes |
| 14..19 | port 0 bits 14..15, port 1 bits 0..3 | A0..A5 as digital pins | `ck_a[5:0]` | no (shares the analog divider node) |
| 26..41 | port 1 bits 10..15, port 2 bits 0..9 | IO26..IO41 | `ck_ioh[15:0]` | yes |
| 42..47 | port 2 bits 10..15 | SCL, SDA, the SPI header's MISO, MOSI, SCK, SS | `ck_x[5:0]` | SCL/SDA: resistors on the board; the header: yes |

Pins 20..25 do not exist: they read 0 and ignore writes. The Forth word `pin` rejects them.

### Pin router

Every output of a peripheral names its pin (PWM channels 0..3, UART TX 0/1, SPI SCK and
MOSI 0/1, the DDS: push-pull; I2C SCL and SDA: open drain, driven low or left to the
pull-up). An enabled peripheral owns that pin: the pin's latch and direction no longer
count, it is driven with the peripheral's level, and several owners of one pin would be
ORed (the Forth words refuse that: `claim`, section 5.16 of the manual). Inputs (UART RX,
SPI MISO, I2C SCL/SDA) read the named pin through its two-flop synchronizer. AXI register
0x38 lets the ARM read the table: a write selects entry 0..13 (UART0 TX/RX, UART1 TX/RX,
SPI0 SCK/MOSI/MISO, SPI1 SCK/MOSI/MISO, I2C SCL/SDA, DDS, tone), a read gives bits 19..16
the selection, bit 15 on, bits 5..0 the pin (63 for the tone: the audio jack).

### Serial interfaces, DDS, tone

- **UART** (`rtl/uart.v`): 8N1, clocks-per-bit divider (8 or more), 64-byte FIFOs; the
  receiver samples each bit in its middle after the start bit's falling edge.
- **SPI** (`rtl/spi.v`): SCK = 50 MHz / (2 × half), half at least 2. MISO comes through the
  synchronizer and is taken at the end of each bit, just before the next change edge, which
  in every mode is after the slave set it and leaves 2 × half − 2 clocks for the slave.
- **I2C** (`rtl/i2c.v`): quarter-period steps; a bit is 2 quarters low, 2 high, sampled a
  quarter after SCL rose. Waits while a slave holds SCL low, for at most 2^20 clocks.
- **DDS** (`rtl/dds.v`): a 32-bit phase accumulator on the 200 MHz clock, its top bit the
  output; settings cross with a toggle handshake (resent if changed meanwhile).
- **Tone** (`rtl/tone.v`): phase accumulator at 50 MHz, quarter-wave sine table (256
  entries, `tools/make_sine.py`), square, triangle, sawtooth; × volume after a linear ramp
  (full scale in 2 ms); first-order delta-sigma modulator on AUD_PWM into the board's
  4th-order low-pass filter (about 17 kHz); AUD_SD on while there is sound.

`sim/tb_periph.v` (`sim/run_periph.ps1`) tests them alone: UART at 1 MBd with all byte
values and an overrun, SPI loopback in every mode, width and bit order at 12.5 MHz, I2C
against a 24Cxx-style slave model that stretches SCL, the DDS frequency and the tone
modulator; `sim/periph.txt` runs them from Forth in the system testbench.

### PWM

`rtl/pwm.v` has four channels. Each counts 50 MHz clocks from 0 to period − 1 and drives
its output high for the first (period × duty) / 65536 clocks; the 32 × 16 product is
pipelined in two stages (a DSP48 per channel). New period and duty values are taken at the
end of the running cycle, so no pulse is cut short; enabling starts a fresh cycle. An
enabled channel owns its pin: `forth_system` drives the pin as an output with the channel's
level (channels on the same pin are ORed); disabled, the pin returns to its latch and
direction. Reset (also a CPU restart) turns every channel off. The ISS keeps the registers
but has no time base: a PWM pin reads high there only at full duty.

### XADC

`xadc_scan` instantiates the XADC primitive in continuous sequence mode (ADCCLK = 12.5 MHz,
16-sample averaging, unipolar, extended settling on the auxiliary inputs). A scanner
reads status registers 0x00..0x1F over the DRP one after another into a 32 × 16
register file, which the CPU reads without waiting. The first DRP read is held back 4096
clocks after configuration; a request during the XADC's own reset is never answered.

| Forth channel | Shield | XADC input | Status register | Volts at the XADC |
|---|---|---|---|---|
| 0..5 | A0..A5 | VAUX 1, 9, 6, 15, 5, 13 | 0x11, 0x19, 0x16, 0x1F, 0x15, 0x1D | shield voltage / 3.32 (2.32k/1k divider) |
| 6 | A6 (+), A7 (−) | VAUX 12 | 0x1C | A6 − A7, 0..1 V |
| 7 | A8 (+), A9 (−) | VAUX 0 | 0x10 | A8 − A9, 0..1 V |
| 8 | A10 (+), A11 (−) | VAUX 8 | 0x18 | A10 − A11, 0..1 V |

On-chip sensors in the sequence: 0x00 temperature (°C = code × 503.975 / 4096 − 273.15),
0x01 VCCINT, 0x02 VCCAUX, 0x06 VCCBRAM, 0x0D VCCPINT, 0x0E VCCPAUX, 0x0F VCCO_DDR
(volts = code × 3 / 4096).

In simulation the Xilinx XADC model reads its inputs from `sim/xadc_stim.txt`;
`tools/hf32.py` uses the same values (table `XADC_SIM`).

## AXI-Lite registers (ARM side)

The same registers as in the 16-bit design, so the board view and the HASE board functions
read the same values; only the loader and the PC field are wider.

| Offset | Register |
|---|---|
| 0x00 | CTRL: bit 0 = hold CPU in reset (1 after PL reset), bit 1 = pause the CPU |
| 0x04 | MEM_ADDR: RAM cell address (15 bits) for the loader |
| 0x08 | MEM_DATA: write stores the 32-bit cell at MEM_ADDR and increments it (only while in reset); read returns the cell at MEM_ADDR (while in reset or paused) |
| 0x0C | TX_POP: bit 8 = valid, bits 7..0 = byte from the CPU; reading pops it |
| 0x10 | RX_PUSH: write a byte for the CPU |
| 0x14 | STATUS: bit 0 = RX FIFO full, bit 1 = TX FIFO not empty, bits 7..4 = BTN3..BTN0 and bits 9..8 = SW1..SW0 (debounced, as at 0x10000C), bits 30..16 = CPU PC (cell address) |
| 0x18 | BOARD: bits 3..0 = LD3..LD0, bits 9..4 = RGB (as at 0x100010), bits 19..16 = BTN3..BTN0, bits 21..20 = SW1..SW0 |
| 0x1C, 0x20, 0x24 | PORT0..PORT2: bits 15..0 = pin levels (as at 0x100020 + 16p), bits 31..16 = pins driven (direction or PWM) |
| 0x28 | PWM: bits 3..0 = channel enables, then the pin of channel c in bits 9+6c..4+6c |
| 0x2C | XADC: write selects a status register 0..31; read returns bits 20..16 = the selection, bits 15..0 = its value (the scanner's second read port) |
| 0x30 | PG_STATUS: bits 27..16 = the pattern generator's current step, bits 6..0 = its status (as at 0x100180) |
| 0x34 | PG_OUT: bits 31..16 = channel enable mask, bits 15..0 = current output levels |
| 0x38 | FUNC: write selects a pin-function entry 0..13; read returns bits 19..16 = the selection, bit 15 = on, bits 5..0 = the pin (see *Pin router*) |

Registers 0x18 to 0x34 let the ARM show the board's state in the app while Forth runs; they
only read, so they never disturb the CPU.

**Pause.** CTRL bit 1 freezes the CPU between two instructions: no register changes, no
store, no I/O side effects, and RAM port B goes to MEM_ADDR/MEM_DATA, so the ARM can read
RAM while a program runs. When the bit is cleared the CPU stays stalled one more cycle,
in which port B re-reads the CPU's [T] and the code port the current instruction; then it
continues as if nothing had happened (except that the timers kept running and a running
floating-point operation may have finished). The RTL testbench pauses at random moments
throughout its runs and checks that the output does not change.

## The ARM program (`sw/src`)

Bare metal on Cortex-A9 core 0, polling. At start it waits 50 ms (so the debounced BTN3 is
valid), reads the saved source from flash, loads and verifies the kernel image and releases
the CPU. Then it loops:

- UART bytes go to the input ring (64 KB) unless they belong to a service request (SOH ...
  LF, see [Protocol.md](Protocol.md)) or to the HASE link; Ctrl-R restarts Forth.
- The input ring and the autostart feed go into the RX FIFO whenever it has room.
- Forth's TX FIFO goes into a 4 KB output ring, which drains into the UART as fast as it
  sends. When the ring is full the TX FIFO stays full and Forth waits in `emit`, so no output
  is lost. A 0x01 from Forth is doubled on the way out.
- An answer first flushes the output ring, then goes out whole, so it never splits Forth's
  output.

**Saved source.** One source, at most 65,520 bytes, lives in the 64 KB flash sector at
0xFE0000, far above `BOOT.bin`, behind a 16-byte header (magic `HF32`, length, CRC-32 of
the text, autostart flag; little-endian). The 16-bit ZYNQ-FORTH keeps its own source at
0xFF0000 (magic `HF16`), so switching between the two firmwares loses neither. Lines end in
CR, as typed at the terminal. At power-up and at every restart, if autostart is on, the ARM
feeds the source into Forth's input right after the kernel, as if it were typed; holding
**BTN3** during the start or `restart skip` skips it. While the flash is erased or programmed
(up to ~3 s) nothing moves, so Forth may wait on a full TX FIFO for that time.

**Dictionary.** The ARM pauses the CPU (CTRL bit 1), reads `latest` and `dp` (their
addresses are in `build/forth_image.h`, written by `tools/hf32.py`), follows the header
links downwards and formats the answer, then resumes the CPU.

**Stacks.** `quit` copies the data stack into `ssnap` and the float stack into `fssnap`
before each line, with `sdepth` = -1 while it copies; `fsdepth` holds the float depth. The
ARM pauses the CPU and reads the depths and the items; if it caught the copy half-done, it
lets the CPU run 10 µs and tries again (up to 100 times).

**HASE.** The same firmware is the HASE Forth-Lab MCNF node (node type AE.07.30.80,
firmware 3.0), switching automatically between the terminal and the HASE host as described
in [Protocol.md](Protocol.md#hase-mode). It answers the functions of the 16-bit node as
before, so today's Forth-Lab panel works with it; where those carry 16-bit values the node
clips (stack cells to −32768..32767, addresses above 0xFFFF to 0xFFFF). Functions 0x45 to
0x48 carry the 32-bit data stack, the float stack and the full dictionary addresses.

## Dictionary layout

```
header:  link (cell, address of previous header, 0 ends the list)
         count byte: bits 4..0 = name length, bit 7 = IMMEDIATE, bit 6 = HIDDEN, bit 5 = INLINE
         name bytes, padded to a cell boundary
xt:      code
```

A `create`d word's code is a literal of its data address and an exit, so its data starts at
xt + 8.
