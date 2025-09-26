# RISC-V_Tapeout_WEEK-1

<details>
<summary>Section 1 — Verilog rtl design and synthesis</summary>

### Introduction

`Design` refers to the Verilog source files that implement the required functionality and meet the specifications. These are the RTL modules you write and verify.

A **testbench** is a Verilog file that instantiates the design under test (DUT), drives stimulus (vectors and clocks), and checks outputs. It does not synthesize. Its purpose is to validate functional behaviour through simulation.

### How a Basic Simulator Works

1. The simulator compiles Verilog design files and testbench files into an executable simulation (compiler step).
2. Running the simulator executable executes the testbench which applies stimulus to the DUT.
3. The testbench or simulator records signal value changes in a VCD (Value Change Dump) file.
4. A waveform viewer (gtkwave) loads the VCD file and displays signal transitions for analysis.

Example flow:

```text
Design + Testbench  --> iverilog (compile) --> ./a.out (run) --> tb_good_mux.vcd (VCD) --> gtkwave (view)
```

### Block diagram

![iverilog simulation flow image](/mnt/data/95d03aa0-8262-4e18-bdbf-36d794e7a706.png)

---

## Setup / Instructions

1. Open a terminal and choose or create the directory where you want the repository.

```bash
# example: make and enter workspace
mkdir -p ~/vsd/vlsi
cd ~/vsd/vlsi
```

2. Clone the repository (this creates the `sky130RTLDesignAndSynthesisWorkshop` directory):

```bash
git clone https://github.com/kunalg123/sky130RTLDesignAndSynthesisWorkshop.git
```

3. Repository layout notes (relevant to this workshop)

* The folder `verilog_model` under `my_lib` contains the Verilog design files and their testbenches.
* The example design to use is `good_mux.v` located in `verilog_models`.

4. Simulation toolchain used in examples

* `iverilog` for compilation
* `gtkwave` for waveform viewing

### Example: simulate `good_mux`

Change into the directory containing the design and testbench. Then run the exact commands shown below.

```bash
# compile design and testbench
iverilog good_mux.v tb_good_mux.v

# run the produced simulation executable
./a.out

# open the produced VCD in gtkwave
gtkwave tb_good_mux.vcd
```


### Files to inspect

* `good_mux.v`
  The Verilog source implementing the multiplexer logic.

* `tb_good_mux.v`
  The testbench. It will instantiate `good_mux`, apply test vectors, and request a VCD dump.

When you open these files look for the following elements:

* `module` declaration and input/output ports in `good_mux.v`.
* `initial` and `always` blocks in `tb_good_mux.v` that generate clocks, drive inputs, and call `$dumpfile` / `$dumpvars`.

<img width="661" height="646" alt="Image" src="https://github.com/user-attachments/assets/53e89da0-fa5a-43b5-91cb-8994ba96d3f7" />

### Basic MUX working logic (reference)

This explains the operation used by the example mux. The example uses two select lines `sel1` and `sel0` to pick one of four inputs.

| sel1 | sel0 | output = selected input |
| ---- | ---- | ----------------------- |
| 0    | 0    | input0                  |
| 0    | 1    | input1                  |
| 1    | 0    | input2                  |
| 1    | 1    | input3                  |


The testbench toggles `sel1` and `sel0` and drives `in0..in3` to verify that `out` follows the selected input. The waveform viewer shows `sel1`, `sel0`, inputs and output transitions.

---

## Yosys and Logic Synthesis (integrated)

### What is RTL design?

RTL (Register Transfer Level) design is writing hardware behavior in Verilog or VHDL at the level of registers, combinational logic and the data transfer between them. RTL describes what hardware must do each clock cycle. It is the input to synthesis.

### What is a netlist?

A netlist is a structural representation of the design after synthesis. It lists standard cells (gates), their interconnections and instances. Netlists are the input to place-and-route and downstream physical tools.

### What is a `.lib` (Liberty) file?

A Liberty file describes the standard-cell library used by synthesis and timing tools. It contains cell names, timing arcs, area, drive strength options and power models. The synthesizer maps RTL operators to cells available in the `.lib` you provide.

#### Why different flavours of gate?

Standard libraries provide multiple flavours of the same logical cell. Flavours differ by drive strength, area and delay. Use cases:

* Fast cells: lower intrinsic delay. They reduce combinational path delay and help meet setup timing.
* Slow cells: higher intrinsic delay. They help meet hold timing and reduce leakage and area.

Timing relevance (setup):

```
Tclk >= Tcq(A) + Tcombi + Tsetup(B) + Tclock_skew
```

To meet the required clock period you must reduce the right-hand side. One lever is to use faster cells to make `Tcombi` smaller.

Timing relevance (hold):

```
Tcq(A) + Tcombi - Tclock_skew >= Thold(B)
```

If combinational delay is too small the inequality can fail causing a hold violation. To avoid hold violations you may need slower cells or intentional delay elements on short paths.

#### Why both fast and slow cells are needed

* Fast cells reduce `Tcombi`. This improves maximum clock frequency. But fast cells have larger area and higher dynamic power because they source/sink more current.
* Slow cells increase `Tcombi`. They reduce power and area but can help prevent hold violations or be used on non-critical paths.
* Synthesis must pick cells to balance timing, area and power. You guide it with constraints.

### Cell sizing, load and delay

* A gate's delay depends on the load capacitance it must drive. More load increases delay.
* To reduce delay the cell must drive larger currents. That requires larger transistor widths.
* Larger transistors mean larger area and higher power. There is a tradeoff between speed, area and power.

### Guiding the synthesizer

Supply the correct Liberty file and timing constraints. Typical controls:

* Supply `.lib` (liberty) with available cells and flavours.
* Provide timing constraints in an SDC file: clock definitions, input/output delays, false paths, multi-cycle paths.
* Set area/power constraints if the synthesizer supports them.
  These inputs tell the tool which flavours to prefer when mapping RTL to gates.

### Yosys workflow and commands (example)

Start Yosys in a shell by running:

```bash
yosys
```

Inside Yosys run these commands exactly as shown:

```text
# read the Liberty library
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# read the RTL Verilog
read_verilog good_mux.v
# you should see: "successfully finished verilog frontend"

# run generic synthesis and set the top module
synth -top good_mux

# run ABC to map RTL to library gates
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# inspect the synthesized netlist graphically
show

# write out the gate-level netlist
write_verilog good_mux_netlist.v
```
Notes on synth -top good_mux and ordering

*`synth` runs Yosys's generic synthesis passes. It identifies registers and combinational logic, performs constant propagation, flattens where appropriate, and prepares a generic gate-level netlist.
*`top good_mux` sets the top-level module explicitly. Use it when the RTL has multiple modules and you want to synthesize a specific top.
*Run `synth` before `abc`. `synth` produces the internal representation `abc` expects. 

<img width="303" height="325" alt="Image" src="https://github.com/user-attachments/assets/f6774d5b-6b3d-4fa7-be57-ed27589e7c22" />

Notes on `abc` step

* `abc` performs logic optimization and technology mapping using the provided Liberty file.
* The synthesized netlist will use cells available in the provided library.
* Example inferred output from `abc` after synthesizing `good_mux`:

<img width="262" height="76" alt="Image" src="https://github.com/user-attachments/assets/51ce5e27-1782-4e63-a33e-465b377f033a" />

### Inspecting and writing netlists

After synthesis you can export and inspect the generated netlist. Typical commands and purpose:

```bash
# write full netlist with attributes and comments
write_verilog good_mux_netlist.v

# open in editor to inspect
gvim good_mux_netlist.v
```

`write_verilog good_mux_netlist.v` writes a gate-level Verilog netlist. That netlist includes Yosys-specific attributes. Attributes annotate synthesis choices, mapping details, or tool-specific metadata. The file can be large and include comments and attributes that clutter manual inspection.

<img width="826" height="435" alt="Image" src="https://github.com/user-attachments/assets/b2c1867c-882a-46ec-8bc4-35472972fa15" />

To generate a cleaner, minimal netlist without synthesis attributes use:

```bash
write_verilog -noattr good_mux_netlist.v
gvim good_mux_netlist.v
```
<img width="374" height="287" alt="Image" src="https://github.com/user-attachments/assets/f6f85034-0e8c-4550-8e00-0a98e23688b8" />

`-noattr` removes Yosys attributes and many synthesis comments. Resulting file is smaller and easier to read. Use `-noattr` when you want a compact, human-readable netlist for review or for importing into other tools that do not expect attributes.

What to look for in the netlist

* `module` declaration for the top-level and instantiated sub-modules.
* Instantiated standard cells. Example: `sky130_fd_sc_hd__mux2_1` or `sky130_fd_sc_hd__and2_1`.
* Port connections and net names. Follow how net names created from RTL map to cell ports.
* Parameters and constant nets.

Visual differences between the two netlists

* Full netlist: contains `(* ... *)` attributes on modules, instances and nets. Contains comments like `// attribute ...` and tool metadata.
* `-noattr` netlist: attributes removed. Only structural instances and connections remain.

Use the editor screenshots to show the difference. Paste both screenshots into the repo and I will add concise captions describing what changed and why the `-noattr` output is useful.

</details>

<summary>Section 2 — Timing libs, hierarchical vs Flat synthesis and efficient flop coding styles</summary>

#  Timing libraries, `.lib` anatomy and cell comparison

### Quick start

Open the library file:

```bash
gvim ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

Extract a specific cell block:

```bash
# example: show the and2_4 block
sed -n '/cell ("sky130_fd_sc_hd__and2_4")/,/}/p' ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

---
<img width="289" height="321" alt="Image" src="https://github.com/user-attachments/assets/e40b4ec7-268f-46aa-880b-1fabb2d31a2e" />

### Library name parsed

`sky130_fd_sc_hd__tt_025C_1v80`

* `sky130` = SkyWater 130 nm process node.
* `fd` = foundry distribution / foundry-provided.
* `sc` = standard-cell library.
* `hd` = high-density cell family (area-optimized cell height/pitch).
* `tt` = typical PVT corner. Alternatives: `ff` (fast), `ss` (slow).
* `025C` = temperature (25 °C). Other corners: `-40C`, `125C`.
* `1v80` = nominal voltage 1.8 V.

---

### What a `.lib` contains (essentials)

* **Cells**: name, pins, `area`, `cell_footprint`.
* **Timing data**: `timing` entries, delay tables, output transition vs load (`table_lookup` or other model).
* **Power data**: `leakage_power` (per input condition), `cell_leakage_power`.
* **Units & operating conditions**: `time_unit`, `voltage_unit`, `current_unit`, `capacitive_load_unit`, `operating_conditions { voltage process temperature }`.
* **Delay model**: often `table_lookup` for CMOS libs (delay is read from characterization tables).

Why it matters: synthesis maps RTL operators to the cells described here. Delay and power numbers are valid only at the listed PVT corner.

---

### PVT (Process / Voltage / Temperature)

* **Process**: fabrication speed variation (fast/typical/slow). Affects transistor drive.
* **Voltage**: supply scaling directly affects delay and power.
* **Temperature**: affects carrier mobility and leakage. Higher T -> higher delay and leakage in general.

---

### Practical cell comparison (AND2 flavours)
<img width="227" height="194" alt="Image" src="https://github.com/user-attachments/assets/74f757ac-dd71-48e7-bfca-554ca0e1364c" />
<img width="220" height="189" alt="Image" src="https://github.com/user-attachments/assets/31550251-980f-4786-9b6b-0d6aa24b9387" />
<img width="232" height="191" alt="Image" src="https://github.com/user-attachments/assets/731eb220-a554-452e-9871-977401ddea97" />
Data taken from your `.lib` excerpts.

| Cell   |   Area | cell_leakage_power | leakage/area (approx) |
| ------ | -----: | -----------------: | --------------------: |
| and2_0 | 6.2560 |        0.001921380 |              0.000307 |
| and2_1 | 6.2560 |        0.002665008 |              0.000426 |
| and2_2 | 7.5072 |        0.003369228 |              0.000449 |
| and2_4 | 8.7584 |        0.004546817 |              0.000519 |

Observations

* Area rises across flavours and absolute leakage rises accordingly.
* Leakage density (leakage/area) also increases with bigger flavours here.
* `and2_0` vs `and2_1`: same area but different leakage. That implies internal transistor sizing or Vth/topology differences even within the same footprint. Do not assume identical timing from identical footprint alone.

Conclusion

* Larger/higher-index flavours will generally have lower delay and higher area/power. Use them for timing-critical nets.
* Smaller flavours save area and power and may help hold timing (or at least avoid making short paths even faster).
* Always verify **timing tables** (`timing` / `cell` block) for real delay numbers versus load and input transition. Leakage alone is not a timing indicator.

---

### How to inspect timing entries (commands)

Show timing table for a cell:

```bash
sed -n '/cell ("sky130_fd_sc_hd__and2_4")/,/timing/p' ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
sed -n '/timing/,/}/p'  ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib | sed -n '1,200p'
```

Search all `area` and `cell_leakage_power` lines quickly:

```bash
grep -E 'cell \(|area :|cell_leakage_power' ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```


# Hierarchical vs Flat Synthesis 

### Example RTL (multiple_modules.v)

```verilog
module sub_module2 (input a, input b, output y);
  assign y = a | b;
endmodule

module sub_module1 (input a, input b, output y);
  assign y = a & b;
endmodule

module multiple_modules (input a, input b, input c, output y);
  wire net1;
  sub_module1 u1(.a(a), .b(b), .y(net1));  // net1 = a & b
  sub_module2 u2(.a(net1), .b(c), .y(y));  // y = net1 | c => y = (a & b) | c
endmodule
```

### Quick block diagram

```
   a ----\        /-- net1 --\
           AND(u1)           OR(u2) --> y
   b ----/        \--       /        \
                        c --/          
```

* u1 implements `net1 = a & b`.
* u2 implements `y = net1 | c`.

---

## Yosys synthesis steps (hierarchical)

Start Yosys and run:

```text
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog multiple_modules.v
synth -top multiple_modules
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show multiple_modules
write_verilog -noattr multiple_modules_hier.v
```
<img width="302" height="320" alt="Image" src="https://github.com/user-attachments/assets/1c38b8cb-4d28-4d65-9f91-002888d0cec6" />

Open the generated netlist:

```bash
gvim multiple_modules_hier.v
```
<img width="431" height="541" alt="Image" src="https://github.com/user-attachments/assets/33ccf144-8b8e-49cb-80be-5a743481ef02" />

**Outcome:** hierarchy preserved in the netlist. Submodules appear as instantiated modules or as named cells depending on mapping.

### Why an OR became NAND + inverters in the netlist

Synthesis used De Morgan equivalence to implement `a | b` as `~( ~a & ~b )`. The resulting mapped implementation is a NAND with input inverters.

**Why prefer NAND+inverters over direct OR/NOR?**

* CMOS NAND pull-up network uses parallel PMOS and series NMOS. CMOS NOR uses series PMOS in the pull-up.
* PMOS transistors have lower mobility. Series PMOS stacking hurts pull-up strength and increases delay.
* Implementing OR via NAND+inverters avoids stacked PMOS in the pull-up path. That often reduces area or delay when using standard-cell libraries.
* The library may lack an optimized OR cell but has strong NAND/inverter cells. `abc` chooses the cheapest/fastest mapping based on availability and timing.

Related concept: **logical effort**. To reduce delay you can increase transistor width. Stacked devices change logical effort and required sizing. Synthesis trades sizing, cell choice and mapping to meet constraints.

---

## Flat synthesis

In Yosys run a flatten pass (after generic synth or before `abc`):

```text
flatten
write_verilog -noattr multiple_modules_flat.v
```

Open the flat netlist:

```bash
gvim multiple_modules_flat.v
```
<img width="446" height="458" alt="Image" src="https://github.com/user-attachments/assets/f866e854-8823-4e36-adb5-65fb3a2fef1c" />

**Outcome:** modules are flattened. Submodule boundaries removed. The netlist will show primitive gates (AND/OR) directly under `multiple_modules`.

**Difference:**

* **Hierarchical**: keeps module instantiations. Easier incremental flows and protects boundaries. May limit cross-module optimizations.
* **Flat**: exposes everything to global optimization. Can yield better area/timing at the cost of runtime and loss of module identity.

---

## Module-level synthesis (`synth -top <module>`)

```text
synth -top sub_module1
write_verilog -noattr sub_module1_netlist.v
```

**Why use it?**

* If a top-level module contains many identical instances (e.g., `n` adders), synthesize one instance then replicate its mapped netlist. This saves runtime and ensures consistent mapping.
* Useful for divide-and-conquer on very large designs. Synthesize critical blocks independently, optimize them, then integrate at top-level.

**When to prefer module-level synthesis**

1. Multiple identical instantiations of a component.
2. Need to iterate on a block without resynthesizing the whole chip.
3. Floorplanning or P&R constraints that require fixed, repeatable block shapes and hierarchies.

**Flow**

1. Synthesize modules hierarchically by default.
2. Identify timing-critical blocks and re-synthesize them flat or individually (module-level) for aggressive optimization.
3. Run `abc` with the target Liberty and perform STA on the netlist to validate timing and hold.
4. If area or timing still unsatisfactory, selectively flatten problematic modules and re-run mapping.

---

</details>



