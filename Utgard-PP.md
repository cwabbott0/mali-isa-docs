# PP architecture

The architecture consists of a large pipeline, consisting of a number of vector and scalar units which can be enabled and disabled by the control word. In particular, there are two vector ALU's, one of which can do addition, another which can do multiplication.  There are also two similar scalar ALU's, and the "combiner" capable of executing scalar-vector multiplies as well as various complex/transcdental functions (sqrt, rcp, sin, cos, exp2, etc.). Furthermore, there are varying, uniform/temporary, and texture load units, a temporary store unit, and a branching unit for implementing jumps. Each unit can affect/produce results which are used by all later units, as if all 6 registers are passed between each unit in the pipeline (see "Lima Fragment Pipeline" below). The only exception is that each vector ALU unit is executed in parallel with its scalar counterpart - so the vector and scalar multiply ALU's run in parallel, and so do the vector and scalar addition ALU's. Furthermore, to reduce register pressure, there are a number of "pipeline registers". A pipeline register is a direct connection between two units in the pipeline, in addition to the normal registers which are passed between every unit. For more details on registers (including pipeline registers), see the "Registers" section below. To overcome the pipeline stall issues inherent in such a long pipeline (128 stages for Mali-200, see [this page](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka12787.html)), the architecture is likely barrelled and interleaves execution of a large number of fragments at once, and scheduling is done by the machine in order to minimize stalls.
The instruction stream is compressed down from a maximum of 18-words per instruction dependant on what units are in use.
The remaining bits give each unit individual instructions and constants.

The encodings are variable length 32-bit aligned.

## Control Word Encoding

Fragment shaders begin with a 32-bit control word.
The control word dictates what units will be used in each instruction.
The lowest 5 bits of the control word determines the number 32-bit DWORDs in the entire instruction "word" (including the control word).

## Speculation on derivatives

Apparently, most GPU's processes fragments in groups of 2x2; I suspect ours does this as well. Normally, each shader in the group is contained from one another, except for derivatives. Derivatives are an extension in gles 2.0 (OES_standard_derivatives), however Mali does support it. Usually, derivatives are accomplished by each pixel swapping the value passed into the function with either it's vertical (dFdy) or horizontal (dFdx) neighbor in the group, and then computing the difference with it's own value. Furthermore, all 4 shaders have to execute this swap-and-difference instruction at once; I believe control bit 6 synchronizes the shaders (I'm not sure what happens when each shader takes a different side of a branch).  Bit 6 is enabled for texture fetching because the processor computes the level of detail by computing the screen-space derivative of the texture coordinates, and therefore needs to compute derivatives as part of fetching the texture.

### Bitfield Description

     0...4: {  } Instruction Length
     5:     {0 } Output to gl_FragColor and end program.
     6:     {0 } Inter-thread synchronization?
     7:     {34} Varying Fetch
     8:     {62} Texture Sampler
     9:     {41} Uniform/Temporary Fetch
    10:     {43} Vec4 Multiply ALU
    11:     {30} Scalar Multiply ALU
    12:     {44} Vec4 Addition ALU
    13:     {31} Scalar Addition ALU
    14:     {30} Vec4-Scalar Multiply/Transcendental Scalar ALU
    15:     {41} Temporary Write/Framebuffer Read
    16:     {73} Branch/Discard
    17:     {64} Vec4 Constant Fetch 0
    18:     {64} Vec4 Constant Fetch 1
    19..24: {  } Next Instruction Length (prefetch)
    25:     {  } Prefetch enable
    28..31  {  } Unknown (=0)
    
    
    {n} means this bit set adds n bits to the instruction 

Prefetch enable is set for all instructions which can possibly reach the next instruction, i.e. all instructions except for the last instruction and discard instructions. For all instructions, the next instruction length is set to the instruction length of the next instruction (duh) except for the last instruction where it's set to 0; presumably this is for prefetching, since it will work if you set a too high value for this field but it will crash if you set a value too low. If you add the n's together for each enabled unit, and then round upward to the nearest 32-bit word, (+1 word for the control word) they should equal the bottom 5 bits.


### Vector Swizzling
These 8 bits encode how to swizzle a 4-component vector in the vec4 addition and multiplication opcodes.The bits are grouped into 4 groups of 2. Bits 0 and 1 tell what goes in the first element, 2 and 3 what goes in the second element, etc. For example: 

     00 = 00 00 00 00 = .xxxx
     04 = 00 00 01 00 = .xyxx
     10 = 00 01 00 00 = .xxyx
     14 = 00 01 01 00 = .xyyx
     24 = 00 10 01 00 = .xyzx
     40 = 01 00 00 00 = .xxxy
     44 = 01 00 01 00 = .xyxy
     50 = 01 01 00 00 = .xxyy
     54 = 01 01 01 00 = .xyyy
     E4 = 11 10 01 00 = .xyzw (identity)


## Opcode formatting

The opcodes are placed in the order of the bits in the control word: Varying loading (bit 5) will always come first if enabled, followed by texture sampling (bit 6), then uniform loading (bit 7), etc.

To encode the opcode for each enabled unit in the control word, start with bit 0 of the first word. Then, for each unit, append the opcode to the current word. If the opcode overflows the current word, then start at bit 0 the next one.

Note that, because the bits are numbered right-to-left but the words are numbered left-to-right, this means that the opcodes usually look "broken" when looking at the instruction word-for-word. For example, a varying load followed by some other opcode might look like this:

aaaa aaaa aaaa aaaa aaaa aaaa aaaa aaaa  bbbb bbbb bbbb bbbb bbbb bbbb bbbb bbAA ...

where the varying load opcode is actually:
AA aaaa aaaa aaaa aaaa aaaa aaaa aaaa aaaa

Note that bits 32-33 of the opcode "overflowed" into bits 0-1 of the next word.

##Registers

It appears there are 6 vec4 registers actually used out of up to 12 possible, which are encoded as four bits in a control field. The last 4 registers are read-only, "pipeline registers" and are hardcoded as:

    12 - Vec4 constant 0
    13 - Vec4 constant 1
    14 - Texture sampler result
    15 - Uniform fetch result

Also, each of these registers can be divided into 4 scalar registers when used as scalar inputs or outputs, for 24 scalar registers and 16 read-only special registers, which are encoded as six bits in a control field. Furthermore, when these registers run out there is a larger number of "temporary variables", which can be read using the uniform/temporary fetch (bit 9) and written using the temporary write (bit 15).  When writing to a vector register, you can specify a "write mask" or "output mask". Conceptually, all 4 components of the result are calculated, but only the components specified by the write mask are actually written to the register.

There also exists various "pipeline registers" (four of them listed above) which are only used within one particular instruction (i.e. within the pipeline) and are used as the inputs/outputs of various units. The pipeline registers can be grouped into:

* Constants
* Texture Sample registers
* Uniform fetch register
* ALU registers (^vmul, ^fmul or ^v0, ^s0)

## Deciphered Opcodes

    control[7], Varying Fetch
    
    00 mmmm dddd iiii iiOO 00oo oo00 0aa0 ssss
    Or, for loading from a register (used for loading texture coordinates from a register):
    00 mmmm dddd SSSS SSSS ANrr rr00 0000 01pp
    Or, for normalizing a vec3 input register (uses most of the same fields as loading from a register):
    00 mmmm dddd SSSS SSSS ANrr rr00 1000 1010
    
    m - Mask, (0001 = float, 0011 = vec2, 0111 = vec3, 1111 = vec4)
    d - Destination Register
        Note: writing to register 15 discards the output (used for loading texture coordinates)
    i - Varying Index
    a - alignment
        It seems that varyings (floats) can be loaded in aligned groups of 1, 2, or 4.
        This specifies how many to load at once. Note that the alignment affects the addressing;
        for example, loading from an index of x at an alignment of 4 is equivalent to loading from 2*x
         and 2*x+1 at an alignment of 2.
        00 - no alignment (load 1 float)
        01 - alignment by 2 (load 2 floats)
        11 - alignment by 4 (load 4 floats)
    A - absolute value input modifier
    N - negate input modifier
    r - input register
    o - Offset register - vector part
        1111: no offset (0)
    O - Offset register - scalar part
        Note: it seems the offset register is formed by ooooOO.
        However, I haven't been able to test this theory because I haven't gotten the compiler
        to produce a value for O other than 11. 
    s - source:
        00pp - normal varying
        01pp - register (see second instruction format)
        1000 - varying, input to textureCube()
        1001 - register, input to textureCube()
        1010 - vec3 normalize (see third instruction format)
        1011 - gl_FragCoord
        1100 - gl_PointCoord
        1101 - gl_FrontFacing
    S - swizzle descriptor for source
    p - perspective (used for texture2DProj)
        00 - normal
        10 - divide by z
        11 - divide by w

    Note: for gl_FragCoord and gl_PointCoord, the shader has to apply some transforms to get the right value.
    For gl_FragCoord, it looks like this in pseudocode:
    gl_FragCoord.xyz = gl_FragCoord_orig.xyz;
    gl_FragCoord.w = 1.0 / gl_FragCoord_orig.w;

    And for gl_PointCoord:
    uniform vec4 gl_mali_PointCoordScaleBias; //created by the compiler
    gl_PointCoord = gl_mali_PointCoordScaleBias.zw + gl_mali_PointCoordScaleBias.xy * gl_PointCoord_orig;

    control[8], Texture fetch

    00 1110 0100 0000 0000 01ss ssss ssss ssot tttt 0000 0lb0 0000 cccc ccrr rrrr

    The coordinates for the texture fetch are always the output of the varying load.

    s - sampler index
    o - sampler index register offset enable
    c - sampler index offset register
    t - sampler type
        00000 - sampler2D
        11111 - samplerCube
    l - lod register enable
    b - explicit lod
        If true, the LOD register specifies the actual LOD.
        If false, the LOD register specifies an offset applied to the normally-calculated LOD.
    r - lod register

    control[9], Uniform/Temporary Fetch

    i iiii iiii iiii iiio rrrr rr00 0000 aa00 0000 00ss

    i - source index
    a - alignment
        00 - 1-aligned
        01 - 2-aligned
        10 - 4-aligned
    s - source
        00 - uniform
        11 - temporary
    o - register offset enable
    r - offset register
    
    control[10], Vec4 Multiply ALU

    ooo ooMM mmmm dddd CCaa aaaa aaAA AADD bbbb bbbb BBBB

    Everything except the opcode is the same as the vec4 addition ALU.

    Opcode:
        00xxx - arg0 * arg1 * 2^x, where x is in two's-complement format
        01000 - not(arg0)
        01001 - and(arg0, arg1)
        01010 - or(arg0, arg1)
        01011 - xor(arg0, arg1)
        01100 - notEqual(arg0, arg1)
        01101 - lessThan(arg0, arg1)
        01110 - lessThanEqual(arg0, arg1)
        01111 - equal(arg0, arg1)
        10000 - min(arg0, arg1)
        10001 - max(arg0, arg1)
        11111 - arg1 (passthough)

    control[11], Scalar Multiply ALU

    oo oooM Medd dddd AAaa aaaa BBbb bbbb

    o - opcode:
        00xxx - arg0 * arg1 * 2^x where x is in two's complement format
        01000 - not(arg0)
        01001 - and(arg0, arg1)
        01010 - or(arg0, arg1)
        01011 - xor(arg0, arg1)
        01100 - notEqual(arg0, arg1)
        01101 - lessThan(arg0, arg1)
        01110 - lessThanEqual(arg0, arg1)
        10001 - max(arg0, arg1)
        10000 - min(arg0, arg1)
        11111 - arg1 (passthough)
     
    e - output enable, set to 0 when outputting to scalar accumulate (above)   
    d - destination register
    A - argument 0 modifiers, bit 0 is abs() and bit 1 is negate
    a - argument 0 register
    B - argument 1 modifiers
    b - argument 1 register
    M - output modifier:
        00 - passthrough
        01 - saturate - clamp(output, 0.0, 1.0)
        10 - max(0.0, output)
        11 - round to integer

    control[12], Vec4 Addition ALU

    iooo ooMM mmmm dddd CCaa aaaa aaAA AADD bbbb bbbb BBBB
    
    i - whether to get Argument 1 from the vector multiplication ALU (above)
    o - opcode:
        00000 - arg0 + arg1
        00100 - fract(arg1)
        01000 - notEqual(arg0, arg1)
        01100 - floor(arg1)
        01101 - ceil(arg1)
        01011 - equal(arg0, arg1)
        01001 - lessThan(arg0, arg1)
        01010 - lessThanEqual(arg0, arg1)
        01111 - max(arg0, arg1)
        01110 - min(arg0, arg1)
        10000 - sum3 - dest.xyzw = sum of first 3 components of arg1
        10001 - sum4 - dest.xyzw = sum of all components of arg1
            Note: for sum3 and sum4, the output is broadcast to all channels - 
            you can use the write mask to select which component to write to
        10100 - dFdx(arg0, arg1)
        10101 - dFdy(arg0, arg1)
            Note: dFdx(x) is actually implemented as dFdx(-x, x) (same for dFdy)
            See "Speculation on derivatives" above
        11111 - arg1 (passthrough)
    m - Mask, same as varying fetch opcode
    d - Destination register
    C - Argument 0 modifier
    a - Argument 0 Swizzle descriptor
    A - Argument 0 Source
    D - Argument 1 modifier
    b - Argument 1 Swizzle descriptor
    B - Argument 1 Source
    M - output modifier:
        00 - don't round
        01 - saturate - clamp(output, 0.0, 1.0)
        10 - max(0.0, output)
        11 - round to integer

    Modifiers:
    bit 0 - absolute value
    bit 1 - negate
    
    control[13], Scalar Addition ALU

    ioo ooor r1dd dddd AAaa aaaa BBbb bbbb

    i - whether to get Argument 1 from Scalar Multiply (above)
    o - opcode:
        00000 - arg0 + arg1
        00100 - fract(arg1)
        01100 - floor(arg1)
        01101 - ceil(arg1)
        10100 - dFdx(arg0, arg1)
        10101 - dFdy(arg0, arg1)
            Note: dFdx(x) is actually implemented as dFdx(-x, x) (same for dFdy)
            See "Speculation on derivatives" above
        10111 - sel
            Seems to select between two input values based on ^fmul: (^fmul ? arg1 : arg0).
        11111 - arg1 (passthrough)
    d - destination
    A - argument 0 modifiers, bit 0 is abs() and bit 1 is negate
    a - argument 0 register
    B - argument 1 modifiers
    b - argument 1 register
    M - output modifier:
        00 - passthrough
        01 - saturate - clamp(output, 0.0, 1.0)
        10 - max(0.0, output)
        11 - round to integer
    
    control[14], Vec4-Scalar Multiply/Transcendental Scalar ALU

    dd dddd MMss ssss na00 0000 00oo oo00

    d - destination
    M - output modifier
    s - source
    n - negate source
    a - take absolute value of source
    o - operation:
        0x0 - 0000 - reciprocal (1.0 / x)
        0x1 - 0001 - nop/passthrough?
        0x2 - 0010 - sqrt
        0x3 - 0011 - inversesqrt
        0x4 - 0100 - exp2
        0x5 - 0101 - log2
        0x6 - 0110 - sin
        0x7 - 0111 - cos
        0x8 - 1000 - atan_pt1 (see below)
        0x9 - 1001 - atan2_pt1 (see below)

    Or, for float * vec4:
    dd ddmm mmaa aaaa AAbb bbBB BBBB BB11

    d - destination
    m - mask (for destination)
    a - scalar source
    A - scalar modifier (negative, absolute value)
    b - vec4 source
    B - vec4 source swizzle descriptor

    Note - for sin and cos, the input is multiplied by the constant 1/(2*pi), presumably to
    simplify the hardware.

    atan is divided into two instructions, which we'll call atan_pt1 and atan_pt2.
    atan_pt1 takes the (scalar) input and produces a 3-component vector.
    atan_pt2 takes the vector and produces the final output.

    Unlike atan_pt1, you need to do an additional multiply between atan2_pt1 and atan_pt2:
    $temp.xyz = atan2_pt1 y, x;
    $temp.x *= $temp.y;
    result = atan_pt2 $temp;

    asin and acos are implemented using atan2, as follows:
    asin(x) = atan2(x, sqrt(1 - x^2))
    acos(x) = atan2(sqrt(1 - x^2), x)

    atan_pt1:

    dd ddmm mmaa aaaa AAbb bbbb BBoo oo01
    d - destination (vector)
    m - write/output mask (always 0111 in this case)
    a - scalar 0 source
    A - scalar 0 modifier
    b - scalar 1 source
    B - scalar 1 modifier
    o - opcode
        0x8 - 1000 - atan_pt1(src0)
        0x9 - 1001 - atan2_pt1(src0, src1)

    atan_pt2:

    dd dddd 0000 0000 00aa aaAA AAAA AA10
    d - destination (scalar)
    a - source (vector)
    A - swizzle descriptor

    control[15], Temporary Write/Framebuffer Read

    Temporary Write:

    i iiii iiii iiii iiio rrrr rr00 0000 aass ssss 00dd

    i - destination index
    a - alignment
        0 - float
        1 - vec2
        2 - vec4
    s - source register
        If the alignment is set to vec4, then only the upper 4 bits are used.
    d - destination
        11 - temporary
    o - register offset enable
    r - offset register

    Framebuffer Read:

    0 0000 0000 0000 0000 0000 0000 0000 10dd dd00 11ss

    d - destination register
    s - source
        11 - gl_FBColor
        10 - gl_FBDepth
        Note: since gl_FBDepth is a float, and the alignment is set to 1,
        this instr will always set the x component of the specified destination register.

    control[16], Branch/Discard

    Branch:
    0 0011 tttt tttt tttt tttt tttt tttt ttt0 0000 0000 0000 0000 0000 0ccc aaaa aabb bbbb 0000

    Discard:
    0 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0111 1111 0000 0000 0000 0011

    c - condition:
        bit 0 - jump if a > b
        bit 1 - jump if a = b
        bit 2 - jump if a < b
       The jump will happen if any of the conditions signaled by the appropriate bit is true.
       For example, a condition code of 011 means "jump if a >= b" and 111 is an unconditional jump.

    t - target address, relative to the start of the current instruction

    control[17] and control[18], Vec4 constant fetch

    This opcode simply consists of 4 
    (half-floats)[http://en.wikipedia.org/wiki/Half-precision_floating-point_format]
    to be loaded. bits 0-15 store the first argument, bits 16-31 the second one, etc.


## Lima Fragment Pipeline
[<img src="http://img850.imageshack.us/img850/1056/limapipelinehorizontal2.png">](http://img850.imageshack.us/img850/1056/limapipelinehorizontal2.png)
