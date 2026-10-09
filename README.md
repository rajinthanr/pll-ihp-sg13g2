# PLL_IHP

A charge-pump phase-locked loop designed in the IHP SG13G2 130 nm BiCMOS open PDK. The schematics are drawn in xschem and simulated in ngspice. The feedback divider is a behavioural Verilog-A model compiled to OSDI with OpenVAF, so the closed loop simulates in reasonable time.

This is a work in progress (see [Status](#status)).

## Architecture

```
          +-------+   UP   +--------------+       +--------------+  CTRL  +--------+
F_REF --->|       |------->| Charge pump  |------>| Loop filter  |------->| LC VCO |---+---> OUT
100 MHz   |  PFD  |   DN   | (current-    |       | R + C, then  |        | (ext.) |   |
     +--->|       |------->|  mirror)     |       | NMOS source  |        +--------+   |
     |    +-------+        +--------------+       | follower     |                     |
     |                                            +--------------+                     |
     |                    +----------------------------+                               |
     +--------------------| Divider, /24 (Verilog-A)   |<------------------------------+
            F_VCO         +----------------------------+
```

- **PFD** (`src/PFD_std.sch`): tri-state phase-frequency detector from SG13G2 standard cells: two `sg13g2_dfrbp_2` flip-flops with D tied high, clocked by `F_REF` and `F_VCO`, reset together by a `sg13g2_nand2_2` of their outputs. `UP` leaves through a `sg13g2_inv_2`, so it is active low for the PMOS switch in the charge pump.
- **Charge pump** (`src/CP.sch`): `sg13_lv_nmos` / `sg13_lv_pmos` transistors. A reference branch (diode-connected PMOS and an `rsil` resistor to the `Ibias` input) sets `Vbp` and, through a replica branch, `Vbn`. The output branch is a PMOS current source switched by `UP` and an NMOS current sink switched by `DN`, with series dummy switches in the bias branches to match them. Device sizes are parameters (`CP_P_W`, `CP_N_W`, `CP_*_L`, `CP_*_M`) set from the testbench.
- **Loop filter and buffer** (inside `src/CP.sch`): the pump output goes through an `rsil` resistor to a `cap_cpara` capacitor of value `C_CP`. That node drives an NMOS source follower (W = 100 µm, 20 fingers, `rsil` load) whose output is `CTRL`, the VCO control voltage.
- **VCO**: an LC VCO (`LC_VCO.sym`) with a 4 nH inductor model, taken from a separate design at `/foss/designs/frac-n-pll-vco-mixdes_2026/`. **It is not in this repository**, so `tb_VCO` and `tb_COMB` need that design installed at that path. Its output is squared up by a `sg13g2_inv_2` before the divider.
- **Divider** (`src/freq_div.va`, symbol `src/freq_div.sym`): Verilog-A counter with parameters `ratio` (default 24), `vth` (0.8 V), `vh` (1.2 V), `vl`, `tr`. OpenVAF does not support event statements such as `@(cross)`, so the module detects rising edges and stores its count on internal nodes driven by small RC equations, and it declares its own natures and discipline instead of including `disciplines.vams`. `src/freq_div.osdi` is the compiled model (x86-64 Linux).
- **Standard-cell divider** (`src/Freq_Div_std.sch`): a ripple counter of four `sg13g2_dfrbp` toggle flip-flops (divide by 16). It was used in an earlier combined simulation that ran too slowly and is not used by the current testbenches.

With a 100 MHz reference and the default ratio of 24, the loop targets 2.4 GHz.

## Status

From the commit history and testbenches:

- The PFD and charge pump work in simulation (`tb_PFD_std`, `tb_CP`).
- The Verilog-A divider works in ngspice (`tb_Frequency_Divider`).
- The full loop (`tb_COMB`) simulates, but the VCO control voltage oscillates instead of settling. Loop-filter and stability work is still to do.
- No layout yet; schematic level only, typical corners.

## Tools and PDK

- [xschem](https://xschem.sourceforge.io/) 3.4.8 (schematics, testbench launchers)
- [ngspice](https://ngspice.sourceforge.io/) with OSDI support (`pre_osdi`)
- [OpenVAF](https://openvaf.semimod.de/) (Verilog-A to OSDI)
- [IHP SG13G2 open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK): `sg13_lv_nmos/pmos`, `rsil`, `cap_cpara` and the `sg13g2_stdcell` library, with `mos_tt`, `res_typ`, `cap_typ_stat` corners
- The hard-coded paths (`/foss/pdks/ihp-sg13g2`, `/foss/designs/...`) follow the [IIC-OSIC-TOOLS](https://github.com/iic-jku/IIC-OSIC-TOOLS) container layout.

## How to simulate

1. Inside IIC-OSIC-TOOLS (or an equivalent setup with the SG13G2 PDK at `/foss/pdks/ihp-sg13g2`), clone the repository to the path the schematics expect:
   ```sh
   git clone https://github.com/rajinthanr/PLL_IHP.git /foss/designs/PLL_IHP_PDK
   ```
   Symbols and the OSDI file are referenced as `/foss/designs/PLL_IHP_PDK/src/...`; to use another location, edit those paths in the `tb/*.sch` files.
2. If `src/freq_div.osdi` does not load on your machine, or after editing the Verilog-A, rebuild it:
   ```sh
   cd /foss/designs/PLL_IHP_PDK/src && openvaf freq_div.va    # writes freq_div.osdi
   ```
   The testbenches load it with `pre_osdi /foss/designs/PLL_IHP_PDK/src/freq_div.osdi` and `.model freq_div freq_div`; the instance is netlisted as `n1 <out> <in> freq_div`.
3. Open a testbench with xschem (using the SG13G2 xschemrc) and click the **SimulateNGSPICE** launcher, then **load waves**:

| Testbench | What it runs |
|---|---|
| `tb/tb_PFD_std.sch` | 100 MHz `F_REF` and `F_VCO` 2 ns apart; 10 transient runs with Gaussian-varied VDD (1.2 V ± 0.05 V) and temperature |
| `tb/tb_CP.sch` | PFD + charge pump, 6 ns phase offset, 100 ns transient |
| `tb/tb_Frequency_Divider.sch` | 2.5 GHz pulse into the Verilog-A divider, 200 ns transient and FFT of input and output |
| `tb/tb_VCO.sch` | LC VCO with `VCTRL` ramped 0.3 V to 1.0 V; a `.control` script computes instantaneous frequency from rising crossings |
| `tb/tb_COMB.sch` | Closed loop: 100 MHz reference, PFD, charge pump/filter (`C_CP` = 700 pF), VCO, divider; 2 µs transient |

## Repository layout

```
src/
  PFD_std.sch/.sym       phase-frequency detector (standard cells)
  CP.sch/.sym            charge pump, loop filter and CTRL buffer
  freq_div.va            Verilog-A divide-by-N model (OpenVAF compatible)
  freq_div.osdi          compiled model for ngspice
  freq_div.sym           xschem symbol for the Verilog-A divider
  Freq_Div_std.sch/.sym  standard-cell /16 ripple divider (not used by the testbenches)
tb/
  tb_PFD_std.sch, tb_CP.sch, tb_Frequency_Divider.sch, tb_VCO.sch, tb_COMB.sch
  tb_Frequency_Divider.sym
LICENSE
```

## Licence

MIT, copyright (c) 2026 Rajinthan Rameshkumar. See [LICENSE](LICENSE).
