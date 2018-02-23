# Utgard GP

The GP (used for vertex shaders on Utgard) has a scalar VLIW architecture. Each instruction has a field for 2 addition ALU's, 2 multiplication ALU's, a complex ALU, a passthrough ALU, an attribute load unit, a register load unit, a uniform/temporary load unit, and a varying/register/temporary store unit. Instructions are fixed-length - each instruction consists of 4 words. Constants are implemented internally by uniforms.

Unlike a normal CPU, there are no explicit output registers for the ALU's, nor are there any explicit input registers. Instead, the input field(s) for each ALU can directly reference the ALU results from previous instructions (see below). However, there are 16 registers that can be used when two instructions cannot be scheduled so that one references the result of the other (either directly, or through one or more passthroughs), or for special cases such as loops. Only one (4-component) register may be loaded & stored per instruction, and storing registers and temporaries shares some of the same fields as storing varyings.

# Temporaries

Akin to the fragment shader, there are also temporaries, which unlike registers can be indexed using a base register/input. They share the same namespace and method of loading as uniforms. Storing temporaries uses the same fields as storing a register/varying does, except that the "temporary store flag" is enabled, the unknown field is changed, and the complex ALU is used to select the store address instead of the "varying/register store 0" and "varying/register store 1" fields, which are set to 0.

## Output Transformation

gl_Position is implemented internally as a varying; it seems that it is hard-coded to varying 0. The compiler implements some transforms internally to convert the value calculated for gl_Position in the shader to the actual value sent to the hardware. In particular (in pseudocode):

    uniform vec4 gl_mali_ViewportTransform[2];
    gl_position_actual.w = clamp(1.0 / gl_Position.w, -1e10, 1e10);
    gl_position_actual.xyz  = gl_Position.xyz * gl_position_actual.w * gl_mali_ViewportTransform[0].xyz + gl_mali_ViewportTransform[1].xyz;

gl_PointSize is also implemented internally as a varying. However, its position doesn't appear to be fixed. There are also some transforms involved:

    uniform vec4 gl_mali_PointSizeParameters;
    gl_PointSize_actual = clamp(gl_PointSize, gl_mali_PointSizeParameters.x, gl_mali_PointSizeParameters.y) * gl_mali_PointSizeParameters.z;

## Complex functions

Complex functions are implemented using multiply ALU opcodes 1 and 3, as well as various complex ALU opcodes. The computation looks like this:

    output = mul.op1(complex(input), mul.op3(input, input), complex(input), input)

for exp2, it looks like this (note the addition of pass.op4):

    output = mul.op1(complex.exp2(pass.op4(input)), mul.op3(pass.op4(input), pass.op4(input)), complex.exp2(pass.op4(input)), pass.op4(input))

and finally for log2 it looks like this:

    output = pass.op5(mul.op1(complex.log2(input), mul.op3(input, input), complex.log2(input), input))

it would seem that pass.op5 performs the opposite of pass.op4.

## Inputs to the ALU's

These are the known inputs:

    0-3:   Register 0 Output [0, current] (Register/Attribute)
    4-7:   Register 1 Output [0, current] (Register)
    8:     Unused, same as 21? (seen in m200_hw_workarounds.c nop shader)
    9-11:  Unknown
    12-15: Load Result [0, current] (Uniform/Temporary)
    16,17: Accumulator 0,1 Output [-1, last instruction]
    18,19: Multiplier 0,1 Output [-1, last instruction]
    20:    Passthrough Output [-1, last instruction]
    21:    Unused/nop (i.e. this ALU is not used during this instruction)
    22:    Complex Output [-1, last instruction]
    22:    Identity/Passthrough (0 for add, 1 for multiply)
             Accumulator 0,1 Input 1: add(a, -ident) means pass(a)
             Multiplier  0,1 Input 1: mul(a,  ident) means pass(a)
    23:    Passthrough Output [-2, two instructions ago]
    24,25: Accumulator 0,1 Output [-2, two instructions ago]
    26,27: Multiplier 0,1 Output [-2, two instructions ago]
    28-31: Register 0 Output [-1, last instruction] (Register/Attribute)
Note: If attribute_load_en is disabled then the attribute slot can be used to load registers too.

## Latencies

Temporaries have a latency of 4 instructions, i.e. writes take 4 cycles to appear. Registers have a similar latency of 3 instructions. Writes to address registers 1-3 have a latency of 4 instructions. Writes to address register 0 (temporary store) have no latency though, so it can be set in the same instruction as the temporary store itself. The complex1 operation has a latency of 2 cycles.

Instruction format:

    0-4:   Multiply 0 Input A
    5-9:   Multiply 0 input B
    10-14: Multiply 1 Input A (Wide-Operation Input C)
    15-19: Multiply 1 Input B (Wide-Operation Input D)
    20:    Multiply 0 Output Negate
    21:    Multiply 1 Output Negate
    22-26: Accumulator 0 Input A
    27-31: Accumulator 0 Input B
    32-36: Accumulator 1 Input A
    37-41: Accumulator 1 Input B
    42:    Accumulator 0 Input A Negate
    43:    Accumulator 0 Input B Negate
    44:    Accumulator 1 Input A Negate
    45:    Accumulator 1 Input B Negate
    46-54: Load Address (Uniform/Temporary)
    55-57: Load Offset (Uniform/Temporary)
        0   - Address Register 0? (Never seen)
        1   - Address Register 1
        2   - Address Register 2
        3   - Address Register 3
        4-6 - Unknown (Never seen)
        7   - Unused (No offset)
    58-61: Register 0 Address (Register/Attribute)
    62:    Register 0 Attribute (Load attribute in Register 0 unit)
    63-66: Register 1 Address
    67: Store 0 Temporary (Store Temporary in Store 0)
    68: Store 1 Temporary (Store Temporary in Store 1)
    69: Branch
    70: Branch Target Low (< 0x100)
    71-73: Store 0 Input X (Register/Varying/Temporary)
    74-76: Store 0 Input Y (Register/Varying/Temporary)
    77-79: Store 1 Input Z (Register/Varying/Temporary)
    80-82: Store 1 Input W (Register/Varying/Temporary)
        0 - Accumulator 0 Output
        1 - Accumulator 1 Output
        2 - Multiplier 0 Output
        3 - Multiplier 1 Output
        4 - Passthrough Output
        5 - Unknown
        6 -  Complex Output
        7 - Unused (Don't store)
    83-85: Accumulator (0 & 1) opcode
        0 - add
        1 - floor
        2 - sign
        3 - unknown
        4 - greater-equal/step (a >= b)
        5 - less-than (src0 < src1)
        6 - min/logical and (a && b)
        7 - max/logical or (a || b)
        note: abs(a) is implemented as max(a, -a)
    86-89: Complex OpCode
        0 - unused
        2 - exp2 (Partial)
        3 - log2 (Partial)
        4 - inverse sqrt (Partial)
        5 - reciprocal (Partial)
        9 - passthrough
        10 - Set Address Register 0 & Address Register 1 from result of passthrough unit
        12 - Set Address Register 0 (Temporary Store address)
        13 - Set Address Register 1
        14 - Set Address Register 2
        15 - Set Address Register 3
    90-93: Store 0 Address (Varying/Register/Temporary)
    94:    Store 0 Varying (Store Varying in Store 0)
    95-98: Store 1 Address (Varying/Register/Temporary)
    99:    Store 1 Varying (Store Varying in Store 1)
    100-102: Multiply (0 & 1) OpCode
        0 - multiply (out = a * b)
        1 - complex 1 (inverse, inverse sqrt, etc.)
            takes all four inputs as arguments
            This instruction has a latency of 2 cycles.
        3 - complex 2 (inverse, inverse sqrt, etc.)
            takes first two inputs as arguments,
            the other two are normal (multiply)
        4 - select (out = (b ? a : c), wide operation)
        5-7: unknown
    103-105: Passthrough OpCode
        2 - passthrough (out = in)
        6 - clamp (out = max(min(in, uniform.x), uniform.y))
        0-1,3-5,7: unknown
    106-110: Complex Input
    111-115: Passthrough Input
    116-119: Unknown (Treating as flags)
        0 - Normal
        12 - Temporary Write
        13 - Branch
    120-127: Branch Target (Absolute address, bit 9 is  inverse of Branch Target Low)



## Lima Vertex Pipeline
[<img src="http://img857.imageshack.us/img857/6044/limavertexpipeline.png">](http://img857.imageshack.us/img857/6044/limavertexpipeline.png)
