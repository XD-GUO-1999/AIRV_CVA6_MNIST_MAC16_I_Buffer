# AIRV CVA6 MAC16 Input Buffer --- User and Implementation Guide

This guide documents the cleaned MAC16 input-buffer source snapshot
supplied with this package.

All implementation line numbers refer directly to `cleaned_code/`. The
guide describes the delivered code itself: each entry gives the relevant
line range, what the code does, and why it exists.

The supplied snapshot contains the input buffer together with the
block-local accumulator and the combined `MAC16BUF_PARA` path. Those
mechanisms are therefore documented here because they are part of this
source version.

------------------------------------------------------------------------

## 1. Architecture Overview

Three custom instructions are used:

``` text
BUF4
MAC16BUF
MAC16BUF_PARA
```

`BUF4` configures/fills the input buffer. `MAC16BUF` reads input data
from that buffer and combines it with four packed weight words.
`MAC16BUF_PARA` supplies four packed weight words and four packed input
words in the same instruction, allowing input-buffer filling and MAC
computation to overlap.

The hardware input buffer contains 100 32-bit words:

``` text
100 words × 4 bytes = 400 bytes
400 bytes / 16 bytes per block = 25 blocks
```

The software selects the active block count through the `rd` field of
`BUF4`:

``` text
active_blocks = rd + 1

Conv1: rd = x0  -> 1 block
Conv2: rd = x24 -> 25 blocks
FC1:   rd = x23 -> 24 blocks
FC2:   rd = x8  -> 9 blocks
```

For the combined instruction, CVA6 exposes nine GPR source values to
CV-X-IF: four weight words, the accumulator, and four input words.

------------------------------------------------------------------------

## 2. Execution Flow

``` text
NetworkPropagate.c
        ↓
custom GNU assembler encoding
        ↓
decoder.sv
        ↓
MAC16BUF / BUF4 / MAC16BUF_PARA
        ↓
issue_read_operands.sv
        ↓
9 GPR read ports
        ↓
scoreboard.sv
        ↓
cvxif_fu.sv
        ↓
CV-X-IF request
        ↓
cvxif_example_coprocessor.sv
        ↓
input buffer + 16-lane MAC + local accumulator
        ↓
final block result
        ↓
rd writeback
```

------------------------------------------------------------------------

## 3. Software Layer --- `NetworkPropagate.c`

### Lines 48--93 --- layer-specific input-buffer configuration

Defines four small `BUF4` helpers.

`buffer4_setmode_conv1()` selects one active block,
`buffer4_setmode_conv2()` selects 25, `buffer4_setmode_fc1()` selects
24, and `buffer4_setmode_fc2()` selects nine.

The `rd` field is used as the block-count configuration value rather
than as a normal architectural destination for these configuration
instructions.

### Lines 97--141 --- aligned Conv1 buffer-and-MAC path

`mac16buf_para_conv1_aligned()` loads four packed weight words and four
aligned packed input words.

The input words are placed in `t3`--`t6` and `MAC16BUF_PARA` performs
the 16-way MAC while also making the input data available to the
hardware buffer path.

### Lines 144--201 --- unaligned Conv1 path

`mac16buf_para_conv1_unaligned()` reconstructs each four-byte input word
from two halfword loads.

This preserves the MAC16 path when a Conv1 input row is not suitable for
a direct aligned `lw`.

### Lines 204--227 --- buffered Conv1 MAC

`mac16buf_conv1()` loads four weight words and executes `MAC16BUF`.

The corresponding input block is read from the hardware input buffer
rather than reloaded from memory.

### Lines 230--313 --- first/middle MAC16 helpers

`mac16buf_first()`, `mac16buf_middle()`, and `mac16buf_middle4()`
implement the software instruction sequence used by multi-block outputs.

The first instruction carries the initial accumulator value. Middle
instructions continue the hardware-local accumulation without requiring
an architectural result after every block. `mac16buf_middle4()` unrolls
four middle blocks to reduce loop/control overhead.

### Lines 316--422 --- four-way combined buffer/MAC helper

`mac16buf_para_middle4()` issues four `MAC16BUF_PARA` operations.

Each operation loads four packed weights and places four packed input
words in the extra fixed registers. This path overlaps input-buffer
population with useful MAC work.

### Lines 425--448 --- final MAC16 block

`mac16buf_final()` executes the final block and receives the completed
accumulated value.

This corresponds to the hardware rule that only the final block writes
the architectural result.

### Lines 451--606 --- two-row offset helpers

`mac16buf_first_offset2()`, `mac16buf_middle_offset2()`, and
`mac16buf_final_offset2()` support the Conv2 two-row memory layout.

They preserve the layer-specific input addressing needed when packed
data are drawn from separated rows.

### Lines 609--708 --- generic combined buffer/MAC helpers

`mac16buf_para_first()`, `mac16buf_para_middle()`, and
`mac16buf_para_final()` provide first/middle/final forms of the combined
instruction for contiguous input data.

### Lines 711--753 --- Conv2 25-block MAC sequence

`mac16buf_conv2_25blocks()` processes the complete 25-block Conv2
input-buffer window.

The function starts with a first block, uses unrolled middle blocks, and
finishes with a final block so the coprocessor can keep the partial sum
locally until the output is complete.

### Lines 756--824 --- FC1 24-block MAC sequence

`mac16buf_fc1_24blocks()` applies the same block-accumulation scheme to
the 384-byte FC1 input vector.

### Lines 827--1091 --- Conv1 integration

`convcellPropagate1()` configures the one-block Conv1 mode and selects
the aligned or unaligned `MAC16BUF_PARA` path according to the input
address.

The first output can populate the buffer while computing. Subsequent
work can reuse the buffered input through `MAC16BUF`.

### Lines 1094--1258 --- Conv2 integration

`convcellPropagate2()` selects the 25-block mode and integrates the
Conv2-specific buffer population and `mac16buf_conv2_25blocks()`
computation.

### Lines 1261--1385 --- FC1 integration

`fccellPropagateUDATA_T()` selects the 24-block mode.

The code uses combined buffer/MAC operations while input data are being
introduced, then reuses the buffered 384-byte input for later output
neurons.

### Lines 1388--1748 --- FC2 integration

`fccellPropagateDATA_T()` selects nine buffered blocks for the first 144
input bytes.

The remaining six bytes cannot form another 16-byte block and are
handled by the scalar remainder path.

### Lines 1751 onward --- network execution

`propagate()` invokes Conv1, Conv2, FC1, and FC2 using the accelerated
layer functions.

------------------------------------------------------------------------

## 4. CPU Configuration

### `cv32a6_ima_sv32_fpga_config_pkg.sv` --- Line 21

Enables CV-X-IF:

``` systemverilog
CVA6ConfigCvxifEn = 1
```

The accelerator instructions are executed by the CV-X-IF coprocessor
path.

### `cva6.sv` --- Line 164

Sets:

``` systemverilog
NrRgprPorts = 9
```

Nine GPR read values are required by `MAC16BUF_PARA`.

### `ariane_pkg.sv` --- Line 75

Sets the global register-file read-port count to nine.

### `ariane_pkg.sv` --- Lines 447--449

Adds the three accelerator operations:

``` text
MAC16BUF
MAC16BUF_PARA
BUF4
```

### `ariane_pkg.sv` --- Lines 582--587

Adds `operand_d` through `operand_i` to `fu_data_t`.

Together with the existing operands, these fields transport all nine
source values toward CV-X-IF.

------------------------------------------------------------------------

## 5. Instruction Decode --- `decoder.sv`

### Lines 1189--1220 --- custom instruction decode

Adds decode entries for the three custom opcodes and sends them to the
CV-X-IF functional unit.

The operations are identified as `MAC16BUF`, `BUF4`, and
`MAC16BUF_PARA`.

### Lines 1300--1304 --- additional register fields

Reuses the RS3 immediate path to carry the two custom register indices
encoded in instruction bits `[31:27]` and `[26:22]`.

Those indices later become additional GPR read addresses.

------------------------------------------------------------------------

## 6. Nine-Operand Read Path --- `issue_read_operands.sv`

### Lines 43--61 --- additional source interfaces

Adds rs4 through rs9 interfaces for the accelerator path.

### Lines 109--117 --- additional operand storage

Adds register-file and pipeline storage for `operand_d` through
`operand_i`.

### Lines 136--156 --- forwarding and FU-data outputs

Adds forwarding state for the extra sources and connects all additional
operands into `fu_data_o`.

### Lines 215--240 --- accelerator source-register mapping

Uses `rd` as the accumulator source for MAC operations.

For `MAC16BUF_PARA`, the extra input words are mapped to fixed registers
x28--x31. The encoded W3/W4-style fields provide the additional
explicitly encoded register addresses used by the custom instruction
format.

### Lines 283--322 --- dependency checks

Extends clobber/forwarding checks to the accumulator and the additional
accelerator sources.

If a required register is still in flight, the instruction either
receives a forwarded value or stalls until the value is ready.

### Lines 329--396 --- operand selection and forwarding

Selects the nine-port GPR path and substitutes forwarded values for
rs4--rs9 when required.

### Lines 578--588 --- nine-port register address packing

Builds the nine GPR read addresses.

For `MAC16BUF_PARA`, the upper four ports read x28, x29, x30, and x31,
allowing four packed input words to be supplied in parallel with the
four packed weight/accumulator operands.

### Lines 688--708 --- register-file output mapping

Maps the nine register-file outputs into the normal and extended
execution operands.

### Lines 718--742 --- pipeline registers

Resets and pipelines `operand_d` through `operand_i`.

### Line 756 --- supported read-port configurations

Allows the nine-port register-file configuration used by this design.

------------------------------------------------------------------------

## 7. Issue Stage --- `issue_stage.sv`

### Line 100

Uses an XLEN-wide third source for the nine-port configuration so the
accumulator remains a full-width value.

### Lines 116--139

Declares the rs4--rs9 channels.

### Lines 178--201

Connects rs4--rs9 to the scoreboard for dependency checking and
forwarding.

### Lines 246--264

Connects rs4--rs9 to `issue_read_operands.sv`.

------------------------------------------------------------------------

## 8. Scoreboard --- `scoreboard.sv`

### Lines 43--67

Adds scoreboard interfaces for rs4 through rs9.

### Lines 337--339

Adds forwarding request and validity state for all nine sources.

### Lines 355--379

Checks current writeback ports and in-flight scoreboard entries for
dependencies on rs6--rs9, in addition to the earlier sources.

### Lines 399--402

Generates validity for rs6--rs9 while excluding x0.

### Lines 510--577

Adds forwarding arbiters for rs6, rs7, rs8, and rs9.

These paths are required because x28--x31 remain architectural registers
and can have normal RAW dependencies.

------------------------------------------------------------------------

## 9. CV-X-IF Operand Transport

### `cvxif_pkg.sv` --- Line 15

Sets the CV-X-IF source count from `NR_RGPR_PORTS`, which is nine in
this snapshot.

### `cvxif_fu.sv` --- Lines 43--46

Enables the nine-source valid vector.

### `cvxif_fu.sv` --- Lines 60--67

Maps the additional execution operands into CV-X-IF `rs[3]` through
`rs[8]`.

This is the final CPU-side transport step before the request reaches the
coprocessor.

------------------------------------------------------------------------

## 10. CV-X-IF Instruction Table --- `cvxif_instr_pkg.sv`

### Line 19

Sets the coprocessor instruction-table size to three.

### Lines 20--57

Defines the patterns for:

``` text
BUF4            custom-1 / opcode 0x2b
MAC16BUF        custom-0 / opcode 0x0b
MAC16BUF_PARA   custom-2 / opcode 0x5b
```

The table determines which instructions are accepted and whether they
are eligible for result writeback.

------------------------------------------------------------------------

## 11. Coprocessor --- `cvxif_example_coprocessor.sv`

### Lines 84--95 --- FIFO metadata

Extends each queued issue entry with instruction type and first/final
block flags.

This keeps the block state associated with the instruction while it
moves through the FIFO.

### Lines 102--124 --- issue-side block tracking

Detects `BUF4`, `MAC16BUF`, and `MAC16BUF_PARA`.

`issue_active_blocks_q` and `issue_block_cnt_q` determine whether a MAC
instruction is the first, middle, or final block of one output
accumulation.

### Lines 126--136 --- writeback policy

Overrides the normal decoded response so only the final MAC block
requests architectural writeback.

Middle blocks still complete internally but keep their partial sum in
the local accumulator.

### Lines 143--168 --- FIFO metadata and counter update

Stores the accelerator flags with the request and advances/resets the
issue-side block counter on accepted instructions.

### Lines 213--231 --- input-buffer storage and addressing

Defines:

``` systemverilog
INPUT_BUF_WORDS = 100
```

and the 100-word input buffer.

`wr_block_cnt_q`, `rd_block_cnt_q`, and `active_blocks_q` track buffer
write/read positions and the configured number of active blocks.

### Lines 248--303 --- input-buffer state update

`BUF4` writes four 32-bit words into the selected 16-byte block.

`MAC16BUF_PARA` can also write four input words while performing a MAC.

The write and read block counters wrap according to the active block
count.

### Lines 292--300 --- local accumulator update

The local accumulator follows three phases:

``` text
first  -> old rd + partial_sum
middle -> acc_q + partial_sum
final  -> acc_q + partial_sum, then architectural writeback
```

This avoids reading and writing `rd` for every 16-byte block.

### Lines 308--381 --- 16-lane INT8 MAC

Four packed input words and four packed weight words are split into 16
byte lanes.

The hardware computes `p0` through `p15`, sums them into `partial_sum`,
chooses either old `rd` or `acc_q` as `mac_base_acc`, and produces
`mac_next_acc`.

`MAC16BUF` takes its inputs from `input_buffer`.

`MAC16BUF_PARA` takes its input words directly from CV-X-IF and
simultaneously stores them into the buffer.

### Lines 386--397 --- result and final writeback

Returns `mac_next_acc` as the result data.

`x_result_o.we` is asserted only for a final MAC block, so middle blocks
update only the local accumulator.

------------------------------------------------------------------------

## 12. GNU Toolchain Support

### `rv_i` --- Lines 27--31

Defines the three custom opcodes:

``` text
mac16buf
buf4
mac16buf_para
```

### `riscv-opc.h` --- Lines 25--30

Defines the match/mask constants:

``` text
MAC16BUF      -> 0x0b
BUF4          -> 0x2b
MAC16BUF_PARA -> 0x5b
```

### `riscv-opc.h` --- Lines 2795--2797

Declares the three instructions to the GNU RISC-V opcode infrastructure.

### `riscv-opc.c` --- Lines 323--325

Adds assembler entries using the custom operand form:

``` text
d,W1,W2,W3,W4
```

### `riscv.h` --- Lines 109 onward

Defines extraction/encoding helpers for the four custom register fields
W1--W4.

### `tc-riscv.c` --- Lines 1392--1402

Registers the `W1`--`W4` operand fields as normal GPR indices in custom
bit positions.

### `tc-riscv.c` --- Lines 3310--3332

Parses the W operands and inserts their register numbers into the custom
instruction encoding.

------------------------------------------------------------------------

## 13. `instr_decoder.sv`

No accelerator-specific modification is required in the delivered
`instr_decoder.sv`.

The instruction patterns are supplied through `cvxif_instr_pkg.sv`, and
this generic decoder selects the matching table entry.

------------------------------------------------------------------------

## 14. Recommended Debugging Order

For an incorrect MAC16 input-buffer result, inspect the design in this
order:

``` text
1. Disassembly
   -> BUF4 / MAC16BUF / MAC16BUF_PARA encoding

2. decoder.sv
   -> operation and register fields

3. issue_read_operands.sv
   -> nine GPR addresses and operand values

4. scoreboard.sv
   -> stalls / forwarding for extra sources

5. cvxif_fu.sv
   -> rs[0] ... rs[8]

6. coprocessor issue state
   -> issue_active_blocks_q
   -> issue_block_cnt_q
   -> is_first_block
   -> is_final_block

7. input-buffer state
   -> active_blocks_q
   -> wr_block_cnt_q
   -> rd_block_cnt_q
   -> input_buffer

8. MAC datapath
   -> p0 ... p15
   -> partial_sum
   -> mac_base_acc
   -> mac_next_acc
   -> acc_q

9. final writeback
   -> x_result_o.we
   -> x_result_o.data
```

Useful Questa signals include:

``` text
issue_is_buf4
issue_is_mac16buf
issue_is_mac16buf_para
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
x_result_o.we
```

------------------------------------------------------------------------

## 15. Validation Note

The cleanup is intentionally conservative. It focuses on readable
comments, removal of obsolete commented development code, clearer naming
where safe, and consistent formatting.

The custom instruction encodings, buffer size, block-count convention,
register mapping, 16-lane arithmetic, local accumulator behavior, and
layer-specific memory accesses were not intentionally changed.

This archive contains the modified source subset rather than a complete
standalone AIRV/CVA6 build tree. After integrating the cleaned files
into the full project, run the same MNIST simulation used for this
source version and compare functional outputs, cycles/instructions, and
the key Questa signals above before freezing the cleaned snapshot.
