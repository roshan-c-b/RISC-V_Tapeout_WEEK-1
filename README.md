# RISC-V_Tapeout_WEEK-1

<details>
<summary>Section 1 — Introduction</summary>

### What is "Design"

`Design` refers to the Verilog source files that implement the required functionality and meet the specifications. These are the RTL modules you write and verify.

### What is a Testbench

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

</details>

---

# Instructions / Setup

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

# Example: simulate `good_mux`

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

---

# Files to inspect

* `good_mux.v`
  The Verilog source implementing the multiplexer logic.

* `tb_good_mux.v`
  The testbench. It will instantiate `good_mux`, apply test vectors, and request a VCD dump.

When you open these files look for the following elements:

* `module` declaration and input/output ports in `good_mux.v`.
* `initial` and `always` blocks in `tb_good_mux.v` that generate clocks, drive inputs, and call `$dumpfile` / `$dumpvars`.

# Basic MUX working logic (reference)

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

---

# Final notes

* Follow the exact commands shown when compiling and running the examples.
* If a VCD does not appear after running `./a.out`, open the testbench and confirm `$dumpfile("tb_good_mux.vcd")` and `$dumpvars` are present.
* If you want the README adjusted or expanded, tell me which section to extend.
