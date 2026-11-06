# High-Performance Digital Smart Home SoC  (Verilog RTL, FPGA)

## Folder layout
| Path | Content |
|---|---|
| `rtl/` | Synthesizable Verilog-2001 modules (non-blocking `<=` in all sequential logic) |
| `tb/tb_soc_top.v` | Self-checking testbench (31 checks, prints PASS/FAIL) |
| `sim/` | `run_modelsim.do` (ModelSim) and `run_sim.sh` (Icarus) |
| `constraints/basys3.xdc` | Pin constraints, Basys3 / Artix-7 |
| `docs/REPORT_TEMPLATE.md` | Skeleton of the 10-15 page utilization report |

## Module hierarchy
```
board_top (Basys3 wrapper)
 └─ soc_top  (6. Core RTL Integration Engine: reset tree + integration)
     ├─ reg_file       (5. Global State & Security Governance Register)
     ├─ thermal_block  (1. 8-bit comparator + hysteresis loop)
     ├─ luminance      (2. IIR low-pass + Schmitt switching)
     ├─ security_fsm   (3. NORMAL/ARMED/ALARM FSM + 2 x clock_divider)
     │    └─ sync_edge x2
     └─ gate_ctrl      (4. 50 Hz servo PWM + auto-close counter)
          └─ sync_edge
```

## Register map
| Addr | Name | Notes |
|---|---|---|
| 0 | MODE | `00` Comfort, `01` Lockdown (`1x` ignored) |
| 1 | TEMP_THR | default 75 |
| 2 | LUX_THR | default 64 |
| 3 | STATUS | read-only `{2'b0,intr,gate_open,light_on,cooling,sec_state}` |

## Behaviour summary
* **Thermal**: cooling + buzzer ON when temp > threshold; OFF only when temp < threshold-5 (hysteresis). Lockdown lowers threshold by 5.
* **Luminance**: IIR filter y += (x-y)/8 at 1 kHz sample rate; lamp ON if y < LUX_THR, OFF if y > LUX_THR+16. Lockdown forces lamp ON.
* **Security**: arm -> ARMED; rising motion edge -> ALARM (8 Hz strobe + 1 kHz tone); disarm -> NORMAL. Lockdown forces ARMED.
* **Gate**: doorbell edge -> servo pulse 1 ms -> 2 ms; auto-closes after 250 PWM periods (5 s). Lockdown ignores doorbell and closes gate.

## Run the simulation
ModelSim (Verilog-2001, no SystemVerilog needed):
```
vsim -do sim/run_modelsim.do
```
Expected console end: `31 checks, 0 errors -> ALL TESTS PASSED`.
The TB scales `CLK_HZ` to 100 kHz so the 5 s gate timer and 1 kHz tone simulate in seconds. Take your waveform screenshots from the ModelSim Wave window (the .do file already groups the signals).

## Vivado flow (Basys3 / Artix-7)
1. New RTL project, part `xc7a35tcpg236-1`. Add every file in `rtl/`, constraints `constraints/basys3.xdc`. Set `board_top` as top.
2. Add `tb/tb_soc_top.v` as a simulation source (top = `tb_soc_top`).
3. Run Synthesis -> **Open Synthesized Design -> Schematic** (RTL Analysis -> *Open Elaborated Design -> Schematic* for the RTL view).
4. Run Implementation -> **Report Utilization** (LUTs, FFs, I/O) and **Report Timing Summary**.
5. Generate Bitstream -> Hardware Manager -> Program Device. Record a short video/photos for the demo.

For **Intel DE10-Lite / Cyclone** (Quartus Prime): keep `soc_top` as is, make a small top that maps `clk`=50 MHz, switches/buttons/LEDs, and **set `CLK_HZ` = 50_000_000** in the `soc_top`/`board_top` instantiation.

## FPGA demo script (Basys3)
| Action | Expected |
|---|---|
| Press btnD, release | all outputs reset |
| sw[7:0] = 80 (above 75) | LED0 (cooling) + LED1 (overheat) ON, JA3 high |
| sw[7:0] = 72 | still ON (hysteresis); at 60 -> OFF |
| sw[11:8] = 0000 (dark) | LED2 lamp ON; 1111 (bright) -> OFF |
| sw[14] = 1 | LED6 armed, LED11:10 = 01 |
| Press btnU | alarm: LED3 strobes, LED4/JA2 tone, LED7 intr, state = 10 |
| sw[14] = 0 | back to normal |
| Press btnR | LED5 gate open, servo (JA1) moves, closes after 5 s |
| sw[13:12] = 01, press btnL | Lockdown: LED8 on, forced armed, doorbell ignored |
| sw[7:0] = N, press btnC | writes new temperature threshold N |

Servo on Pmod JA1 needs 5 V power from a separate supply or the board's 5 V pin, and common ground.

