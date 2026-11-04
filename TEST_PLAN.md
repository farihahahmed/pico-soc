# Test Plan — PicoRV32 SoC on GF180MCU (A45)

Post-silicon plan for the packaged chip. Every test reuses firmware already in
`firmware/` and the scan protocol in `rtl/scan_chain.sv`, so each silicon
result maps to a passing pre-silicon test (`VERIFICATION.md`).

## 1. Equipment

| Item | Purpose |
| --- | --- |
| Bench supply, 5.0 V, current-limited | VDD/VSS. Expected ~10 mA (48.1 mW nominal) |
| Function generator or 25 MHz oscillator | `clk` (40 ns period) |
| Scan host: microcontroller or FPGA (e.g. RP2040) | Drives `scan_in/shift/load/i0o1`, `rst_n`, `clk_int`; reads `scan_out` |
| USB-UART adapter | `uart_tx` / `uart_rx`, 115200 8N1 |
| Logic analyzer / oscilloscope | GPIO, SPI, clocks, scan timing |
| Test PCB: socket, decoupling, 4 LEDs, 2 buttons, piezo, SPI header | Demo hardware |

All I/O is 5 V (GF180MCU 5 V pads). Use 5 V-tolerant or level-shifted host I/O.

## 2. Scan protocol (reference)

48-bit frame `{tgt[1:0], addr[13:0], data[31:0]}`, shifted on `scan_shift`,
applied on `scan_load`.

| tgt | Action |
| --- | --- |
| 0 | Write instruction-memory word |
| 1 | FSM config: mode (0 IDLE, 1 RUN, 2 COUNTDOWN) + 16-bit count |
| 2 | Clock-gen config: source + divider (internal clk = clk / 2·(div+1)) |
| 3 | Read memory word; with `addr[13]=1`, read status word |

Status word: `{28'b0, scan_owns_mem, fsm_mode[1:0], trap_sticky}`.

## 3. Test sequence

Stop at the first failure and debug before moving on.

### T1 — Power and continuity
- Diode check every signal pin to VSS/VDD before power-up.
- Power with 20 mA current limit. Pass: steady current near 10 mA, no heating.

### T2 — Reset and scan chain
- Hold `rst_n` low, apply `clk`, release reset.
- Read status word (tgt=3, addr[13]=1). Pass: `fsm_mode = IDLE`, `scan_owns_mem = 1`, `trap_sticky = 0`.
- Shift a known 48-bit pattern through and confirm it on `scan_out`.

### T3 — Clocking
- `clk_int = 0`: system runs from external `clk`.
- `clk_int = 1` + tgt=2 config: check divided clock on a GPIO toggle program for several divider values.
- Pass: measured frequency matches clk / 2·(div+1).

### T4 — Memory load and readback
- Load each word of a test image with tgt=0, read all back with tgt=3.
- Patterns: all-0, all-1, 0x55/0xAA, address-as-data.
- Pass: every word matches (checks the narrow 8-bit SRAM and byte gather).

### T5 — CPU execution (FSM)
- Load `fw_gpio_walk`, set FSM to RUN.
- Pass: walking-1 on `gpio_out[3:0]`, UART prints `GW+`, status `trap_sticky = 0`.
- Repeat with COUNTDOWN mode: CPU stops after the programmed cycle count.

### T6 — Peripherals (self-test firmware)

| Firmware | Setup | Pass |
| --- | --- | --- |
| `fw_dmem_walk` | none | UART `D+` |
| `fw_uart_echo` | send 3 bytes from host | bytes echoed + `E+` |
| `fw_spi_loop` | tie `spi_miso` high | UART `S+`, SCLK on scope |
| `game` | press each button | sprite moves (exercises `gpio_in[1:0]`) |

UART baud is derived from the 12.5 MHz system clock; recompute the host baud if the clock changes.

### T7 — Custom accelerator (PCPI)
- `fw_pcpi_check`: UART prints `abP+` (CRC32 of "123456789" = 0xCBF43926).
- `pcpi_demo`: all 7 custom-0 instructions match the Python golden model.
- `cycles_demo` / `crc_demo`: measure speedup. Target ~10.3× on CRC32 (`docs/PCPI_BENCHMARKS.md`).

### T8 — Demos
- `primes` (UART), `piezo_tune` (GPIO out[0]), `game` (SPI LCD + buttons), `fir_demo` (UART).

### T9 — Characterization
- Max frequency: sweep `clk` until T5–T7 fail. Pre-silicon worst-case fmax is 13.2 MHz CPU clock.
- Supply sweep: 4.5–5.5 V at nominal clock.
- Current vs. frequency and FSM mode (IDLE vs RUN), to show clock-gating savings.
- Repeat T4–T7 on every available die.

## 4. Pass criteria and reporting

The chip passes when T1–T7 pass on at least one die. For each test, record die
ID, supply, clock, and result in `test_results/`, along with scope or
logic-analyzer captures. Compare against simulation results and publish them in
this repo.
