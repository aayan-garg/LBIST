# Low-Switching Scan-Based LBIST for CMIPS32

VHDL implementation and verification of a programmable scan-based Logic Built-In Self-Test (LBIST) system integrated into the 5-stage pipelined **CMIPS32** processor[cite: 14]. Based on the dynamic Pseudo Low-Pass Filter (PLPF) method by Kato et al., the hardware controls scan-shift switching activity (**Weighted Transition Metric, WTM**) to arbitrary target levels without adding test clock cycles or degrading capture-cone fault coverage[cite: 1, 14, 15, 29].

---

## Overview

Pseudo-random patterns from a standard LFSR exhibit ~50% bit-to-bit toggle rates, causing excessive dynamic switching during serial scan shifts[cite: 1, 4, 14]. Rather than locking the test generator to a single fixed low-power filter, this design dynamically switches a combinational PLPF between filtered ($n=3$) outer regions and an unfiltered ($n=1$) middle window during each 64-bit scan shift[cite: 4, 15, 16].

* **Target Core:** 5-stage pipelined CMIPS32 (`IF`, `RF`, `EX`, `MM`, `WB`)[cite: 14].
* **Scan Chain:** 1 serial chain of **64 scan cells** replacing `reg_IF_RF` (`PCincd_q[31:0] & RF_instr[31:0]`), leaving the remaining 656 pipeline candidate bits unscanned to avoid control-state hazards[cite: 14, 15, 16].
* **Runtime Modes:** `Direct` ($n=1$), `Constant n=3`, `Basic`, `Swap`, and `Moving` selectable on a single synthesized netlist[cite: 15, 16].
* **FPGA Target:** Digilent Basys 3 (Artix-7 `xc7a35tcpg236-1`), synthesized and routed out-of-context in AMD Vivado/XSim 2025.1.1[cite: 14, 21].

---

## System Architecture

The top-level wrapper (`final_processor_system.vhd`) supports three build configurations: **(A)** original CMIPS32 core, **(B)** scan-inserted core with state isolation, and **(C)** complete self-contained LBIST system[cite: 15, 21].

1. **16-Bit LFSR & Zero-Latency Phase Shifter**
   * Primitive polynomial $x^{16} + x^{15} + x^{13} + x^4 + 1$, reset seed `0xAAAA`, with all-zero/unknown lockup protection across all 65,535 nonzero states[cite: 15].
   * Combinational phase shifter taps `state(0)`, `state(1)`, and `state(2)` simultaneously as current ($T_j$), next ($T_{j+1}$), and next-next ($T_{j+2}$) bits with zero cycle overhead[cite: 15].

2. **Dynamic PLPF & Programmable Pattern Router**
   * Uses live scan-input stage feedback `Q(0)` to select between an $n$-input AND gate (`Q(0) = '0'`) and an $n$-input OR gate (`Q(0) = '1'`)[cite: 15].
   * Supports mode `11` ($n=1$ bypass), mode `01` ($n=2$), and mode `00` ($n=3$)[cite: 15].
   * On-chip shift/pattern counters drive three dynamic schedules across the $L = 64$ scan chain (enforcing the $\ge 14$-bit minimum length on each $n=3$ filtered end)[cite: 16]:
     * **Basic ($\beta = 20$):** Fixed regions $(\alpha, \beta, \gamma) = (22, 20, 22)$[cite: 16].
     * **Swap ($\beta = 16$):** Alternates $(\alpha, \beta, \gamma)$ between $(16, 16, 32)$ and $(32, 16, 16)$ on consecutive patterns[cite: 16].
     * **Moving ($\beta = 20$):** Slides the unfiltered window bit-by-bit via $\gamma = 14 + (p \bmod 17)$ and $\alpha = 64 - 20 - \gamma$[cite: 16].

3. **Processor Test-Mode Isolation (`core_scan_test`)**
   * `TM / ACK` entry handshake waits for an idle memory bus before starting test operations[cite: 15, 16].
   * Freezes `PC`, downstream pipeline registers (`RF/EX`, `EX/MM`, `MM/WB`), `GPR`, and `CP0`, while suppressing all memory and register write commits[cite: 15, 16].
   * Only a legal `CAPTURE` pulse updates `reg_IF_RF_scan`; exiting test mode requires a reset[cite: 16].

4. **Test Controller & Completion-Safe 16-Bit MISR**
   * Executes **256 patterns** (`64 SHIFT` $\rightarrow$ `1 CAPTURE` $\rightarrow$ `64 UNLOAD` edges per pattern = **33,024 payload cycles**) with `start`, `pause`, and `abort` controls[cite: 14, 15, 16, 20].
   * Compresses serial `Q(63)` unload bits using $x^{16} + x^{12} + x^5 + 1$ (mask `0x1021`, seed `0x0000`)[cite: 15, 16].
   * Gates `signature_valid`, `MATCH`, and `fault_detected` low until the controller reaches `DONE` (`101`) with all 256 captures, 16,384 response bits, and zero sticky error flags[cite: 16, 17].

---

## Experimental Results

### 1. Scan-Shift Switching (WTM), Fault Coverage & Test Time
Evaluated across **256 patterns** against **16 single stuck-at faults** (`SA0`/`SA1` across 8 capture-input sites: `increment_operand` bits `2, 3, 16, 31` before the `+4` adder and `instruction_input` bits `0, 1, 16, 31` before `reg_IF_RF_scan`)[cite: 14, 17, 18]:

| Approach | Schedule ($\beta$) | Measured `SHIFT` WTM | Capture-Cone Coverage | Golden Signature | Aliases | Payload Cycles |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Direct ($n=1$)** | fixed[cite: 18] | 50.1154%[cite: 18] | 12 / 16 (75%)[cite: 18, 19] | `0x9A6A`[cite: 18] | 0[cite: 18] | 33,024[cite: 18] |
| **Constant $n=3$** | fixed[cite: 18] | 7.0665%[cite: 18] | 12 / 16 (75%)[cite: 18, 19] | `0x5DCB`[cite: 18] | 0[cite: 18] | 33,024[cite: 18] |
| **Basic** | 20[cite: 18] | 19.9764%[cite: 18] | 12 / 16 (75%)[cite: 18, 19] | `0xAFBC`[cite: 18] | 0[cite: 18] | 33,024[cite: 18] |
| **Swap** | 16[cite: 18] | 17.7531%[cite: 18] | 12 / 16 (75%)[cite: 18, 19] | `0x9C7F`[cite: 18] | 0[cite: 18] | 33,024[cite: 18] |
| **Moving** | 20[cite: 18] | 19.8701%[cite: 18] | 12 / 16 (75%)[cite: 18, 19] | `0xF21E`[cite: 18] | 0[cite: 18] | 33,024[cite: 18] |

* All five modes reach **12/16 (75%)** detection within the first **4 patterns (516 cycles)** with zero final-signature aliases[cite: 18, 19, 20]. The remaining 4 faults (`IDs 9, 12, 13, 15`) match the fixed polarity of the held ROM instruction (`0x1460FFFA` at PC `0x2C`) and are structurally unexcited[cite: 17].

### 2. FPGA Utilization, Timing Closure & SAIF Power (Basys 3 / Artix-7)
Out-of-context post-route results at the timing-clean **9.756098 MHz** MMCM operating point (`M = 8, D = 82`, `102.5 ns` period; `3.39 ms` per 256-pattern run)[cite: 14, 21, 24, 25]:

| Configuration | LUTs | FFs | BRAM36 | DSPs | WNS / WHS | Timing Clean |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **A: Original CMIPS32** | 5,220[cite: 24] | 2,008[cite: 24] | 1[cite: 24] | 7[cite: 24] | +0.154 ns / +0.069 ns[cite: 24] | PASS[cite: 24] |
| **B: Scan + Isolation** | 6,138[cite: 24] | 3,084[cite: 24] | 0[cite: 24] | 7[cite: 24] | +1.120 ns / +0.001 ns[cite: 24] | PASS[cite: 24] |
| **C: Complete LBIST** | 6,243[cite: 24] | 3,215[cite: 24] | 0[cite: 24] | 7[cite: 24] | +0.155 ns / +0.007 ns[cite: 24] | PASS[cite: 24] |
| **Incremental LBIST (B → C)** | **+105 (+1.71%)**[cite: 24] | **+131 (+4.25%)**[cite: 24] | 0[cite: 24] | 0[cite: 24] | — | — |

* Vivado RTL SAIF dynamic power estimates remain ~**0.099 W** across all five approaches at 9.756 MHz (and **0.123–0.124 W** at the historical 50 MHz run, where shared MMCM clock power accounted for 0.106 W)[cite: 22, 23, 25]. Reported power savings reflect measured scan-chain switching activity (`WTM`), not physical board measurements[cite: 14, 27].

---

## Repository Structure

```text
├── CMIPS/vhdl/                  # Original CMIPS32 processor VHDL sources[cite: 14, 28]
├── Verification/                # XSim testbenches, safety/abort checks, and audit logs[cite: 18]
├── docs/                        # Scan register inventory and architectural documentation[cite: 14]
├── tables/                      # Fault inventory, target feasibility, and comparison CSVs[cite: 17, 19, 28]
└── Results/
    ├── Baseline/                # Unmodified CMIPS32 simulation logs and waveforms[cite: 31]
    └── Final/
        ├── DEMO.md              # Step-by-step simulation and reproduction instructions[cite: 28, 31]
        └── FPGA_Analysis/       # 50 MHz baseline and 9.756 MHz timing-closed Vivado reports[cite: 14, 25, 31]
            ├── Timing_Closure/  # Closed checkpoints (m8_d82), frequency sweeps, and SAIF logs[cite: 31]
            └── power/           # Per-approach switching and power reports (approach_0..4)[cite: 31]
