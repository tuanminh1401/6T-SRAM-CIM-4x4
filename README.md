# 4x4 6T SRAM Analog Computing-in-Memory (CIM) Array

An analog Computing-in-Memory (CIM) macro based on a **4x4 6T SRAM array** designed and verified in **Cadence Virtuoso / Spectre** using a **90nm Generic PDK (`gpdk090`)** at $V_{DD} = 1.0\text{ V}$. The architecture executes parallel analog Multiply-Accumulate (MAC) operations directly inside the memory bitlines using charge-domain bitline discharge and **Wordline Underdrive (WLUD)** to prevent read disturb.

---

## 1. Repository Structure

    6T-SRAM-CIM-4x4/
    ├── virtuoso_lib/
    │   ├── cds.lib                  # Virtuoso library definition
    │   └── MinhNT/                  # Custom design library (SRAM_6T, SRAM_CIM_4x4, tb_SRAM_CIM_4x4)
    ├── netlist/
    │   ├── input.scs                # Spectre simulation netlist
    │   └── spectre6.ocn             # OCEAN script for simulation setup
    ├── images/                      # Schematic and transient waveform screenshots
    └── README.md

---

## 2. Circuit Architecture (Bottom-Up Design)

### 2.1. 6T SRAM Bitcell (`SRAM_6T`)
* **Technology:** `gpdk090` ($L = 100\text{ nm}$, $V_{DD} = 1.0\text{ V}$)
* **Transistor Sizing:**
  * Pull-down NMOS (`NM2`, `NM3`): $W/L = 240\text{ nm} / 100\text{ nm}$
  * Access NMOS (`NM0`, `NM1`): $W/L = 120\text{ nm} / 100\text{ nm}$ (Cell Ratio $\text{CR} = 2.0$ for read stability)
  * Pull-up PMOS (`PM0`, `PM1`): $W/L = 120\text{ nm} / 100\text{ nm}$

![6T SRAM Bitcell Schematic](images/Screenshot%202026-11-22%20001534.png)
*Figure 1: Schematic of the 6T SRAM bitcell with transistor sizing optimized for read stability.*

### 2.2. 4x4 SRAM CIM Array (`SRAM_CIM_4x4`)
* Organized as 4 wordline rows (`WL0`–`WL3`) and 4 differential bitline columns (`BL0/BL_INV0`–`BL3/BL_INV3`).
* Each column is terminated with an explicit bitline capacitance $C_{BL} = 25\text{ fF}$ and pre-charge switches.

![4x4 SRAM CIM Array Schematic](images/Screenshot%202026-11-22%20104913.png)
*Figure 2: Top-level schematic of the 4x4 6T SRAM CIM macro.*

### 2.3. Simulation Testbench (`tb_SRAM_CIM_4x4`)
* **Stored Weight Matrix ($W_{4\times 4}$):** Thermometer-coded column weights storing `1`, `2`, `3`, and `4` active `'1'` bits across columns `0` to `3`, respectively.
* **Input Vector ($X$):** All-ones activation vector $[1, 1, 1, 1]^T$ applied simultaneously during the compute phase.

![Testbench Schematic](images/Screenshot%202026-11-22%20100920.png)
*Figure 3: Complete Spectre testbench including pre-charge switches, 25 fF bitline capacitors, and PWL wordline drivers.*

---

## 3. Operating Phases & Transient Verification

The transient simulation (`0 ns – 20 ns`) operates across three distinct phases:
1. **Write Phase (`0.5 ns – 4.5 ns`):** Sequential row-by-row writing ($V_{WL} = 1.0\text{ V}$) to program the $4\times 4$ weight matrix into the SRAM cells.
2. **Pre-charge Phase (`4.6 ns – 11.0 ns`):** All `BL` and `BL_INV` nodes are pre-charged to $V_{DD} = 1.0\text{ V}$ while wordlines remain idle at $0\text{ V}$.
3. **Parallel Compute Phase (`11.0 ns – 12.0 ns`):** All four wordlines (`WL0`–`WL3`) are asserted simultaneously for $\Delta t = 1.0\text{ ns}$ using **Wordline Underdrive ($V_{WL,\text{compute}} = 250\text{ mV}$)**.

![Full Transient Operation](images/Screenshot%202026-11-22%20100938.png)
*Figure 4: Transient waveforms illustrating sequential Write, Pre-charge, and parallel Compute phases.*

![Analog MAC Output Levels](images/Screenshot%202026-11-22%20101349.png)
*Figure 5: Bitline output voltages (`BL_INV0`–`BL_INV3`) sampled at t = 15.0 ns corresponding to MAC outputs `[1, 2, 3, 4]`.*

---

## 4. Quantitative Performance & Linearity Analysis

### 4.1. Analog MAC Output & Linearity (Sampled at $t = 15.0\text{ ns}$)

| Column | Expected MAC | Final $V_{\text{BL-INV}}$ (mV) | Voltage Drop $\Delta V$ (mV) | Step Size $\Delta V_{\text{step}}$ (mV) | DNL (LSB) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **BL_INV0** | 1 | 867.66 | 132.34 | 132.34 | +0.31 |
| **BL_INV1** | 2 | 760.97 | 239.03 | 106.69 | +0.06 |
| **BL_INV2** | 3 | 672.44 | 327.56 | 88.53 | -0.12 |
| **BL_INV3** | 4 | 597.34 | 402.66 | 75.10 | -0.25 |

* **Average Step Size ($1\text{ LSB}$):** $100.67\text{ mV}$
* **Peak Differential Non-Linearity ($|\text{DNL}|_{\max}$):** $0.31\text{ LSB}$ (due to channel-length modulation and $V_{DS}$ compression as $V_{\text{BL-INV}}$ discharges).

### 4.2. Energy Efficiency & Read Disturb Immunity

| Metric | Value | Notes |
| :--- | :---: | :--- |
| **Supply Voltage ($V_{DD}$)** | $1.0\text{ V}$ | `gpdk090` nominal supply |
| **Compute Pulse Width ($T_{\text{compute}}$)** | $1.0\text{ ns}$ | Wordline Underdrive at $250\text{ mV}$ |
| **Total Bitline Discharge Energy ($E_{\text{BL,total}}$)** | $23.24\text{ fJ}$ | Summed across 4 columns ($C_{BL} = 25\text{ fF}$) |
| **Energy per MAC ($E_{\text{MAC}}$)** | $1.45\text{ fJ/MAC}$ | $16\text{ MACs}$ per compute cycle ($4\times 4$ array) |
| **Core Energy Efficiency** | $1376.9\text{ TOPS/W}$ | Array + bitline capacitor core (excluding peripheral ADC) |
| **Internal Storage Node Disturbance** | $< 20\text{ mV}$ | Verified at `/I0/I31/Q_INV` — zero bit-flip (Read Disturb free) |

![Read Disturb Verification](images/Screenshot%202026-11-22%20102202.png)
*Figure 6: Internal storage nodes (`/I0/I31/Q` and `/I0/I31/Q_INV`) during parallel MAC computation (`/net2`–`/net8` correspond to `BL_INV0`–`BL_INV3`), confirming data retention without bit flipping.*

---

## 5. Future Optimization Directions

1. **Linearity Enhancement ($\text{DNL} < 0.1\text{ LSB}$):** Transition from resistive bitline discharge to **charge-sharing capacitive coupling** or constant-current saturation discharge to eliminate step compression on larger arrays ($8\times 8$, $16\times 16$).
2. **Decoupled Read Port (8T/10T SRAM):** Separate the compute discharge path from the storage nodes to eliminate read disturb completely, enabling full $1.0\text{ V}$ wordline activation and sub-300 ps compute latency.
3. **Differential Signed MAC (XNOR):** Utilize both `BL` and `BL_INV` simultaneously for signed weights ($w \in \{-1, +1\}$) with differential sense amplification.
