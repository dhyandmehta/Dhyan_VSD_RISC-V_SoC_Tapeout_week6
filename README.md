# RISC-V Based SoC Implementation with Custom Standard Cell Integration using OpenLANE & Sky130 PDK

![Digital\_VLSI\_SoC\_Design](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/5b8bdb5f-95c7-41c0-b809-711b2b8ad171)

![OS: linux](https://img.shields.io/badge/OS-linux-orange)
![EDA Tools](https://img.shields.io/badge/EDA%20Tools-OpenLANE--Flow%2C_Yosys%2C_abc%2C_OpenROAD%2C_TritonRoute%2C_OpenSTA%2C_magic%2C_netgen%2C_ngspice-navy)
![Langs](https://img.shields.io/badge/Languages-Verilog%2C_Bash%2C_TCL-crimson)
![GitHub last commit](https://img.shields.io/github/last-commit/fayizferosh/soc-design-and-planning-nasscom-vsd)
![Repo size](https://img.shields.io/github/repo-size/fayizferosh/soc-design-and-planning-nasscom-vsd)

> *Complete RTL → GDSII implementation using open-source EDA and SkyWater 130 nm PDK.*
> This document is written in the same tone, structure and detail as the VSD–NASSCOM workshop reports — it is intended as a workshop/lab report style README for a RISC-V SoC project that integrates a custom standard cell (inverter) into an OpenLANE flow.

---

## Table of Contents

1. [Overview & Goals](#overview--goals)
2. [Environment & Prerequisites](#environment--prerequisites)
3. [Section 1 — Inception, OpenLANE & Sky130 PDK](#section-1---inception-openlane--sky130-pdk)
4. [Section 2 — Floorplanning & Placement](#section-2---floorplanning--placement)
5. [Section 3 — Custom Standard Cell Design (Magic) & ngspice Characterization](#section-3---custom-standard-cell-design-magic--ngspice-characterization)
6. [Section 4 — Integrating Custom Cell into OpenLANE Flow](#section-4---integrating-custom-cell-into-openlane-flow)
7. [Section 5 — Pre/Post-CTS STA, Clock Tree, ECOs, and Sign-off](#section-5---prepost-cts-sta-clock-tree-ecos-and-sign-off)
8. [Run Folders, Logs & Reproducibility](#run-folders-logs--reproducibility)
9. [Results & Metrics](#results--metrics)
10. [Acknowledgements & References](#acknowledgements--references)
11. [License](#license)

---

## Overview & Goals

This repository documents a complete hands-on exercise demonstrating an open-source RTL → GDSII flow for a RISC-V core (picorv32a) using OpenLANE and the SkyWater 130 nm PDK. The project objectives:

* Run synthesis (Yosys / OpenLane) for picorv32a and collect synthesis statistics.
* Floorplan and place the design using OpenROAD / OpenLANE scripts.
* Design, extract and characterize a custom inverter standard cell in Magic and ngspice.
* Integrate custom cell `.lef` & `.lib` into `picorv32a` OpenLANE flow, re-run synthesis and PnR.
* Perform pre- and post-CTS STA (OpenSTA / OpenROAD), apply timing ECOs and finalize layout sign-off (DRC/LVS/STA).

Everything below is written as a workshop lab report: theory context, step-by-step commands, screenshots, calculations, and links to run directories.

---

## Environment & Prerequisites

Minimum environment used for the lab (Ubuntu / Linux oriented):

* Docker with the `efabless/openlane:v0.21` image (or newer).
* OpenLANE (local copy in `~/Desktop/work/tools/openlane_working_dir/openlane`).
* SkyWater PDK (sky130A) installed and referenced via `$PDK_ROOT`.
* Tools installed inside OpenLane docker: Magic, ngspice, OpenROAD, TritonRoute, OpenSTA, netgen, yosys, abc.
* Shell: Bash. TCL used inside `./flow.tcl -interactive` sessions.

Key local paths used in the workshop (examples):

```
~/Desktop/work/tools/openlane_working_dir/openlane
~/Desktop/work/tools/openlane_working_dir/pdks/sky130A
~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a
~/Desktop/work/tools/openlane_working_dir/openlane/vsdstdcelldesign
```

> NOTE: environment variables (e.g., `PDK_ROOT`, `OPENLANE_ROOT`) must be setup correctly before invoking docker/openlane.

---

## Section 1 — Inception, OpenLANE & Sky130 PDK (14/03/2024 – 15/03/2024)

### Theory (concise refresher)

* **Package vs Die vs Chip:** package encloses die; die consists of core + pads; pads connect to board via package.
* **ISA → Compiler → Assembler → RTL:** high-level code compiles to RISC-V instructions; assembler creates machine code which maps to RTL implementation; RTL synthesized to gate-level netlist → PnR → GDSII.
* **OpenLANE Flow:** automates RTL → GDS II using open tools and Skywater PDK.

![flow](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/533f58ee-4524-4a18-abb5-36b4d6a56b1f)

### Implementation — Synthesis

**Commands (host machine → openlane docker):**

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
# launch container (or alias long docker command to 'docker')
docker
# inside docker
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
run_synthesis
exit
exit
```

Screenshots of the synthesis run:

![synth1](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/d19f6d0f-16f8-4e79-aa5a-f2a34b9fb203)
![synth2](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/5e03c8ca-8c7f-4579-a7bc-10161007910e)

**Flop ratio calculation example (from synthesis report):**

```math
Flop\ Ratio = \frac{Number\ of\ DFFs}{Total\ Cells} = \frac{1613}{14876} = 0.108429685
Percentage\ of\ DFFs = 10.84296854\%
```

Run folder for the synthesis stage:
`Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/15-03_15-51`

---

## Section 2 — Floorplanning & Placement (16/03/2024 – 17/03/2024)

### Theory

* Floorplan: define core boundary, IO placement, power rails and keepout regions.
* Placement: global placement then legalization/detailed placement to remove overlaps and align to tracks.
* Power planning: use upper metal layers for power distribution to reduce IR drop.

Key conceptual images:

![floorplan image](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/38ecd866-ac83-42c7-83ba-2ba995f9ba4e)
![clock tree image](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/6db284d4-065f-450a-9200-a6e6cbfd7fbb)

### Implementation — Floorplan & Placement Commands

Inside OpenLANE interactive session:

```tcl
prep -design picorv32a
run_floorplan
run_placement
# or use finer-grained commands if run_floorplan fails:
init_floorplan
place_io
tap_decap_or
run_placement
```

Screenshots:

![floorplan run](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/7deda325-2ae8-4e98-aa71-7a54f5c34fcb)
![placement magic view](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/e703ef0b-3968-4132-a9c7-05b53f50b214)

**Die size computation (example from floorplan DEF):**

```math
1000\ units = 1\ micron
Die\ Width\ (units) = 660685 -> 660.685\ \mu m
Die\ Height\ (units) = 671405 -> 671.405\ \mu m
Area = 660.685 * 671.405 = 443587.212425\ \mu m^2
```

Run folder for floorplan/placement:
`Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/17-03_12-06`

---

## Section 3 — Custom Standard Cell Design (Magic) & ngspice Characterization (18/03/2024 – 21/03/2024)

> This section follows a lab procedure: clone a custom inverter layout repo, fix DRC rules in tech file when needed, extract to SPICE, and run post-layout electrical characterization.

### High-level steps

1. Clone `vsdstdcelldesign` (custom inverter repo).
2. Load inverter layout in Magic; verify PMOS/NMOS, nets, ports.
3. Extract parasitics and convert to SPICE.
4. Edit spice netlist to a simulator-friendly format.
5. Run ngspice and measure rise/fall transition and delays.
6. Fix any techfile DRC problems (poly.9, difftap.2, nwell.4) and re-check DRC.

**Commands:**

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
git clone https://github.com/nickson-jose/vsdstdcelldesign
cd vsdstdcelldesign
cp /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech .
magic -T sky130A.tech sky130_inv.mag &
# In tkcon (Magic)
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

Screenshots: layout & extraction views

![inv layout](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/6eae887c-ebc6-4771-8bcc-e4edaf9947d9)
![extraction tkcon](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/831b0be9-3c02-4bbb-800e-6f1c3dc1ba1a)
![spice file](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/2c645f55-c4d5-4007-8b9c-73ba8a8e5bcb)

### ngspice simulation & measurement

```bash
ngspice sky130_inv.spice
# then within ngspice:
plot y vs time a
```

Plot and screenshots:

![ngspice plot](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/dd14a5d5-ffd9-4ad8-a871-e32af61362a3)
![plot zoom 20%/80% markers](https://github.com/fayizferosh/risc-v-myth-report/assets/63997454/261c420f-219f-4c26-ae32-6c0db82a722e) <!-- retains same style / consistent markers -->

**Transition & Delay calculations (worked example):**

* Definitions:

  * 20% of 3.3 V = 0.66 V
  * 50% = 1.65 V
  * 80% = 2.64 V

* Rise transition:

```math
t_{rise} = t_{80\%} - t_{20\%} = 2.24638\ ns - 2.18242\ ns = 0.06396\ ns = 63.96\ ps
```

* Fall transition:

```math
t_{fall} = t_{20\%} - t_{80\%} = 4.0955\ ns - 4.0536\ ns = 0.0419\ ns = 41.9\ ps
```

* Rise cell delay (50% out - 50% in):

```math
Delay_{rise} = 2.21144\ ns - 2.15008\ ns = 0.06136\ ns = 61.36\ ps
```

* Fall cell delay:

```math
Delay_{fall} = 4.07\ ns - 4.05\ ns = 0.02\ ns = 20\ ps
```

(See screenshots for the exact waveform cursors and extracted times.)

### Fixing techfile DRC issues

Some of the Magic techfile rules (old sky130A.tech examples) may be incorrect for the current PDK and must be patched for strict DRC correctness. Example issues:

* `poly.9` simple rule spacing incorrectly implemented
* `difftap.2` rule missing or too lax
* `nwell.4` complex rule not detecting missing taps

Commands used to test and adjust:

```bash
# Download drc_tests
wget http://opencircuitdesign.com/open_pdks/archive/drc_tests.tgz
tar xfz drc_tests.tgz
cd drc_tests
# open files, edit .magicrc or sky130A.tech
gvim .magicrc
magic -d XR &
# In tkcon:
tech load sky130A.tech
drc style drc(full)
drc check
drc why
```

Screenshots that show before/after DRC fixes and `.magicrc` edits:

![magic .magicrc](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/89b46a0f-63b7-445c-bf2b-e6cda16853c7)
![drc example poly rule screenshot](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/9260cf37-5933-44a1-8362-597183644334)

---

## Section 4 — Integrating Custom Cell into OpenLANE Flow (24/03/2024)

### Theory

To use a custom standard cell in synthesis and PnR, you must provide:

* `*.lef` — physical abstract for PnR
* `*.lib` — timing library (typical/slow/fast variants) for synthesis/STA
* `*.spice` (optional) — for LVS / analog verification

OpenLANE picks up the design libraries via `config.tcl` variables like `LIB_SYNTH`, `LIB_TYPICAL`, and `EXTRA_LEFS`.

### Implementation — Steps & Commands

1. Generate `.lef` from magic:

```tcl
# in magic (tkcon)
lef write
```

2. Copy `.lef` and `.lib` files into the `picorv32a/src/` directory:

```bash
cp sky130_vsdinv.lef ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
cp libs/sky130_fd_sc_hd__* ~/Desktop/work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
```

3. Edit `config.tcl` to include the custom cell libraries:

```tcl
set ::env(LIB_SYNTH) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS) [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

4. Re-run synthesis and PnR with the custom LEF added:

```bash
# host -> docker -> openlane
docker
# inside
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a -tag 24-03_10-03 -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
run_synthesis
run_floorplan
run_placement
```

Screenshots showing merged.lef and synthesis results with the custom inverter macro present:

![merged.lef screenshot](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/55de3fc6-498d-4456-8e79-ae6e175d2ca6)
![synthesis custom inverter present](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/4170c3c5-1a95-4165-9461-03b298cc20ef)

### Dealing with introduced violations & tuning

When you add a custom cell, area may increase and slack can change. The flow parameters to tune include:

* `SYNTH_STRATEGY` (e.g., `DELAY 3`)
* `SYNTH_SIZING` (enable sizing)
* `SYNTH_BUFFERING` (enable/disable)
* `SYNTH_DRIVING_CELL` (choose driving cell)

Example commands to change variables:

```tcl
echo $::env(SYNTH_STRATEGY)
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
prep -design picorv32a -tag 24-03_10-03 -overwrite
run_synthesis
```

Observe the timing vs area tradeoffs and re-run placement to ensure legal placement and acceptable WNS.

---

## Section 5 — Pre/Post-CTS STA, Clock Tree, ECOs, and Sign-off

### Theory

* **Pre-CTS STA** (OpenSTA) on synthesized netlist identifies timing violations before CTS.
* **CTS** (Clock Tree Synthesis) creates clock buffers/insertion to balance arrival times.
* **Post-CTS STA** confirms whether setup/hold violations are fixed or require ECOs.
* **ECOs** (Engineering Change Orders) are minimally intrusive netlist edits (buffer insertion, gate resizing, buffering) to fix timing.
* **Final sign-off**: DRC = 0, LVS matched, STA clean.

### Implementation — Commands

```tcl
# inside OpenLANE interactive session after placement
run_cts
run_routing
run_magic_drc
run_lvs
run_sta
# If violations present:
# run_timing_eco or manual ECO flow
```

Screenshots of CTS and routing:

![cts screenshot](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/0bc13ad3-d800-4681-b39d-8b64c9c9104f)
![routing screenshot](https://github.com/fayizferosh/soc-design-and-planning-nasscom-vsd/assets/63997454/b54ebd4c-127a-441f-829e-6000531e9b8)

**Sample workflow to remove a clock buffer from the `CTS_CLK_BUFFER_LIST`:**

```tcl
# edit config or list variable used by CTS
# In TCL:
set ::env(CTS_CLK_BUFFER_LIST) [lsearch -all $::env(CTS_CLK_BUFFER_LIST) "sky130_fd_sc_hd__clkbuf_1" ;# then modify accordingly]
# Re-run CTS:
run_cts
run_sta
```

If the timing is clean and DRC/LVS succeed:

```tcl
run_magic_gds_export
# final gds file available in results directory
```

---

## Run Folders, Logs & Reproducibility

All important run folders & logs (examples):

* Synthesis run: `openlane/designs/picorv32a/runs/15-03_15-51`
* Floorplan/placement run: `openlane/designs/picorv32a/runs/17-03_12-06`
* Integration run (with custom inverter): `openlane/designs/picorv32a/runs/24-03_10-03`
* vsdstdcelldesign (custom inverter sources & magic files): `openlane/vsdstdcelldesign`
* DRC fixes run: top-level `drc_tests/` (examples from `opencircuitdesign`)

**How to reproduce (short checklist):**

1. Setup `PDK_ROOT` and `OPENLANE_ROOT`.
2. Launch docker with PDK bind mounted.
3. `./flow.tcl -interactive` → `prep -design picorv32a`.
4. Run `run_synthesis` → collect `synthesis_stats` and `synth.log`.
5. `run_floorplan` → check `floorplan.def` and compute die area.
6. `run_placement` → confirm legal placement.
7. Design custom cell in `vsdstdcelldesign`, generate `.lef` & `.lib` and copy to `picorv32a/src`.
8. Update `config.tcl` to include new libs/lefs.
9. `run_synthesis` (with new LEFs) → `run_floorplan` → `run_placement` → `run_cts` → `run_routing`.
10. `run_magic_drc`, `run_lvs`, `run_sta`. If necessary, perform ECOs and iterate.

---

## Results & Metrics (Summary)

| Stage           |                     Tool | Key Output / Metric                                                            |
| --------------- | -----------------------: | :----------------------------------------------------------------------------- |
| Synthesis       |         Yosys / OpenLane | DFF Ratio ≈ 10.84% (1613 / 14876)                                              |
| Floorplan       |        OpenROAD/OpenLane | Die ~660.685 μm × 671.405 μm → 443,587 μm²                                     |
| Custom inverter |          Magic + ngspice | Rise Delay ≈ 61.36 ps; Fall Delay ≈ 20 ps                                      |
| Integration     |                 OpenLane | Custom LEF merged; area increased; timing improved after SYNTH_STRATEGY tuning |
| Sign-off        | Magic / netgen / OpenSTA | DRC/LVS/STA passes after ECOs and techfile fixes (see runs)                    |

### Artifacts produced

* `merged.lef` with custom `sky130_vsdinv` macro
* `sky130_vsdinv.lef` (from magic)
* `sky130_inv.spice` and `ngspice` plots
* Final `gds` (run results) — available in `runs/<timestamp>/results/gds/`

---

## Notes, Tips & Gotchas

* Always keep a backup of the original `sky130A.tech` before modifying. Fixing DRC rules should be done carefully — incorrect rules can hide violations.
* When adding EXTRA_LEFS, ensure the physical abstract names (CELL names) match the synthesized cell names or alias them appropriately.
* If OpenLANE floorplan commands fail, use the underlying sequence (`init_floorplan`, `place_io`, `tap_decap_or`) to debug granularly.
* For timing fixes, prefer minimal ECOs (buffer insertion, resizing) and re-measure WNS and TNS after each change.
* Use `magic -d XR` for a better interactive GUI when debugging LVS/DRC.
