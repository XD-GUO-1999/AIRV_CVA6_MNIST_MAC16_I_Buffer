# AIRV CVA6 MAC16 Input-Buffer Accelerator

This repository snapshot implements the MAC16 input-buffer stage of the
AIRV/CVA6 MNIST accelerator.

The design includes:

-   `MAC16BUF`: 16-way INT8 MAC using inputs already stored in the
    hardware input buffer
-   `BUF4`: input-buffer configuration/fill instruction
-   `MAC16BUF_PARA`: combined input-buffer fill and MAC operation
-   400-byte input buffer organized as 25 blocks of 16 bytes
-   local block accumulator with first/middle/final behavior
-   nine GPR read ports for the combined MAC/buffer instruction
-   CV-X-IF execution path
-   GNU assembler/toolchain support
-   software integration for Conv1, Conv2, FC1, and FC2

## Documentation

See [USER_GUIDE_MAC16_INPUT_BUFFER.md](USER_GUIDE_MAC16_INPUT_BUFFER.md)
for the architecture, software flow, exact cleaned-code line locations,
and debugging procedure.

## Source Code

The cleaned source files are in:

``` text
cleaned_code/
```

## Main Data Flow

``` text
NetworkPropagate.c
        ↓
BUF4 / MAC16BUF / MAC16BUF_PARA
        ↓
CVA6 decoder
        ↓
9-port GPR read path
        ↓
scoreboard / forwarding
        ↓
CV-X-IF
        ↓
input buffer
        ↓
16-way INT8 MAC
        ↓
local block accumulator
        ↓
final rd writeback
```

## Input-Buffer Modes

The software configures the number of active 16-byte blocks through the
`rd` field of `BUF4`:

``` text
Conv1 -> 1 block
Conv2 -> 25 blocks
FC1   -> 24 blocks
FC2   -> 9 blocks + 6 scalar inputs
```

The exact implementation and line numbers are documented in
`USER_GUIDE_MAC16_INPUT_BUFFER.md`.
