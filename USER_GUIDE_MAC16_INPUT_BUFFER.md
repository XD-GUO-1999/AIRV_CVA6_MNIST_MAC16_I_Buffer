# AIRV CVA6 MAC16 Input Buffer — User and Implementation Guide

This document describes the cleaned **MAC16 Input Buffer** implementation supplied in this package.

The guide is organized in the same way as the previous AIRV accelerator guides: first the project is explained as something that can be built, simulated, run, and debugged; the second half then documents the implementation file by file with exact line ranges from `cleaned_code/`.

This document describes the code that is actually present in this snapshot. It does not compare against another implementation.

---

## 1. Project Structure

The package contains:

```text
MAC16_InputBuffer_Cleaned_v2/
├── README.md
├── USER_GUIDE_MAC16_INPUT_BUFFER.md
└── cleaned_code/
    ├── NetworkPropagate.c
    ├── ariane_pkg.sv
    ├── cva6.sv
    ├── cv32a6_ima_sv32_fpga_config_pkg.sv
    ├── cvxif_example_coprocessor.sv
    ├── cvxif_fu.sv
    ├── cvxif_instr_pkg.sv
    ├── cvxif_pkg.sv
    ├── decoder.sv
    ├── instr_decoder.sv
    ├── issue_read_operands.sv
    ├── issue_stage.sv
    ├── scoreboard.sv
    ├── riscv-opc.c
    ├── riscv-opc.h
    ├── riscv.h
    ├── rv_i
    └── tc-riscv.c
```

These files are the modified source subset. They must be placed back into their corresponding locations in the complete AIRV/CVA6 tree for a full build.

---

## 2. Main Requirements

This implementation needs the same AIRV environment used by the CVA6 project:

```text
CVA6 RTL
RISC-V GNU toolchain with the custom instruction definitions
QuestaSim for RTL simulation
Vivado for FPGA synthesis / bitstream generation
OpenOCD + RISC-V GDB for FPGA execution
Zybo Z7-20 for the FPGA target used by the AIRV setup
```

The supplied archive is not a complete standalone repository, so environment paths are intentionally not hard-coded here.

---

## 3. Environment Setup

Load the normal AIRV environment before building:

```bash
source /path/to/setup.sh
```

Useful checks are:

```bash
echo $PROJECTROOT
which riscv-none-elf-gcc
which riscv-none-elf-objdump
which vsim
which vivado
```

The compiler and assembler must be the custom RISC-V toolchain containing `BUF4`, `MAC16BUF`, and `MAC16BUF_PARA`.

---

## 4. Compile the MNIST Application

In the complete AIRV tree, compile the MNIST application with the project build flow, for example:

```bash
cd $PROJECTROOT/sw/app
make clean
make mnist
```

After compilation, check the disassembly and confirm that the custom instructions are present:

```text
buf4
mac16buf
mac16buf_para
```

This is the first useful validation step because it confirms that the software and modified assembler agree on the instruction names and encodings.

---

## 5. Questa Simulation

Run the normal AIRV simulation flow from the complete project:

```bash
cd $PROJECTROOT
make sim APP=mnist
```

The most useful checks are:

```text
MNIST output
cycle count
instruction count
custom-instruction issue
input-buffer block counters
local accumulator
final writeback
```

For this version, the coprocessor waveform is particularly important because one output can be produced by a sequence of first, middle, and final MAC blocks rather than by one architecturally visible writeback per block.

---

## 6. FPGA Execution

Use the normal AIRV FPGA flow to build and program the Zybo Z7-20.

A typical workflow is:

```text
1. Build the FPGA design / bitstream with Vivado.
2. Program the FPGA.
3. Start OpenOCD.
4. Connect riscv-gdb to the CVA6 target.
5. Load the MNIST ELF.
6. Open the UART terminal.
7. Run the program.
```

The exact commands depend on the full AIRV checkout and local installation, so this source-only package does not invent paths that are not present in the supplied files.

When comparing simulation and FPGA execution, use the same application binary and confirm the network output before comparing performance counters.

---

## 7. Accelerator Overview

This snapshot contains three custom instructions:

```text
BUF4
MAC16BUF
MAC16BUF_PARA
```

They work together around a 400-byte input buffer.

### 7.1 `BUF4`

`BUF4` configures the active input-buffer size and writes one 16-byte block.

The `rd` field is reused as a configuration value:

```text
active_blocks = rd + 1
```

The software uses:

```text
Conv1 -> rd = x0  -> 1 active block
Conv2 -> rd = x24 -> 25 active blocks
FC1   -> rd = x23 -> 24 active blocks
FC2   -> rd = x8  -> 9 active blocks
```

### 7.2 `MAC16BUF`

`MAC16BUF` performs a 16-lane INT8 MAC.

Four packed 32-bit weight words are supplied by the CPU. The corresponding four packed input words are read from the hardware input buffer.

Conceptually:

```text
partial_sum =
    input[0]  * weight[0]
  + input[1]  * weight[1]
  + ...
  + input[15] * weight[15]
```

The block result is accumulated with either the old `rd` value or the local accumulator.

### 7.3 `MAC16BUF_PARA`

`MAC16BUF_PARA` combines two jobs in one instruction:

```text
perform the 16-way MAC
+
write the current 16-byte input block into the input buffer
```

This is used while the buffer is being populated so the first pass over an input does useful computation instead of only filling storage.

---

## 8. Input Buffer Organization

The coprocessor declares:

```systemverilog
INPUT_BUF_WORDS = 100
```

Each word is 32 bits:

```text
100 × 4 bytes = 400 bytes
```

Each MAC16 block consumes four words:

```text
4 × 4 bytes = 16 bytes
```

Therefore the buffer contains:

```text
400 / 16 = 25 blocks
```

The buffer uses separate block counters:

```text
wr_block_cnt_q
rd_block_cnt_q
```

and one active-range register:

```text
active_blocks_q
```

The read and write counters wrap at the configured active-block count rather than always traversing all 25 physical blocks.

---

## 9. Local Accumulator and First / Middle / Final

A multi-block output does not need to write `rd` after every 16-byte MAC.

The coprocessor classifies the MAC sequence as:

```text
first
middle
final
```

The accumulation behavior is:

```text
first:
    acc = old rd + partial_sum

middle:
    acc = acc_q + partial_sum

final:
    result = acc_q + partial_sum
    write result back to rd
```

This is controlled by:

```text
issue_block_cnt_q
issue_active_blocks_q
issue_is_first_block
issue_is_final_block
acc_q
```

Only the final block requests architectural writeback. Middle blocks update the local accumulator and continue.

This mechanism is already part of the supplied Input Buffer snapshot, so it is documented here as part of this implementation.

---

## 10. Layer Mapping

The four network layers use the buffer differently.

### Conv1

Conv1 uses one 16-byte block:

```text
active_blocks = 1
```

The first computation for an input position can use `MAC16BUF_PARA` to compute and populate the buffer at the same time.

The code contains both aligned and unaligned input-loading paths.

### Conv2

Conv2 uses the full buffer:

```text
25 blocks × 16 bytes = 400 bytes
```

The first output filter can populate the 25 blocks while computing. Later filters can reuse the buffered input and execute 25 `MAC16BUF` blocks.

### FC1

FC1 uses:

```text
24 blocks × 16 bytes = 384 bytes
```

The input vector can therefore be buffered once and reused across output neurons.

### FC2

FC2 configures:

```text
9 blocks × 16 bytes = 144 bytes
```

The remaining six input bytes do not form a complete 16-byte block and are handled by the software remainder path.

---

## 11. Nine-Register Operand Path

`MAC16BUF_PARA` needs more operands than a normal RISC-V instruction can encode directly.

This implementation therefore expands the CVA6 integer register-file read path to nine ports.

The combined instruction uses:

```text
four packed weight/source words
old rd / accumulator
four packed input words
```

The extra input words are supplied through fixed architectural registers:

```text
x28
x29
x30
x31
```

The complete path is:

```text
register file
    ↓
issue_read_operands.sv
    ↓
scoreboard.sv
    ↓
issue_stage.sv
    ↓
fu_data_t
    ↓
cvxif_fu.sv
    ↓
CV-X-IF rs[0] ... rs[8]
    ↓
cvxif_example_coprocessor.sv
```

---

## 12. Modified RISC-V Toolchain

The GNU toolchain recognizes:

```text
mac16buf
buf4
mac16buf_para
```

The opcode values in this source snapshot are:

```text
MAC16BUF      -> 0x0b
BUF4          -> 0x2b
MAC16BUF_PARA -> 0x5b
```

The assembler format is:

```text
d,W1,W2,W3,W4
```

`W1`–`W4` are custom five-bit GPR fields. Their extraction and encoding macros are defined in `riscv.h`, and `tc-riscv.c` parses normal RISC-V register names into those fields.

---

## 13. End-to-End Data Path

For `MAC16BUF_PARA`, the complete path is:

```text
NetworkPropagate.c
        ↓
inline assembly
        ↓
GNU assembler
        ↓
32-bit custom instruction
        ↓
decoder.sv
        ↓
operation = MAC16BUF_PARA
        ↓
issue_read_operands.sv
        ↓
9 GPR values
        ↓
scoreboard dependency / forwarding
        ↓
cvxif_fu.sv
        ↓
CV-X-IF rs[0] ... rs[8]
        ↓
cvxif_example_coprocessor.sv
        ↓
four input words + four weight words
        ↓
16 signed products
        ↓
partial_sum
        ↓
old rd or acc_q
        ↓
mac_next_acc
        ↓
input buffer updated in parallel
        ↓
final block only: rd writeback
```

For `MAC16BUF`, the MAC datapath is similar, but the four packed input words come from `input_buffer` instead of the extra CV-X-IF input registers.

---

## 14. Quick Validation

After integrating the cleaned files into the full project, validate in this order:

```text
1. Build the custom toolchain/application.
2. Check disassembly for BUF4 / MAC16BUF / MAC16BUF_PARA.
3. Run the known MNIST simulation.
4. Confirm network output.
5. Compare cycle and instruction counts with the known working snapshot.
6. Inspect the input-buffer and accumulator waveform.
7. Run the same ELF on FPGA.
8. Confirm the FPGA output matches the expected network result.
```

Do not use timing or performance numbers as the first correctness test. Functional output and the MAC/buffer state sequence should be checked first.

---

# 15. Implementation Guide

All line numbers below refer to the files in `cleaned_code/`.

## 15.1 `NetworkPropagate.c`

### Lines 48–93 — layer-specific `BUF4` configuration

Defines:

```text
buffer4_setmode_conv1()
buffer4_setmode_conv2()
buffer4_setmode_fc1()
buffer4_setmode_fc2()
```

Each helper emits `buf4` with a different `rd` field.

The hardware interprets that field as `active_blocks = rd + 1`, giving one block for Conv1, 25 for Conv2, 24 for FC1, and nine for FC2.

### Lines 97–141 — aligned Conv1 `MAC16BUF_PARA`

`mac16buf_para_conv1_aligned()` loads four packed input words and four packed weight words with aligned word loads.

The four input words are placed in the fixed extra registers used by the nine-source path, then `mac16buf_para` performs the 16-way MAC and updates the hardware input buffer.

### Lines 144–201 — unaligned Conv1 `MAC16BUF_PARA`

`mac16buf_para_conv1_unaligned()` provides the corresponding path for input addresses that cannot use the aligned four-word sequence directly.

The function reconstructs packed input data with halfword loads while preserving the same accelerator operand layout.

### Lines 204–227 — Conv1 buffered MAC

`mac16buf_conv1()` loads the four packed weight words and issues `mac16buf`.

No new input words are supplied by the software instruction because the input data are taken from the hardware input buffer.

### Lines 230–250 — first block helper

`mac16buf_first()` starts a multi-block accumulation.

The destination register contains the initial accumulator value. The coprocessor recognizes block zero as the first block and combines the partial sum with the old `rd`.

### Lines 253–271 — middle block helper

`mac16buf_middle()` issues a non-final block using `x0` as the software-visible destination.

The useful partial sum is retained by the coprocessor in `acc_q`, so no architectural result is needed for this middle block.

### Lines 274–313 — four-way middle unrolling

`mac16buf_middle4()` emits four consecutive middle `MAC16BUF` instructions.

This reduces software loop/control overhead while the hardware read-block counter advances through four buffered blocks.

### Lines 316–422 — four `MAC16BUF_PARA` middle blocks

`mac16buf_para_middle4()` combines four MAC operations with four input-buffer writes.

Each instruction provides a new 16-byte input block through the fixed extra registers while also supplying the corresponding packed weights.

### Lines 425–448 — final block helper

`mac16buf_final()` issues the final MAC block and receives the completed accumulated result.

This is the point at which the coprocessor allows architectural writeback.

### Lines 451–504 — first block with two-row input addressing

`mac16buf_first_offset2()` prepares the first MAC block for the Conv2-specific two-row memory layout.

The helper preserves the non-contiguous row relationship required by this layer rather than treating the input as one simple packed linear stream.

### Lines 507–553 — middle block with two-row input addressing

`mac16buf_middle_offset2()` performs the corresponding middle-block operation for the same Conv2 memory organization.

### Lines 556–606 — final block with two-row input addressing

`mac16buf_final_offset2()` completes the two-row sequence and returns the final accumulated result.

### Lines 609–642 — generic combined first block

`mac16buf_para_first()` supplies one 16-byte input block and four packed weight words through `MAC16BUF_PARA`.

It is used when buffer population and the first accumulation block can be performed together.

### Lines 645–672 — generic combined middle block

`mac16buf_para_middle()` performs the same combined input-buffer fill and MAC operation for a middle block.

The intermediate result remains local to the coprocessor.

### Lines 675–708 — generic combined final block

`mac16buf_para_final()` performs the last combined block and receives the completed result.

### Lines 711–753 — complete Conv2 25-block buffered MAC

`mac16buf_conv2_25blocks()` explicitly builds one 25-block accumulation:

```text
1 first
23 middle
1 final
```

The middle section uses `mac16buf_middle4()` where possible to reduce software overhead.

### Lines 756–788 — complete FC1 24-block buffered MAC

`mac16buf_fc1_24blocks()` performs the 24-block FC1 accumulation using the same first/middle/final convention.

### Lines 791–798 — scalar fallback MAC

`macsOnRange()` remains available for data that are not processed through a complete MAC16 buffer block.

This is important for remainders and general fallback paths.

### Lines 801–824 — activation and saturation

`saturate()` and `sat()` apply the existing post-accumulation activation and output-range handling.

The accelerator changes the MAC computation, not the network activation semantics.

### Lines 827–1091 — Conv1 integration

`convcellPropagate1()` configures the one-block input-buffer mode.

For the first output of a patch, the code can use `MAC16BUF_PARA` to compute while loading the input block. The code selects aligned or unaligned loading according to the input address.

After the block is available in hardware, later output-channel calculations can use `MAC16BUF` and reuse that input.

### Lines 1094–1258 — Conv2 integration

`convcellPropagate2()` configures 25 active blocks.

The first filter can use the combined `MAC16BUF_PARA` path to populate the 400-byte input-buffer window while computing. Once the input window has been stored, subsequent filters can reuse the same 25 blocks through `MAC16BUF`.

### Lines 1261–1385 — FC1 integration

`fccellPropagateUDATA_T()` configures 24 active blocks, corresponding to the 384-byte FC1 input vector.

The buffer allows the same input vector to be reused across output neurons instead of reloading all input data for every output.

### Lines 1388–1703 — FC2 integration

`fccellPropagateDATA_T()` configures nine active blocks.

Those blocks cover 144 bytes. The remaining six FC2 input bytes are handled separately because they do not fill another 16-byte MAC block.

### Lines 1706–1748 — maximum-output selection

`maxPropagate1()` selects the final network output after FC2.

This is not part of the custom accelerator datapath but remains part of the complete network propagation flow.

### Lines 1751 onward — network-level layer sequence

`propagate()` calls Conv1, Conv2, FC1, FC2, and the final maximum-selection stage.

This is the software entry point that connects the accelerated layer implementations into the complete MNIST inference.

---

## 15.2 `cv32a6_ima_sv32_fpga_config_pkg.sv`

### Line 21 — enable CV-X-IF

Sets:

```systemverilog
CVA6ConfigCvxifEn = 1
```

The three custom accelerator instructions execute through the CV-X-IF coprocessor path.

---

## 15.3 `cva6.sv`

### Line 164 — nine integer register-file read ports

Sets:

```systemverilog
NrRgprPorts = 9
```

The combined `MAC16BUF_PARA` instruction needs the expanded source path.

---

## 15.4 `ariane_pkg.sv`

### Line 75 — global GPR read-port count

Sets:

```systemverilog
NR_RGPR_PORTS = 9
```

This value is also used by the CV-X-IF package to determine the number of source registers carried in an issue request.

### Lines 447–449 — accelerator operation identifiers

Adds:

```text
MAC16BUF
MAC16BUF_PARA
BUF4
```

to the functional-unit operation enumeration.

### Lines 582–587 — extra execution operands

Adds:

```text
operand_d
operand_e
operand_f
operand_g
operand_h
operand_i
```

to `fu_data_t`.

These fields carry the additional register-file values from issue to CV-X-IF.

---

## 15.5 `decoder.sv`

### Lines 1189–1220 — custom accelerator decode

Decodes the three custom opcodes and marks them as CV-X-IF operations.

The selected operation distinguishes `MAC16BUF`, `BUF4`, and `MAC16BUF_PARA`.

### Lines 1302–1304 — transport additional encoded register indices

For the accelerator instructions, the custom upper instruction fields are packed into `instruction_o.result`.

`issue_read_operands.sv` later uses those bits as additional GPR addresses.

---

## 15.6 `issue_read_operands.sv`

### Lines 43–61 — rs4 through rs9 interface

Adds six additional source-register channels beyond the normal operand interface.

These channels allow the nine-port register-file configuration to participate in scoreboard forwarding and issue stalls.

### Lines 109–117 — extra operand storage

Declares the register-file and pipeline values for `operand_d` through `operand_i`.

### Lines 137 and 151–156 — forwarding controls and FU-data mapping

Adds forwarding controls for rs4–rs9 and connects the registered values into `fu_data_o`.

### Lines 204–210 — forwarding defaults

Initializes all additional forwarding controls to zero at the beginning of operand-selection logic.

### Lines 215–240 — accelerator register-address mapping

For `MAC16BUF` and `MAC16BUF_PARA`, `rd` is used as the accumulator source.

For the custom encoded fields, rs4 and rs5 come from `instruction_o.result`.

For `MAC16BUF_PARA`, four additional sources are fixed to:

```text
rs6 = x28
rs7 = x29
rs8 = x30
rs9 = x31
```

These registers carry the four packed input words used by the combined buffer/MAC operation.

### Lines 283–322 — dependency and forwarding checks

Extends the operand-availability logic to the accelerator accumulator and extra sources.

If a source is still waiting for an older instruction, the logic uses forwarding when possible; otherwise the current instruction stalls.

### Lines 329–396 — extra operand selection

Selects register-file or forwarded values for the expanded source set and builds the next operand values.

### Lines 578–588 — nine-port GPR address packing

Packs the accelerator register addresses into the nine read ports.

This is the physical register-file read configuration that makes the combined instruction possible.

### Lines 688–708 — map register-file data to operands

Maps the register-file outputs into `operand_c` and `operand_d` through `operand_i`.

### Lines 718–742 — reset and pipeline extra operands

Resets and registers the additional operands together with the existing issue-stage data.

### Line 756 — accepted read-port configuration

Extends the configuration assertion to accept the nine-port design.

---

## 15.7 `issue_stage.sv`

### Line 100 — full-width third source in nine-port mode

Uses XLEN width for the third GPR source when `NrRgprPorts == 9`.

This keeps the accumulator path full width.

### Lines 117–139 — rs4 through rs9 signals

Declares the extra address, data, and valid channels.

### Lines 178–201 — scoreboard connections

Connects all additional source channels to `scoreboard.sv`.

### Lines 246–264 — read-operands connections

Connects the same channels to `issue_read_operands.sv`.

---

## 15.8 `scoreboard.sv`

### Lines 43–67 — extra source-register interface

Adds rs4 through rs9 to the scoreboard module interface.

### Lines 337–339 — forwarding state

Declares forwarding request and valid signals for the expanded source set.

### Lines 352–358 — current writeback dependency checks

Checks whether any current writeback result supplies rs4–rs9.

### Lines 374–379 — in-flight scoreboard dependency checks

Checks issued scoreboard entries for pending writes to the same extra source registers.

### Lines 393–402 — source validity

Generates validity for the third through ninth sources in the nine-port configuration.

### Lines 471–577 — forwarding arbiters

Adds independent arbitration for rs4, rs5, rs6, rs7, rs8, and rs9.

Each source therefore receives the newest valid forwarded value when a dependency can be resolved without stalling.

---

## 15.9 `cvxif_pkg.sv`

### Line 15 — CV-X-IF source count

Sets:

```systemverilog
X_NUM_RS = ariane_pkg::NR_RGPR_PORTS
```

With this source snapshot, CV-X-IF therefore carries nine source-register values.

---

## 15.10 `cvxif_fu.sv`

### Lines 43–46 — nine-source valid vector

Enables the nine-source CV-X-IF request path.

### Lines 60–67 — map extra operands into CV-X-IF

Maps:

```text
operand_d -> rs[3]
operand_e -> rs[4]
operand_f -> rs[5]
operand_g -> rs[6]
operand_h -> rs[7]
operand_i -> rs[8]
```

Together with the first three sources, this transfers the complete accelerator operand set to the coprocessor.

---

## 15.11 `cvxif_instr_pkg.sv`

### Line 19 — three supported custom instructions

Sets the instruction table size to three.

### Lines 20–57 — CV-X-IF instruction patterns

Defines the table entries for the three custom instructions.

The relevant opcodes are:

```text
MAC16BUF      -> custom-0 / 0x0b
BUF4          -> custom-1 / 0x2b
MAC16BUF_PARA -> custom-2 / 0x5b
```

The table tells the generic CV-X-IF instruction decoder which requests the coprocessor accepts.

---

## 15.12 `cvxif_example_coprocessor.sv`

This file contains the core input-buffer and MAC16 behavior.

### Lines 84–95 — accelerator metadata stored with FIFO requests

Extends each FIFO entry with flags identifying the custom MAC instructions and their first/final block state.

The metadata stays associated with the request while it moves from issue to execution.

### Lines 102–124 — issue-side block tracking

Defines:

```text
issue_active_blocks_q
issue_block_cnt_q
issue_buf_active_blocks
issue_is_first_block
issue_is_final_block
```

The opcode identifies the instruction type, while the block counter identifies where the current MAC lies inside one output accumulation.

### Lines 126–143 — writeback response control

Starts from the normal instruction-table response and overrides the MAC writeback behavior.

Only a final MAC block requests architectural writeback. Non-final blocks can still complete through CV-X-IF while retaining their partial result locally.

### Lines 147–165 — attach block state and update issue counter

Stores the instruction-type and first/final flags in the request metadata.

`BUF4` changes the active block count and resets the sequence. Accepted MAC instructions advance the issue block counter and wrap it after the final block.

### Lines 213–231 — input-buffer storage and state

Defines:

```text
INPUT_BUF_WORDS = 100
input_buffer[0:99]
wr_block_cnt_q
rd_block_cnt_q
active_blocks_q
acc_q
```

This is the central hardware state for the input-buffer implementation.

### Lines 233–248 — block base addresses and execution classification

Converts block numbers into word indices and identifies whether the current FIFO output is `BUF4`, `MAC16BUF`, or `MAC16BUF_PARA`.

### Lines 251–303 — buffer write/read counters and local accumulator

On reset, the buffer state and counters are cleared.

`BUF4` writes four 32-bit words into one 16-byte block.

`MAC16BUF_PARA` writes its four extra input words into the buffer while the instruction is being executed.

The read block counter advances for MAC operations and wraps at the active block count.

`acc_q` stores `mac_next_acc`, which keeps intermediate multi-block accumulation local to the coprocessor.

### Lines 308–381 — 16-lane INT8 MAC datapath

Splits four packed input words and four packed weight words into 16 byte lanes.

The hardware computes 16 products and adds them into `partial_sum`.

For `MAC16BUF_PARA`, the input words come directly from CV-X-IF.

For `MAC16BUF`, the input words come from `input_buffer` at `rd_base`.

The accumulator base is selected according to the block position, and the result becomes `mac_next_acc`.

### Lines 386–397 — result generation and final writeback

Returns the completed MAC value through CV-X-IF.

The architectural write-enable is asserted only when the current MAC is marked as the final block.

---

## 15.13 `rv_i`

### Lines 27–31 — assembler opcode definitions

Defines:

```text
mac16buf
buf4
mac16buf_para
```

with their custom opcode spaces.

---

## 15.14 `riscv-opc.h`

### Lines 25–30 — match and mask constants

Defines:

```c
MATCH_MAC16BUF
MASK_MAC16BUF
MATCH_BUF4
MASK_BUF4
MATCH_MAC16BUF_PARA
MASK_MAC16BUF_PARA
```

### Lines 2795–2797 — GNU opcode declarations

Registers all three instruction names with the RISC-V opcode infrastructure.

---

## 15.15 `riscv-opc.c`

### Lines 323–325 — assembler instruction entries

Adds the three mnemonics with the custom operand format:

```text
d,W1,W2,W3,W4
```

This connects the textual assembly syntax to the custom encoding fields.

---

## 15.16 `riscv.h`

### Lines 109–128 — custom register-field encoding helpers

Defines the custom register-field extraction/encoding macros.

The W fields occupy four five-bit register-index positions and are shared by the custom accelerator assembly format.

---

## 15.17 `tc-riscv.c`

### Lines 1393–1401 — validate custom W fields

Adds the `W` operand class and marks the instruction bits used by W1–W4.

### Lines 3311–3330 — parse custom register operands

Parses normal RISC-V GPR names and encodes their register numbers into W1–W4.

---

## 15.18 `instr_decoder.sv`

No accelerator-specific logic was added to this delivered file.

The generic decoder consumes the instruction table defined in `cvxif_instr_pkg.sv`; the custom patterns themselves are therefore maintained in the package rather than hard-coded here.

---

## 16. Recommended Questa Signals

For this implementation, a useful waveform set is:

```text
issue_is_buf4
issue_is_mac16buf
issue_is_mac16buf_para

issue_active_blocks_q
issue_block_cnt_q
issue_is_first_block
issue_is_final_block

active_blocks_q
wr_block_cnt_q
rd_block_cnt_q

acc_q
partial_sum
mac_base_acc
mac_next_acc

x_result_valid_o
x_result_o.we
x_result_o.data
```

A normal multi-block output should show:

```text
first block
    ↓
acc_q initialized from old rd + partial_sum
    ↓
middle block(s)
    ↓
acc_q repeatedly updated
    ↓
final block
    ↓
x_result_o.we = 1
    ↓
completed result written to rd
```

For `MAC16BUF_PARA`, also inspect the four extra CV-X-IF source registers and the corresponding `input_buffer` locations to verify that buffer filling and MAC execution occur together.

---

## 17. Troubleshooting

### Custom instruction is not present in disassembly

Check:

```text
rv_i
riscv-opc.h
riscv-opc.c
riscv.h
tc-riscv.c
```

and confirm that the application is using the modified RISC-V toolchain.

### Instruction is present but never reaches the coprocessor

Check:

```text
decoder.sv
cvxif_instr_pkg.sv
CVA6ConfigCvxifEn
```

### `MAC16BUF_PARA` receives wrong input words

Check the x28–x31 software loads, the rs6–rs9 mapping in `issue_read_operands.sv`, forwarding in `scoreboard.sv`, and `cvxif_fu.sv` rs[5]–rs[8].

### First block is correct but later blocks are wrong

Check:

```text
active_blocks_q
rd_block_cnt_q
issue_block_cnt_q
acc_q
```

and verify that the software configured the correct active block count for the layer.

### Every block writes `rd`

Check `issue_is_final_block` and the writeback override in `cvxif_example_coprocessor.sv`.

The intended behavior is that only the final block produces the architectural result.

### Conv1 fails only at some input positions

Check the aligned/unaligned branch in `NetworkPropagate.c`.

The unaligned Conv1 helper exists specifically because not every input position can safely use the same aligned word-load sequence.

### FC2 result differs while earlier layers are correct

Check the nine buffered 16-byte blocks and the six-byte scalar remainder path separately.

FC2 does not consist entirely of complete 16-byte MAC16 blocks in this source snapshot.

---

## 18. Verification Note

The cleanup for this package is intentionally conservative.

It is intended to improve naming, comments, formatting, and documentation without intentionally changing:

```text
custom instruction encodings
nine-port register mapping
x28–x31 input convention
400-byte input buffer
active-block configuration
read/write block counters
16-lane INT8 arithmetic
first/middle/final behavior
local accumulator behavior
layer-specific memory access
```

The supplied files are only the modified source subset, so this package by itself is not sufficient for a complete CVA6 regression.

After replacing the corresponding files in the full AIRV tree, the recommended final check is:

```text
clean rebuild
→ disassembly check
→ MNIST simulation
→ output comparison
→ cycle/instruction comparison
→ Questa waveform inspection
→ FPGA execution
```
