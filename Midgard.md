# Utgard Architecture

## Publicly available information

ARM has been a lot more open this time with the architecture behind the T6xx. For a good overview with some slides from ARM, see [this Anandtech article](http://www.anandtech.com/show/6136/arm-announces-8core-2nd-gen-malit600-gpus). T6xx is the first Mali unified architecture; unlike the Mali 200/400, the vertex and fragment shaders use the same pipelines. There are 3 separate pipelines: ALU, Load/Store, and Texture lookup (A, L, and T in the verbose output of the compiler). The Mali-600 target for the compiler (T604, T622, T624, T628) has 2 ALU's and so can excecute 2 ALU ops per cycle, and the T650 target (T658, T678) has 4 ALU's.

## Patents

[Data processing apparatus and method for processing a received workload in order to generate result data](https://www.google.com/patents/US20120304194?hl=en&sa=X&ei=lpT0UZP-MPjk4AOZu4HwDg&ved=0CDsQ6AEwAQ)

[Processing order with integer inputs and floating point inputs](https://www.google.com/patents/US20120299935?hl=en&sa=X&ei=lpT0UZP-MPjk4AOZu4HwDg&ved=0CDQQ6AEwAA)

[Floating-point vector normalisation](https://www.google.com/patents/WO2012038708A1?cl=en&hl=en&sa=X&ei=lpT0UZP-MPjk4AOZu4HwDg&ved=0CEIQ6AEwAg)

[Vector floating point argument reduction](https://www.google.com/patents/US20120078987?hl=en&sa=X&ei=p5r0UbSJHtXh4AOqo4HYCw&ved=0CF0Q6AEwBjgK)

[Next-instruction-type field](https://www.google.com/patents/EP2585906A1?cl=en&hl=en&sa=X&ei=SZn0UdHEOrj54AOOsYHIBg&ved=0CDsQ6AEwAQ)

[Generating and resolving pixel values within a graphics processing pipeline](https://www.google.com/patents/US8059144?hl=en&sa=X&ei=lpP0UcbvDvOt4AP_7IDgAg&ved=0CD0Q6AEwAQ)

[Number format pre-conversion instructions](https://www.google.com/patents/US20120215822?hl=en&sa=X&ei=1pn0UYb8Orix4APMnICQBg&ved=0CFIQ6AEwBA)

[Graphics processing](https://www.google.com/patents/US20120223946?hl=en&sa=X&ei=p5r0UbSJHtXh4AOqo4HYCw&ved=0CEgQ6AEwAzgK)

[Embedded opcode within an intermediate value passed between instructions](https://www.google.com/patents/US20120204006?hl=en&sa=X&ei=p5r0UbSJHtXh4AOqo4HYCw&ved=0CHIQ6AEwCTgK)

## Instruction format

It appears that the shader binaries are the same between the T600 and T650 targets; the only difference is in how many cycles it takes to execute the ALU instructions (T650 takes half as many due to having twice as many ALU's). The shader consists of a stream of instructions. There are 3 types of instruction words, corresponding to the three pipelines; instruction words can appear in any order. Each instruction is always started only after the previous instruction has fully completed, and like on the Mali 200 PP the pipeline is barreled so a number of threads, potentially with different shaders, are running at once (see next-instruction-type patent). Instruction words are always a multiple of 4 words (128 bits). They can be parsed by reading the type from lowest 4 bits of the first word:

    3 - Texture (4 words)
    5 - Load/Store (4 words)
    8 - ALU (4 words)
    9 - ALU (8 words)
    A - ALU (12 words)
    B - ALU (16 words)

The next 4 bits (bits 4-7) store the type of the next instruction (presumably for prefetch purposes), except if either the instruction is the last instruction or it's the second-to-last and the last instruction is an ALU instruction, in which case the value of 1 is used.

## Comparison to Mali-200 PP

It seems like the architecture for the T6xx pipeline is based off of the Mali-200 PP pipeline. Both are barreled with a relatively deep pipeline that can execute a number of threads, possibly with different shaders/uniforms/other state at the same time. The main difference is that the single, large pipeline is broken down into 3 smaller pipelines. This simplifies the logic; the ALU pipelines don't need to know how to access memory, the Load/Store and texture pipelines don't need to access work registers or uniform registers, and only the texture pipeline needs to have logic for synchronizing threads in a group and exchanging values for computing derivatives. There are more work registers, and now there's a uniform register file, in addition to normal uniform buffers accessed through the load/store pipeline, and perhaps a Radeon-like register sharing mechanism (the compiler now reports the number of work registers and uniform registers used)

## Registers

The ALU pipeline can read/write to 32 128-bit registers, which can be divided into 4 32-bit (highp in GLSL) components (one vec4) or 8 16-bit (mediump) components (two vec4's). Some of the registers, however, are dedicated to special purposes (see below) and are read-only or write-only.

## Special Registers

    r24 - can mean "unused" for 1-src instructions, or a pipeline register
    r26 - inline constant
    r27 - load/store offset when used as output register
    r28-r29 - texture pipeline results
    r31.w - conditional select input when written to in scalar add ALU

r0 - r23 is divided into two spaces: work registers and uniform registers. A configurable number of registers can be devoted to each; if there are N uniform registers, then r0 - r(23-N) are work registers and r(24-N)-r23 are uniform registers.

## ALU Words

The first (32-bit) word is a control word which, in addition to the usual 8-bit tag in bits 0-7, contains a bitfield describing which ALU's in the pipeline are in use. There are 5 ALU's, in addition to a framebuffer write (?) and branch unit.

    0-3: instruction word type (0x8-0xB)
    4-7: next instruction word type
    17: vector multiply ALU (48 bits)
    19: scalar add ALU (32 bits)
    21: vector add ALU (48 bits)
    23: scalar multiply ALU (32 bits)
    25: LUT / multiply ALU 2 (48 bits)
    26: compact output write/branch (16 bits)
    27: output write/branch (48 bits)

It's not clear why only every other bit is used for the ALU's (fp64?).

After the control word comes a series of 16-bit words, one for each enabled ALU (up to 5) which control the input and output registers for each ALU. After that come the actual fields for each ALU/unit, whose sizes are noted in the table above. The instruction word is then padded with 0's to make sure it is a multiple of 4 words. Finally, embedded constants may be inserted, which consist of 4 32-bit numbers, interpreted as 4 IEEE 32-bit floats if the input is a float.

### Register word format

    0-4: input 1 register
    5-9: input 2 register / inline constant
    10-14: output register
    15: input 2 inline constant

The register 2 inline constant is a way to store a 16-bit float directly in the instruction. The upper 5 bits (15-11) are stored where the input 2 register would normally go, and the lower 11 bits (0-10) are stored in the ALU field as defined below. The constant is splattered across all 4 components of the input. This is much more compact than the normal embedded constants, but much more limited as well.

### Vector ALU word format

The vector multiply, add, and LUT ALU's share the same instruction format.

    0-7: opcode
    8-9: input/output mode
        1 - half (16-bit)
        2 - full (32-bit)
    10: input 1 abs
    11: input 1 neg
    if input/output mode is half:
        12: input 1 replicate lower half-register
        13: input 1 replicate upper half-register
    otherwise:
        12: input 1 half-register selection (high or low)
        13: unused
    14: input 1 half-register (when output is a full register)
    15-22: input 1 swizzle
    23: input 2 abs
    24: input 2 neg
    if "input 2 inline constant" set:
        25-35: input 2 inline constant low 11 bits
        25-27: inline const 8-10
        28-35: inline const 0-7
    otherwise:
        if input/output mode is half:
            25: input 2 replicate lower half-register
            26: input 2 replicate upper half-register
        otherwise:
            25: input 2 half-register selection (high or low)
            26: unused
        27: input 2 half-register (when output is a full register)
        28-35: input 2 swizzle
    36-37: output size override
        0 - half, write to lower half
        1 - half, write to upper half
        2 - normal
        Note: I've only seen this for comparison instructions that compare two full floats or ints and need to return a half float
    38-39: output modifier
        0 - none
        1 - clamp positive
        2 - output integer
        3 - saturate
    40-47: write mask
        2 bits for each output when 32-bit, 1 bit when 16-bit

When the register mode is set to half, the operation is performed on the high and low half-registers at the the same time. The low 4 bits of the write mask control what components of the low half-register are written, and the high 4 bits control the high half-register. Normally, the operation is performed on the input 1 low register and input 2 low register to produce the output low register, and on the input 1 high register and input 2 high register to produce the high register. This can be overwritten, however, by the "input 1/2 replicate lower/upper half-register" bits which cause the given half-register to be used as an input to both operations at once.

### Scalar ALU word format

The scalar multiply and add ALU's have the same format as well.

    0-7: opcode
    8: input 1 abs
    9: input 1 negate
    10: input 1 size (0 = half, 1 = full)
    if input 1 size = full
        11: unused
        12-13: input 1 component
    otherwise:
        11-12: input 1 component
        13: input 1 half-register selection (high or low)
    if "input 2 inline constant" set:
        14-24: input 2 inline constant low 11 bits
        14-15: inline const 9-10
        16: inline const 8
        17-19: inline const 5-7
        20-24: inline const 0-4
    otherwise:
        14: input 2 abs
        15: input 2 negate
        16: input 2 size (0 = half, 1 = full)
        17-18: input 2 component
        19-24: unknown
    25: unknown
    26-27: output modifier
        0 - none
        1 - clamp positive
        2 - output integer
        3 - saturate
    28: output size
        0 - half
        1 - full
    if output size = full
        29: unused
        30-31: output component
    otherwise:
        29-30: output component
        31: output half-register selection (high or low)

### Opcodes

    10 - fadd
    14 - fmul
    28 - fmin
    2C - fmax
    30 - fmov
    36 - ffloor
    37 - fceil
    3C - fdot3
    3D - fdot3r
    3E - fdot4
    3F - freduce
    40 - iadd
    46 - isub
    58 - imul
    7B - imov
    80 - feq
    81 - fne
    82 - flt (less than)
    83 - fle (less than or equal)
    99 - f2i
    A0 - ieq
    A1 - ine
    A4 - ilt
    A5 - ile
    C5 - csel (conditional select)
    B8 - i2f
    E8 - fatan_pt2
    F0 - frcp (reciprocal)
    F2 - frsqrt (inverse square root, 1/sqrt(x))
    F3 - fsqrt (square root)
    F4 - fexp2 (2^x)
    F5 - flog2
    F6 - fsin
    F7 - fcos
    F9 - fatan_pt1
        Note: for sin and cos, the input needs to be divided by pi


Pseudocode for how atan/atan2 is implemented:

    vec4 temp1.xzw = fatan_pt1(x, y); //Note: a vec4 temporary is required, although the write mask is xzw so the y component isn't affected
    float result = fatan_pt2(temp1.x, temp1.z * temp1.w);

To do atan instead of atan2, replace y with 1.0. asin and acos are implemented just like in the Mali 200 PP.

### Compact branch/framebuffer

This field is used for encoding branches, as well as framebuffer writes. So far, we've only figured out branches. The branch offsets possible are rather limited. If you need a greater offset, you need to use the normal encoding instead. The bottom three bits are the opcode:

    001 - unconditional branch
    010 - conditional branch
    111 - write to framebuffer

The rest of the 16-bit word is different depending on the opcode. For conditional branches, it looks like this:

    0-2: opcode (=010)
    3-6: Type of instruction to branch to
        Note: this complements the next-instruction-type field that every instruction has, which gives the type of the next instruction to execute if there is no branching.
    7-13: Offset to branch to
        This gives the low 7 bits of the offset in units of quadwords (16 bytes) relative to the next instruction that would be executed. The offset is sign-extended to the full range.
    14-15: Branch condition
        01 - branch if r31.w is false
        10 - branch if r31.w is true

For unconditional branches, it looks like this:

    0-2: opcode (=001)
    3-6: Type of instruction to branch to
        Same as conditional branches
    7-8: unknown
        Always 01 so far.
    9-15: Offset to branch to
        Unlike for conditional branches, this is zero extended. Only positive offsets are possible. For negative offsets, use conditional branches with an always-true condition. Yes, really.

## Load/store words

The load/store word consists of the standard 8-bit tag, followed by two 60-bit instructions whose format is described below. Each instruction can load or store up to 128 bits at once.

    0-7: opcode
        03 - noop (no load/store)
        94 - load attribute (32-bit)
        95 - load attribute (16-bit)
        98 - load varying (32-bit)
        99 - load varying (16-bit)
        AC - load uniform (16-bit)
        B0 - load uniform (32-bit)
        D4 - store varying (32-bit)
        D5 - store varying (16-bit)
    8-12: source/destination register
    13-16: mask
    17-24: swizzle
    25-50: unknown
    51-59: load/store address

The mask and swizzle acts like a move instruction. For example, a load with a mask of xzw and a swizzle of xywz means "take the x, w, and z components of the input and move them into the x, z, and w components of the register respectively."

TODO: indirect access

TODO: uniform buffers
