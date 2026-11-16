#  Project P3: 8-bit RISC System-on-Chip (SoC) with Timer & SAR ADC

**Note:** This repository (**Project P3**) is an advanced System-on-Chip (SoC) integration built directly upon the core architecture developed in **[Project P2](#link-to-your-P2-repo)**. 

While **Project P2** focused on the RTL design of a standalone **8-bit 5-stage Pipelined RISC CPU**, **Project P3** elevates this core into a fully functional microcontroller SoC. It integrates a memory-mapped Hardware Timer, a 4-channel SAR ADC Controller with a synchronous FIFO, and interrupt handling capabilities.

---

##  Key SoC Features & Custom Upgrades

Building upon the base CPU, this SoC introduces several advanced features and custom upgrades beyond standard requirements:

### CPU Core Upgrades (from P2)
* **Early JMP in ID Stage:** Branch penalty reduced from 2 cycles to 1 cycle.
* **Full Status Flags (Z, C, N, V):** Added Negative (N) and Overflow (V) flags for signed comparisons.
* **Memory-Mapped I/O (MMIO):** Dedicated I/O page from `0xF0 - 0xFF` mapped directly into the MEM stage with zero wait states.

### Peripheral Integration (APB-Style Bus)
* **8-bit Hardware Timer:** Configurable prescaler, one-shot/continuous modes, and a dedicated **30% duty cycle PWM output pin**.
* **4-Channel 8-bit SAR ADC Controller:** 
  * Features a window comparator (limit thresholds).
  * **Continuous Mode Upgrade (`TRIG_SEL = 3`):** ADC auto-restarts after each conversion.
  * **Trigger Miss Flag (`IRQ bit 5`):** Reports timer triggers lost while the ADC is busy.
* **Synchronous Result FIFO:** 8-entry deep FIFO to store ADC conversion results (prevents data loss).
* **Interrupt Controller:** 6 configurable IRQ flags (OVF, CMP, EOC, WD, FIFO_OVR, TRIG_MISS) driving a single `irq` pin.

---

##  Memory Map (`0xF0 - 0xFF`)

| Address | Register Name | R/W | Description |
| :--- | :--- | :--- | :--- |
| `0xF0 - 0xF3` | `TMR_CTRL`, `PRESC`, `LOAD`, `CMP` | RW | Timer configuration & PWM duty cycle |
| `0xF4` | `TMR_CNT` | RO | Current timer count |
| `0xF5 - 0xF9` | `ADC_CTRL`, `SMP`, `DIV`, `THR_LO`, `THR_HI` | RW | ADC configuration & Window limits |
| `0xFA` | `ADC_DATA` | RO | Oldest ADC result (**Reading pops FIFO**) |
| `0xFB - 0xFC` | `ADC_RCH`, `ADC_STAT` | RO | ADC Channel & FIFO Status (Empty/Full/Level) |
| `0xFD - 0xFE` | `IRQ_EN`, `IRQ_STAT` | RW/W1C | Interrupt Enables & Status Flags (Write 1 to Clear) |
| `0xFF` | `ID` | RO | Hardcoded ID = `0xA8` |

---

##  Verification & Simulation

The system is rigorously verified using self-checking testbenches and **53 SystemVerilog Assertions (SVA)** bound directly into the DUT.

### Firmware Co-Verification
The Instruction Memory (`imem.v`) is hardcoded with a 150-instruction polling firmware (`soc_demo.s`) that exercises the entire SoC end-to-end:
* **Part A:** Reads ID register.
* **Part B:** Triggers ADC Ch1 via software start.
* **Part C:** Timer-triggered conversions on Ch2 (x4), reads and clears flags.
* **Part D:** Tests Window limits (Ch3) and verifies `irq` pin rises and clears.
* **Part F:** Tests FIFO popping with back-to-back reads.
* **Part E:** Generates a 30% duty cycle PWM and executes `HLT`.
  
##  Results & Implementation

### Verification Results
- **`tb_risc8_soc`**: 22/22 checks PASSED (638 cycles, 121 I/O accesses).
- **`tb_timer_adc8`**: 45/45 checks PASSED (Peripheral standalone test).
- **SVA**: 0 failures across 53 SystemVerilog Assertions.

###  Implementation Results (Artix-7 @ 100 MHz)
The design was synthesized and implemented on a Xilinx Artix-7 FPGA target.

- **Timing**: Met 100 MHz constraint.
  - Worst Negative Slack (WNS): **+1.518 ns** (Fmax ≈ 117.9 MHz)
  - Failing Endpoints: **0**
- **Utilization**:
  - Total LUTs: **540** (2.55%)
  - Registers (FFs): **416** (1.00%)
  - DSP / BRAM: **0**
- **Power**: 
  - Total on-chip power: **0.077 W** (Dynamic: 0.007 W, Static: 0.070 W)

### How to Run Simulation (Vivado 2026.1)
No external `.hex` files are required. The firmware is embedded in the RTL.
1. Clone the repository and open in Vivado.
2. Set `tb_risc8_soc` as the top simulation module.
3. Run the Tcl commands:
   ```tcl
   launch_simulation
   run all
