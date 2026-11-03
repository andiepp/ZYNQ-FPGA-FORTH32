# Changelog

ZYNQ-FPGA-FORTH32 continues the version numbers of the 16-bit
[ZYNQ-FORTH](https://github.com/andiepp/ZYNQ-FORTH) (whose changelog has releases 1.0 to
2.3), as its 32-bit generation 3.x.

## 3.0 — unreleased

Started from ZYNQ-FORTH 2.3.

- **HF32 CPU**: 32-bit cells and 32-bit instructions in the same J1-style classes, 128 KB of
  RAM (was 16 KB), 15-bit cell-address program counter, 32-entry data, return and float
  stacks, arithmetic right shift. One instruction per clock at 50 MHz.
- **Floating point** in hardware (`rtl/hf32_mcu.v`): IEEE single precision, round to
  nearest even, subnormals flushed to zero, one quiet NaN; add, subtract, multiply, divide,
  square root, int↔float conversion and truncation, plus compares and sign operations in the
  ALU. The 32 × 32 multiply moved into the same multi-cycle unit (3 clocks). A bit-exact
  Python model (`tools/hf32.py`) is checked against IEEE (`tools/test_fpu.py`) and the RTL
  against the model (`sim/run_mcu.ps1`).
- **Kernel**: all words of ZYNQ-FORTH 2.3 with the same names and stack effects, now on
  32-bit cells (durations and periods stay double cells, where a double `lo hi` means
  `lo + hi*65536`, so 16-bit sources keep working), plus `us@` (a microsecond counter),
  `arshift`, `s>d`, `dnegate`, `fill`; the floating-point words: float literals,
  `f@ f! fconstant fvariable fliteral`, stack words, arithmetic, compares, `floor fround
  ftrunc`, `s>f f>s d>f f>d`, `>float represent precision set-precision f. fs. fe. f.s`,
  `pi fsin fcos fsincos ftan fasin facos fatan fatan2 fexp fln flog falog f** fsinh fcosh
  ftanh flnp1 fexpm1`, and `volts@ ftemp@`. Number conversion in both directions is correctly
  rounded; the functions are within about 2 ulp.
- **I/O map**: register k at `$100000 + 4k` (was `$8000 + 2k`); the ms counter is 32 bits.
- **Firmware 3.0, protocol 2**: `stack` answers 32-bit cells and a second line with the float
  stack; `words` addresses have five hex digits; `version` adds `cell 32` and `float 32`.
  The saved source has its own flash sector (0xFE0000, magic `HF32`), so the 16-bit one at
  0xFF0000 survives.
- **HASE**: still the Forth-Lab MCNF node (type AE.07.30.80, reports 3.0) with the 16-bit
  node's functions, so the HASE Forth-Lab panel works unchanged (stack cells clipped to
  16 bits there); new functions 0x45 (data stack, 32 bits), 0x46 (float stack), 0x47 and
  0x48 (dictionary with 32-bit addresses).
- **App 3.0** (`ZynqForth32.exe`, product ZYNQ-FPGA-FORTH32): the stack panel shows 32-bit
  cells and a float stack list; settings in `%LocalAppData%\ZYNQ-FPGA-FORTH32`.
