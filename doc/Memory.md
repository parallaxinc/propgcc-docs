---
layout: doc
title: Memory Models
---

The Propeller port of GCC supports a number of memory models, or ways of
storing the program in memory. Basically these models provide a trade
off of speed for code space. By far the fastest model is the native
“cog” model (`-mcog`) in which machine instructions are executed
directly. However, in that model only the 2K of internal memory
(actually slightly less) is available for code. In the other models
(LMM, CMM, XMMC, and XMM) code is stored in RAM external to the cog and
is loaded in by a small kernel. This makes more space available for the
code (and, in the XMM case, the data) but at the cost of having kernel
overhead on each instruction executed.

COG
---

As previously mentioned, in the COG memory model code is stored directly
in the internal memory of the Propeller's cogs. This memory is 512
words, or 2048 bytes, long, but 16 words are reserved for hardware
registers. The GNU C compiler reserves an additional 17 words for
compiler registers. These registers, named `r0`-`r14`,` lr`, and `sp`,
are used by the compiler for various purposes. `r0`-`r7` are temporary
registers within a function (not preserved across function calls).
`r8`-`r14` are working registers within a function and are preserved
across function calls. `lr` is the “link register” which holds the
return address for functions (except for functions declared with the
“native” attribute, for which the return address is stored directly in
the `ret` instruction). `sp` is the stack pointer.

### Assembly programming in COG mode

Inline assembly (or writing external C callable functions in assembly)
in COG mode is very straightforward – the Propeller instruction set is
directly usable. The GNU assembler gas accepts input that is very
similar to PASM. The major differences are: (1) gas does not understand
local labels (labels that start with `:`), and (2) by default addresses
are considered to be byte addresses rather than word addresses; you can
tell gas to treat them as words by using the `.cog_ram` directive. The
instruction set is the same, although for convenience gas offers a
`mova` (“move address”) instruction that works just like `mov` but which
divides any immediate source value by 4, thus converting it from a byte
address to a word address. `mova` is not needed in .cog\_ram mode, but
is convenient in the default mode.

LMM
---

In the Large Memory Model (LMM mode), the program code is stored in hub
memory, which is 32K in size. The program's data and stack are also
stored there. A small kernel program (the “LMM Kernel”) runs in COG
memory and loads and executes program instructions from hub memory. In
this mode the same 17 compiler registers are available as in hub mode
(`r0`-`r14`, `lr`, and `sp`), but there is an additional register `pc`
which is a pointer to the next instruction to be fetched.

### Assembly programming in LMM mode

In LMM mode most PASM instructions can be used, except for any jump or
call instructions (including `djnz`, `tjnz`, and `jmpret`). Instead of
doing direct jumps, we have to modify the value of the `pc` register
used by the LMM interpreter instead. This may be done either by directly
adding an offset to `pc`, or by loading a new value into it. gas
provides a pseudo-instruction `brs` (“branch short”) which translates
into an immediate `add` or `sub` of the `pc`; this may be used to branch
to a destination within 508 bytes (plus or minus) of the instruction
following the `brs`. For longer branches we can use the `__LMM_JMP`
routine built into the kernel, or else move a different register into
`pc` (for an indirect jump). After the `jmp #__LMM_JMP` instruction
comes a 4 byte address indicating the new value for the `pc`.

Calls are handled by the `__LMM_CALL` and `__LMM_CALL_INDIRECT` kernel
functions. `__LMM_CALL` is like `__LMM_JMP` except that before it loads
the new `pc` it saves the old value into the `lr` register. The called
function can thus return with a simple `mov pc,lr` instruction.
`__LMM_CALL_INDIRECT` uses a special register, `__TMP0`, as the address
to call; thus, an indirect call via compiler register `r6` would be
coded as:

    mov __TMP0, r6
    jmp #__LMM_CALL_INDIRECT

`__LMM_CALL_INDIRECT` also saves the return address in `lr`.

XMMC
----

In the eXtended Memory Model – Code (XMMC mode) the program code is
stored in external memory, either flash or ram. The exact size of this
memory depends on the board used, but it is generally quite a bit larger
than hub memory. The program's data and stack reside in hub memory as in
LMM mode.

Assembly programming in XMMC mode is exactly the same as in LMM mode.

XMM
---

In the eXtended Memory Model both the code and data are placed in
external memory (in the case of data it must be an external RAM). This
allows for the largest possible programs, but at a considerable cost in
execution time, since all data accesses must go through functions in the
XMM kernel. As in all other modes, the stack remains in hub memory (and
hence is limited to 32K bytes less the cache size used by the XMM
kernel, which is typically 8K bytes).

Assembly programming in XMM mode is very similar to LMM mode, except
that data accesses which may point into external memory must be done via
the appropriate kernel functions instead of directly with `rdlong` and
`wrlong` instructions.

CMM
---

There is a special memory model called the "compressed memory model"
(CMM). This model is the same as the LMM model in that code and data are
both stored in hub memory. The difference is that in CMM mode code is
compressed in a special format that takes less space, and decompressed
by the kernel at run time. This saves space, but at the cost of
execution speed.

Kernel APIs
===========

The kernels all provide some common functions which may be used by
programs (and which the compiler takes advantage of).

Note that the GAS assembler has macros built in which provide a "short
form" of some of the calls below. In the compressed memory model (if
-mcmm is used) these short forms are actualy stored compressed in the
instruction stream.

Data movement
-------------

    jmp #__LMM_MVI_rn
    long val

This is a “move immediate” instruction to move the value “val” into
compiler register `rn` (or the link register `lr`). After the
instruction is complete the LMM interpreter resumes automatically.

short form:

    mvil rn, #val

or

    mviw rn, #val
    
if val is known to fit in 16 bits

    mov __TMP0, #(count<<4)|regnum
    call #__LMM_PUSHM

Push multiple compiler registers onto the stack. “count” is the number
of registers to push, and “regnum” is the number of the first register
to push (0 for `r0`, 1 for `r1`, and so on; register number 15 is `lr`).
Note that “count+regnum” should be less than or equal to 16. The
registers count up, so if regnum is 12 and count is 3 then `r12`, `r13`,
and `r14` will be pushed (in that order).

short form: 

    pushm #(count<<4)|regnum

    mov __TMP0, #(count<<4)|regnum
    call #__LMM_POPM   or   call #__LMM\_POPRET

Pop multiple compiler registers from the stack. “count” is the number of
registers to pop, and “regnum” is the number of the first register to
pop, which should be the last register pushed (0 for `r0`, 1 for `r1`,
and so on). Note that “count+regnum” should be less than or equal to 16.
The registers count down, so if regnum is 14 and count is 3 then `r14`,
`r13`, and `r12` will be popped (in that order).

If `__LMM_POPRET` is used then a `mov pc,lr` will be issued right after
the pop

short form:

    popm #(count<<4)|regnum

or 

    popret #(count<<4)|regnum

Branches and calls
------------------

    jmp #__LMM_JMP
    long addr

Moves the immediate value `addr` into the `pc` register (so the LMM
interpreter will begin to execute at address `addr`, thus performing an
unconditional jump).

short form:

    brl #addr

    jmp #__LMM_CALL
    long addr

Moves the address of the next instruction into the `lr` register, and
moves `addr` into the `pc` register (so the LMM interpreter will begin
execution at address `addr`, which is typically a subroutine).

short form:

    lcall #addr

    mov __TMP0,rn
    jmp #__LMM_CALL_INDIRECT

Moves the address of the next instruction into the `lr` register, and
moves `__TMP0` into the `pc` register (so the LMM interpreter will begin
execution at address, it contained, which is typically a subroutine).

Math functions
--------------

    call #__UDIVSI

Performs 32 bit unsigned division of the value in compiler register `r0`
by the one in `r1`. Returns the quotient in `r0`, and the remainder in
`r1`.

    call #__DIVSI

Performs a 32 bit signed division of `r0` by `r1`. Returns the quotient
in `r0`, and the remainder in `r1`. The sign of the remainder is chosen
so as to be consistent with the C standard.

    call #__MULSI

Multiplies the two 32 bit numbers in `r0` and `r1`, and returns the (low
order) 32 bits of the result in `r0`.

    call #__CLZSI

Counts the number of leading 0 bits in register `r0`, and returns the
result in `r0`. For example, if `r0` = 0x00008100, then the result will
be 16. This is useful for normalizing fixed and floating point numbers.

    call #__CTZSI

Counts the number of trailing 0 bits in register `r0`, and returns the
result in `r0`. For example, if `r0` = 0x00008100, then the result will
be 8.

Miscellaneous functions
-----------------------

    jmp #__LMM_FCACHE_LOAD
    long nbytes

Loads some code into the fast cache and executes it. The code to be
loaded and executed immediately follows the load sequence, and is
`nbytes` bytes long (which must be a multiple of 4). Code executing in
the fast cache must use only COG instructions, and must not refer to the
program counter `pc` (`jmp` instructions must be used instead, with
labels referring to the COG memory region starting at
`__LMM_FCACHE_START`, which is where it will be executed from. At the
end of the fastcached block should come a `jmp __LMM_RET` instruction
(note that this is an indirect jump).

For example, the C strcpy function:

    char *strcpy(char *dst_orig, const char *src) {
        char *dst = dst_orig;
        while ((*dst++ = *src++) != 0)
            ;
        return dst_orig;
    }

is translated by the compiler into code using the fast cache like:

            .global _strcpy
    _strcpy
            mov     r7, r0
            jmp     #__LMM_FCACHE_LOAD
            long    .L6-.L5
    .L5
    .L2
            rdbyte  r6, r1
            cmp     r6, #0 wz
            add     r1, #1
            wrbyte  r6, r7
            add     r7, #1
            IF_NE   jmp #__LMM_FCACHE_START+(.L2-.L5)
            jmp     __LMM_RET
    .L6
            mov     pc,lr
