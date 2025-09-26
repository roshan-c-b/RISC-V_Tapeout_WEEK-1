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

### Setup / Instructions

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

**Do not modify these commands.** They are the canonical steps used in the examples.

### Files to inspect

* `good_mux.v`
  The Verilog source implementing the multiplexer logic.

* `tb_good_mux.v`
  The testbench. It will instantiate `good_mux`, apply test vectors, and request a VCD dump.

When you open these files look for the following elements:

* `module` declaration and input/output ports in `good_mux.v`.
* `initial` and `always` blocks in `tb_good_mux.v` that generate clocks, drive inputs, and call `$dumpfile` / `$dumpvars`.

### Basic MUX working logic (reference)

This explains the operation used by the example mux. The example uses two select lines `sel1` and `sel0` to pick one of four inputs.

| sel1 | sel0 | output = selected input |
| ---- | ---- | ----------------------- |
| 0    | 0    | input0                  |
| 0    | 1    | input1                  |
| 1    | 0    | input2                  |
| 1    | 1    | input3                  |

Verilog behavioural equivalent (conceptual):

```verilog
always @(*) begin
  case ({sel1, sel0})
    2'b00: out = in0;
    2'b01: out = in1;
    2'b10: out = in2;
    2'b11: out = in3;
  endcase
end
```

The testbench toggles `sel1` and `sel0` and drives `in0..in3` to verify that `out` follows the selected input. The waveform viewer shows `sel1`, `sel0`, inputs and output transitions.

### Final notes

* Follow the exact commands shown when compiling and running the examples.
* If a VCD does not appear after running `./a.out`, open the testbench and confirm `$dumpfile("tb_good_mux.vcd")` and `$dumpvars` are present.
* This README is section-controlled. You will add sections 2-5 later. For now everything above is inside Section 1.

</details>

<details>
<summary>Section 2 — Introduction to Yosys and Logic Synthesis</summary>

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

# run ABC to map RTL to library gates
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# inspect the synthesized netlist graphically
show

# write out the gate-level netlist
write_verilog good_mux_netlist.v
```

Notes on `abc` step

* `abc` performs logic optimization and technology mapping using the provided Liberty file.
* The synthesized netlist will use cells available in the provided library.
* Example inferred output from `abc` after synthesizing `good_mux`:

```
ABC RESULTS:   sky130_fd_sc_hd__mux2_1 cells:        1
ABC RESULTS:        internal signals:        0
ABC RESULTS:           input signals:        3
ABC RESULTS:          output signals:        1
```

After `show` you will get a graphical depiction of the netlist. Paste the image into the repo or into this README and I will add a short write-up for that image.

### Inspecting and writing netlists

After synthesis you can export and inspect the generated netlist. Typical commands and purpose:

```bash
# write full netlist with attributes and comments
write_verilog good_mux_netlist.v

# open in editor to inspect
gvim good_mux_netlist.v
```

`write_verilog good_mux_netlist.v` writes a gate-level Verilog netlist. That netlist includes Yosys-specific attributes. Attributes annotate synthesis choices, mapping details, or tool-specific metadata. The file can be large and include comments and attributes that clutter manual inspection.

To generate a cleaner, minimal netlist without synthesis attributes use:

```bash
write_verilog -noattr good_mux_netlist.v
gvim good_mux_netlist.v
```

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
