# AArch64 Calling Convention
> **Files covered:**
> - `AArch64CallingConvention.cpp` — Custom C++ logic for argument assignment
> - `AArch64CallingConvention.h`   — Declarations of all CC entry points
> - `AArch64CallingConvention.td`  — TableGen rules describing every calling convention

---

## Table of Contents

1. [What is a Calling Convention?](#1-what-is-a-calling-convention)
2. [Why is it Needed?](#2-why-is-it-needed)
3. [The Problem Without a Calling Convention](#3-the-problem-without-a-calling-convention)
4. [What a Calling Convention Defines](#4-what-a-calling-convention-defines)
5. [AArch64 Register Map (Quick Reference)](#5-aarch64-register-map-quick-reference)
6. [How LLVM Implements Calling Conventions](#6-how-llvm-implements-calling-conventions)
7. [File: AArch64CallingConvention.h — Entry Points](#7-file-aarch64callingconventionh--entry-points)
8. [File: AArch64CallingConvention.td — The Rules](#8-file-aarch64callingconventiontd--the-rules)
9. [File: AArch64CallingConvention.cpp — Custom Logic](#9-file-aarch64callingconventioncpp--custom-logic)
10. [All Calling Conventions Defined in This Backend](#10-all-calling-conventions-defined-in-this-backend)
11. [Callee-Saved Register Sets (CSR)](#11-callee-saved-register-sets-csr)
12. [TableGen CC Actions Reference](#12-tablegen-cc-actions-reference)
13. [Full Conceptual Map](#13-full-conceptual-map)
14. [Key Concepts Quick Reference](#14-key-concepts-quick-reference)

---

## 1. What is a Calling Convention?

A **calling convention** is a set of rules that defines how functions
communicate with each other at the machine level.

It answers questions like:

```
- How does the caller pass arguments to the callee?
- Which registers hold the arguments?
- What happens when there are too many arguments for registers?
- How does the callee return a value back to the caller?
- Which registers must the callee preserve?
- Which registers can the callee freely destroy?
- Who cleans up the stack after a call?
```

Without these rules, two separately compiled pieces of code could never
call each other correctly.

---

## 2. Why is it Needed?

### The fundamental problem

A CPU has a fixed set of registers. When function A calls function B,
both functions need to use registers. Without rules:

```
Function A puts argument in x3
Function B looks for argument in x0
-> B reads garbage, program crashes
```

### The solution

Both functions agree on the same rules before compilation:

```
Rule: first integer argument goes in x0
Rule: second integer argument goes in x1
Rule: integer return value comes back in x0
```

Now A puts the argument in x0, B reads from x0, everything works.

### Why it matters for a compiler

The compiler must:

1. When **generating a call**: place arguments in the right registers/stack slots
2. When **generating a function body**: read arguments from the right places
3. When **generating a return**: place the return value in the right register
4. When **generating a prologue**: save registers the callee must preserve
5. When **generating an epilogue**: restore those registers before returning

All of this is driven by the calling convention.

---

## 3. The Problem Without a Calling Convention

Imagine two files compiled separately:

```c
// file_a.c  (compiled by clang)
int add(int x, int y);

int main() {
    return add(3, 4);
}
```

```c
// file_b.c  (compiled by gcc)
int add(int x, int y) {
    return x + y;
}
```

If clang puts `3` in `x0` and `4` in `x1`, but gcc reads `x` from `x1`
and `y` from `x2`, the result is wrong.

The calling convention is the **contract** that makes separate compilation work.
It is also what makes C code callable from Rust, Swift, Python extensions,
assembly, and any other language.

---

## 4. What a Calling Convention Defines

### A. Argument passing

```
Integer arguments:    x0, x1, x2, x3, x4, x5, x6, x7  (up to 8)
Float arguments:      s0, s1, s2, s3, s4, s5, s6, s7   (up to 8)
Double arguments:     d0, d1, d2, d3, d4, d5, d6, d7   (up to 8)
Vector arguments:     v0, v1, v2, v3, v4, v5, v6, v7   (up to 8)
Extra arguments:      pushed onto the stack
```

### B. Return values

```
Integer return:       x0 (or x0+x1 for 128-bit)
Float return:         s0
Double return:        d0
Vector return:        v0
Large struct return:  caller allocates memory, passes pointer in x8
```

### C. Callee-saved registers (non-volatile)

The callee MUST preserve these — if it uses them, it must save and restore:

```
x19, x20, x21, x22, x23, x24, x25, x26, x27, x28
x29 (fp), x30 (lr)
d8, d9, d10, d11, d12, d13, d14, d15
```

### D. Caller-saved registers (volatile)

The caller must save these before a call if it needs them after:

```
x0-x17    (argument/scratch registers)
x18       (platform register, reserved on some OSes)
d0-d7     (argument FP registers)
d16-d31   (scratch FP registers)
```

### E. Stack alignment

```
The stack pointer must be 16-byte aligned at every function call boundary.
```

### F. Special registers

```
x8   = indirect result location register (sret pointer)
x15  = nest parameter (function pointer trampoline)
x16  = intra-procedure-call scratch (IP0)
x17  = intra-procedure-call scratch (IP1)
x18  = platform register (reserved on Windows/iOS)
x29  = frame pointer (fp)
x30  = link register (lr) = return address
```

---

## 5. AArch64 Register Map (Quick Reference)

```
Register   Alt name   Role in AAPCS64
---------  ---------  ------------------------------------------
x0         -          Argument 1 / Return value
x1         -          Argument 2 / Return value (128-bit)
x2         -          Argument 3
x3         -          Argument 4
x4         -          Argument 5
x5         -          Argument 6
x6         -          Argument 7
x7         -          Argument 8
x8         -          Indirect result location (sret)
x9-x15     -          Caller-saved scratch registers
x16        IP0        Intra-procedure-call scratch
x17        IP1        Intra-procedure-call scratch
x18        -          Platform register (reserved on Windows/iOS)
x19-x28    -          Callee-saved registers
x29        fp         Frame pointer
x30        lr         Link register (return address)
xzr/sp     -          Zero register / Stack pointer

d0-d7      -          FP/SIMD argument registers
d8-d15     -          Callee-saved FP registers
d16-d31    -          Caller-saved FP scratch registers
```

---

## 6. How LLVM Implements Calling Conventions

LLVM uses a three-layer system for calling conventions:

```
Layer 1: TableGen (.td file)
  Declarative rules: "if type is i32, assign to register W0"
  TableGen generates C++ code from these rules automatically.

Layer 2: Generated C++ (AArch64GenCallingConv.inc)
  Auto-generated from the .td file by TableGen.
  Contains the actual CC assignment functions.

Layer 3: Custom C++ (.cpp file)
  Hand-written logic for cases too complex for TableGen.
  For example: consecutive register blocks, SVE tuple passing.
```

### The CCState machine

LLVM uses a `CCState` object to track the state of argument assignment:

```
CCState tracks:
  - Which registers have been allocated
  - How much stack space has been used
  - The list of CCValAssign results (where each argument ended up)
```

### CCValAssign

Each argument gets a `CCValAssign` that records:

```
ValNo    = which argument number this is (0, 1, 2, ...)
ValVT    = the value type (i32, f64, v4i32, ...)
LocVT    = the location type (may differ after promotion/conversion)
LocInfo  = Full / ZExt / SExt / AExt / BCvt / Indirect / ...
Location = register (e.g. X0) or stack offset (e.g. SP+16)
```

---

## 7. File: AArch64CallingConvention.h — Entry Points

**File:** `AArch64CallingConvention.h`

This header declares the **entry point functions** for each calling convention.
These functions are generated by TableGen from the `.td` file, but their
signatures are declared here so other files can call them.

### Function signature pattern

Every CC function has the same signature:

```cpp
bool CC_<Name>(
    unsigned ValNo,           // which argument (0, 1, 2, ...)
    MVT ValVT,                // original value type
    MVT LocVT,                // location type (after promotion etc.)
    CCValAssign::LocInfo,     // how the value is stored (ZExt, SExt, Full...)
    ISD::ArgFlagsTy ArgFlags, // flags: byval, sret, inreg, nest, etc.
    Type *OrigTy,             // original LLVM IR type
    CCState &State            // current allocation state
);
// Returns false = argument was successfully assigned
// Returns true  = argument could not be assigned (caller must handle)
```

### All declared entry points

| Function | Purpose |
|----------|---------|
| `CC_AArch64_AAPCS` | Standard AArch64 calling convention (Linux/ELF) |
| `CC_AArch64_Arm64EC_VarArg` | Arm64EC variadic arguments (x64-compatible layout) |
| `CC_AArch64_Arm64EC_Thunk` | Arm64EC thunk (x64 CC mapped to AArch64 registers) |
| `CC_AArch64_Arm64EC_Thunk_Native` | Native side of Arm64EC thunks |
| `CC_AArch64_DarwinPCS_VarArg` | Darwin variadic arguments |
| `CC_AArch64_DarwinPCS` | Darwin (macOS/iOS) calling convention |
| `CC_AArch64_DarwinPCS_ILP32_VarArg` | Darwin ILP32 (arm64_32) variadic |
| `CC_AArch64_Win64PCS` | Windows AArch64 calling convention |
| `CC_AArch64_Win64_VarArg` | Windows variadic arguments |
| `CC_AArch64_Win64_CFGuard_Check` | Windows Control Flow Guard check |
| `CC_AArch64_Arm64EC_CFGuard_Check` | Arm64EC Control Flow Guard check |
| `CC_AArch64_GHC` | Glasgow Haskell Compiler calling convention |
| `CC_AArch64_Preserve_None` | Preserve-none (all registers volatile) |
| `RetCC_AArch64_AAPCS` | Return value convention for AAPCS |
| `RetCC_AArch64_Arm64EC_Thunk` | Return value convention for Arm64EC thunks |
| `RetCC_AArch64_Arm64EC_CFGuard_Check` | Return value for Arm64EC CFGuard |

### Naming convention

```
CC_*     = argument passing convention (incoming arguments)
RetCC_*  = return value convention (outgoing return values)
```

---

## 8. File: AArch64CallingConvention.td — The Rules

**File:** `AArch64CallingConvention.td`

This is the most important file. It contains the **declarative rules** that
describe every calling convention. TableGen reads this and generates C++ code.

### TableGen CC language basics

TableGen uses a small domain-specific language for calling conventions.
The key actions are:

| Action | Meaning |
|--------|---------|
| `CCAssignToReg<[R0, R1, ...]>` | Assign to the next available register from this list |
| `CCAssignToStack<size, align>` | Assign to a stack slot of given size and alignment |
| `CCIfType<[types], action>` | Apply action only if value type matches |
| `CCIf<"condition", action>` | Apply action only if C++ condition is true |
| `CCIfSRet<action>` | Apply if this is a struct-return pointer |
| `CCIfByVal<action>` | Apply if this is a byval argument |
| `CCIfInReg<action>` | Apply if the `inreg` attribute is set |
| `CCIfNest<action>` | Apply if this is a nest (trampoline) parameter |
| `CCIfSwiftSelf<action>` | Apply if this is a Swift self parameter |
| `CCIfSwiftError<action>` | Apply if this is a Swift error parameter |
| `CCIfSwiftAsync<action>` | Apply if this is a Swift async context |
| `CCPromoteToType<T>` | Promote small type to larger type before assigning |
| `CCBitConvertToType<T>` | Reinterpret bits as a different type |
| `CCPassByVal<size, align>` | Pass directly on stack (byval) |
| `CCPassIndirect<T>` | Pass a pointer to the value instead |
| `CCDelegateTo<CC>` | Fall through to another calling convention |
| `CCCustom<"fn">` | Call a custom C++ function for this argument |
| `CCAssignToRegWithShadow<regs, shadows>` | Assign to reg, mark shadow regs used |
| `CCIfConsecutiveRegs<action>` | Apply if argument is part of a consecutive block |
| `CCIfSplit<action>` | Apply if this is a split (e.g. i128 -> two i64s) |

---

### Block 1: Helper Classes (Lines 1–27)

```tablegen
class CCIfBigEndian<CCAction A>
class CCIfILP32<CCAction A>
class CCIfSubtarget<string F, CCAction A>
```

These are **conditional wrappers** that apply a CC action only when a
specific condition is true:

- `CCIfBigEndian` — only in big-endian mode
- `CCIfILP32` — only when pointers are 32-bit (arm64_32 / Darwin ILP32)
- `CCIfSubtarget` — only when the subtarget has a specific feature

---

### Block 2: AArch64_Common — Shared AAPCS Rules (Lines 29–107)

```tablegen
defvar AArch64_Common = [ ... ];
```

This is a **shared list of rules** used by both `CC_AArch64_AAPCS` and
`CC_AArch64_Win64PCS`. It defines the core AAPCS64 argument passing rules.

#### Rule-by-rule breakdown

**Nest parameter:**
```tablegen
CCIfNest<CCAssignToReg<[X15]>>
```
The "nest" parameter (used by function pointer trampolines) goes in `x15`.

**Pointer type conversion:**
```tablegen
CCIfType<[iPTR], CCBitConvertToType<i64>>
```
Pointer types are treated as 64-bit integers.

**Vector type conversions:**
```tablegen
CCIfType<[v2f32], CCBitConvertToType<v2i32>>
CCIfType<[v2f64, v4f32], CCBitConvertToType<v2i64>>
```
Some vector types are reinterpreted as integer vectors for passing.

**Big-endian vector handling:**
```tablegen
CCIfBigEndian<CCIfType<[v2i32, v2f32, ...], CCBitConvertToType<f64>>>
```
In big-endian mode, vectors are passed as single-element vectors to
preserve lane ordering.

**Struct return (sret) pointer:**
```tablegen
CCIfSRet<CCIfType<[i64], CCAssignToReg<[X8]>>>
```
When a function returns a large struct, the caller allocates memory and
passes a pointer to it. That pointer goes in `x8`.

On Windows, `inreg` sret uses `x0`/`x1` instead:
```tablegen
CCIfInReg<CCIfType<[i64], CCIfSRet<CCIfType<[i64], CCAssignToReg<[X0, X1]>>>>>
```

**ByVal arguments:**
```tablegen
CCIfByVal<CCPassByVal<8, 8>>
```
`byval` arguments are copied directly onto the stack with 8-byte size
and 8-byte alignment minimum.

**Swift special registers:**
```tablegen
CCIfSwiftSelf<CCIfType<[i64], CCAssignToReg<[X20]>>>
CCIfSwiftError<CCIfType<[i64], CCAssignToReg<[X21]>>>
CCIfSwiftAsync<CCIfType<[i64], CCAssignToReg<[X22]>>>
```
Swift uses dedicated registers for its runtime parameters:
- `x20` = Swift self (the object pointer)
- `x21` = Swift error (error return value)
- `x22` = Swift async context

**Consecutive register blocks:**
```tablegen
CCIfConsecutiveRegs<CCCustom<"CC_AArch64_Custom_Block">>
```
Arrays and structs that need consecutive registers are handled by
the custom C++ function `CC_AArch64_Custom_Block`.

**SVE vector arguments (z registers):**
```tablegen
CCIfType<[nxv16i8, nxv8i16, nxv4i32, nxv2i64, ...],
         CCAssignToReg<[Z0, Z1, Z2, Z3, Z4, Z5, Z6, Z7]>>
CCIfType<[nxv16i8, ...], CCPassIndirect<i64>>
```
SVE vectors go in `z0`–`z7`. If no registers are available, they are
passed indirectly (pointer in an integer register).

**SVE predicate arguments (p registers):**
```tablegen
CCIfType<[nxv1i1, nxv2i1, nxv4i1, nxv8i1, nxv16i1, aarch64svcount],
         CCAssignToReg<[P0, P1, P2, P3]>>
```
SVE predicates go in `p0`–`p3`.

**Small integer promotion:**
```tablegen
CCIfType<[i1, i8, i16], CCPromoteToType<i32>>
```
`i1`, `i8`, `i16` are promoted to `i32` before being placed in a register.
AArch64 does not have 8-bit or 16-bit argument registers.

**Integer arguments:**
```tablegen
CCIfType<[i32], CCAssignToReg<[W0, W1, W2, W3, W4, W5, W6, W7]>>
CCIfType<[i64], CCAssignToReg<[X0, X1, X2, X3, X4, X5, X6, X7]>>
```
Up to 8 integer arguments in `w0`–`w7` (32-bit) or `x0`–`x7` (64-bit).

**i128 split handling:**
```tablegen
CCIfType<[i64], CCIfSplit<CCAssignToRegWithShadow<[X0, X2, X4, X6],
                                                  [X0, X1, X3, X5]>>>
```
`i128` is split into two `i64` halves. The halves must go in even-numbered
register pairs: `(x0,x1)`, `(x2,x3)`, `(x4,x5)`, `(x6,x7)`.
The shadow registers ensure the paired register is also marked as used.

**Float arguments:**
```tablegen
CCIfType<[f16],  CCAssignToReg<[H0, H1, H2, H3, H4, H5, H6, H7]>>
CCIfType<[bf16], CCAssignToReg<[H0, H1, H2, H3, H4, H5, H6, H7]>>
CCIfType<[f32],  CCAssignToReg<[S0, S1, S2, S3, S4, S5, S6, S7]>>
CCIfType<[f64],  CCAssignToReg<[D0, D1, D2, D3, D4, D5, D6, D7]>>
CCIfType<[f128, v2i64, ...], CCAssignToReg<[Q0, Q1, Q2, Q3, Q4, Q5, Q6, Q7]>>
```
Float and vector arguments use the SIMD/FP register file, separate from
the integer registers. Up to 8 of each type.

**Stack overflow:**
```tablegen
CCIfType<[i1, i8, i16, f16, bf16], CCAssignToStack<8, 8>>
CCIfType<[i32, f32],               CCAssignToStack<8, 8>>
CCIfType<[i64, f64, ...],          CCAssignToStack<8, 8>>
CCIfType<[f128, v2i64, ...],       CCAssignToStack<16, 16>>
```
When all 8 registers of a type are used, remaining arguments go on the
stack. Note: even small types (i8, i16) use 8-byte stack slots in AAPCS.

---

### Block 3: CC_AArch64_AAPCS (Line ~109)

```tablegen
let Entry = 1 in
def CC_AArch64_AAPCS : CallingConv<AArch64_Common>;
```

The standard AArch64 calling convention used on Linux/ELF.
Simply uses the shared `AArch64_Common` rules.

---

### Block 4: RetCC_AArch64_AAPCS (Lines ~111–148)

```tablegen
let Entry = 1 in
def RetCC_AArch64_AAPCS : CallingConv<[...]>
```

The **return value** convention. Similar to argument passing but:
- No sret, byval, nest, or Swift special cases
- No stack overflow (return values must fit in registers)
- Supports returning multiple values (up to 8 integers, 8 floats)

Return value registers:

```
Integer:  x0, x1, x2, x3, x4, x5, x6, x7
Float:    h0-h7 / s0-s7 / d0-d7 / q0-q7
SVE:      z0-z7
Predicate: p0-p3
```

---

### Block 5: CC_AArch64_Win64PCS (Lines ~150–158)

```tablegen
def CC_AArch64_Win64PCS : CallingConv<!listconcat([
  CCIfCFGuardTarget<CCAssignToReg<[X9]>>
], AArch64_Common)>
```

Windows AArch64 calling convention. Mostly the same as AAPCS with one
addition: the `CFGuardTarget` parameter (used by Control Flow Guard) goes
in `x9`.

---

### Block 6: CC_AArch64_Win64_VarArg (Lines ~160–167)

```tablegen
def CC_AArch64_Win64_VarArg : CallingConv<[
  CCIfType<[f16, bf16], CCBitConvertToType<i16>>,
  CCIfType<[f32], CCBitConvertToType<i32>>,
  CCIfType<[f64], CCBitConvertToType<i64>>,
  CCDelegateTo<CC_AArch64_Win64PCS>
]>
```

Windows variadic functions pass **floats in integer registers**.
This matches the x64 Windows ABI behavior where `printf("%f", 1.0)`
passes the float as an integer bit pattern.

---

### Block 7: CC_AArch64_Arm64EC_VarArg (Lines ~169–215)

Arm64EC (ARM64 Emulation Compatible) is a Windows ABI that allows
AArch64 code to interoperate with x64 code. The variadic convention
uses an x64-compatible stack layout:

- Floats are converted to integers
- Large vectors are passed indirectly
- Only 4 argument registers (x0–x3), not 8
- Everything else goes on the stack

---

### Block 8: CC_AArch64_Arm64EC_Thunk (Lines ~217–280)

Arm64EC thunks are bridge functions between x64 and AArch64 code.
They use a calling convention that precisely mirrors the x64 calling
convention, but with AArch64 register names:

```
x64 register  ->  AArch64 register
RCX           ->  X0
RDX           ->  X1
R8            ->  X2
R9            ->  X3
XMM0          ->  Q0 (shadow of X0)
XMM1          ->  Q1 (shadow of X1)
XMM2          ->  Q2 (shadow of X2)
XMM3          ->  Q3 (shadow of X3)
```

The `CCAssignToRegWithShadow` action handles the x64 rule where integer
and float registers shadow each other (using one consumes the other).

---

### Block 9: CC_AArch64_DarwinPCS (Lines ~300–370)

Darwin (macOS/iOS) calling convention. Differs from AAPCS in two ways:

1. **i128 split**: halves do not need even-numbered registers
   ```tablegen
   CCIfType<[i64], CCIfSplit<CCAssignToReg<[X0, X1, X2, X3, X4, X5, X6]>>>
   ```
   Any two consecutive registers can hold an i128, not just even pairs.

2. **Stack slot sizing**: slots are sized as needed, not padded to 8 bytes
   ```tablegen
   CCIf<"ValVT == MVT::i1 || ValVT == MVT::i8", CCAssignToStack<1, 1>>
   CCIf<"ValVT == MVT::i16 ...", CCAssignToStack<2, 2>>
   CCIfType<[i32, f32], CCAssignToStack<4, 4>>
   ```
   An `i8` argument on the stack uses 1 byte, not 8 bytes.

---

### Block 10: CC_AArch64_DarwinPCS_VarArg (Lines ~372–395)

Darwin variadic convention. All anonymous arguments go on the stack.
Small types are promoted to `i64`/`f64` before stack placement.

```tablegen
CCIfType<[i8, i16, i32], CCPromoteToType<i64>>
CCIfType<[f16, bf16, f32], CCPromoteToType<f64>>
```

---

### Block 11: CC_AArch64_GHC (Lines ~430–460)

The **Glasgow Haskell Compiler** calling convention. GHC implements
its own runtime (the Spineless Tagless G-Machine) and uses a completely
different register assignment:

```
STG register  ->  AArch64 register
Base          ->  X19
Sp            ->  X20
Hp            ->  X21
R1            ->  X22
R2            ->  X23
R3            ->  X24
R4            ->  X25
R5            ->  X26
R6            ->  X27
SpLim         ->  X28
```

These are all callee-saved registers in AAPCS, which means GHC functions
can call C functions without saving/restoring their STG registers.

Float registers used:
```
f32:  S8, S9, S10, S11
f64:  D12, D13, D14, D15
v2f64: Q4, Q5
```

---

### Block 12: CC_AArch64_Preserve_None (Lines ~462–500)

A special calling convention where **no registers are preserved** by the
callee (except `fp`, `lr`, and `x18`). This allows the compiler to pass
arguments in any available register, including callee-saved ones.

The register order puts callee-saved registers first so that normal
function calls do not need to save/reload arguments:

```tablegen
CCIfType<[i64], CCAssignToReg<[X20, X21, X22, X23,
                               X24, X25, X26, X27, X28,
                               X0, X1, X2, X3, X4, X5,
                               X6, X7, X10, X11,
                               X12, X13, X14, X9]>>
```

Excluded registers:
- `x8` — used for sret
- `x15` — used for stack allocation on Windows
- `x16`/`x17` — linker scratch (IP0/IP1)
- `x18` — platform register
- `x19` — base pointer
- `x29` — frame pointer
- `x30` — link register

---

## 9. File: AArch64CallingConvention.cpp — Custom Logic

**File:** `AArch64CallingConvention.cpp`

This file contains **hand-written C++ functions** for argument assignment
cases that are too complex to express in TableGen.

### Register lists

```cpp
static const MCPhysReg XRegList[] = {X0, X1, X2, X3, X4, X5, X6, X7};
static const MCPhysReg HRegList[] = {H0, H1, H2, H3, H4, H5, H6, H7};
static const MCPhysReg SRegList[] = {S0, S1, S2, S3, S4, S5, S6, S7};
static const MCPhysReg DRegList[] = {D0, D1, D2, D3, D4, D5, D6, D7};
static const MCPhysReg QRegList[] = {Q0, Q1, Q2, Q3, Q4, Q5, Q6, Q7};
static const MCPhysReg ZRegList[] = {Z0, Z1, Z2, Z3, Z4, Z5, Z6, Z7};
static const MCPhysReg PRegList[] = {P0, P1, P2, P3};
```

These are the ordered lists of registers for each type class.
The CC assignment functions try registers in this order.

---

### `finishStackBlock` (helper)

```cpp
static bool finishStackBlock(
    SmallVectorImpl<CCValAssign> &PendingMembers,
    MVT LocVT, ISD::ArgFlagsTy &ArgFlags,
    CCState &State, Align SlotAlign)
```

**What it does:**

Finalizes the stack placement of a "pending block" — a group of arguments
that were waiting to be assigned together (e.g. a struct passed as
consecutive registers that did not fit).

**Two cases:**

1. **Scalable vector (SVE tuple):**
   - Temporarily marks all Z and P registers as allocated
   - Re-invokes the CC assignment function to force indirect passing
   - Restores the register state afterward
   - This implements the SVE PCS rule: if a tuple cannot fit in registers,
     pass it indirectly AND leave remaining registers unallocated for
     other arguments

2. **Fixed-size block:**
   - Assigns each pending member to a stack slot
   - Uses `SlotAlign` for the first slot, then 1-byte alignment for the rest

---

### `CC_AArch64_Custom_Stack_Block`

```cpp
static bool CC_AArch64_Custom_Stack_Block(
    unsigned &ValNo, MVT &ValVT, MVT &LocVT,
    CCValAssign::LocInfo &LocInfo,
    ISD::ArgFlagsTy &ArgFlags, CCState &State)
```

**What it does:**

Handles the Darwin variadic PCS rule for anonymous arguments.
Anonymous arguments in Darwin variadic functions are placed in
**8-byte stack slots**, even if the type is smaller.

This is called when `CCIfConsecutiveRegs` triggers for a Darwin
variadic function.

**How it works:**

1. Adds the current argument to a `PendingMembers` list
2. If this is not the last member of the block, returns `true` (keep collecting)
3. When the last member arrives, calls `finishStackBlock` with 8-byte alignment

---

### `CC_AArch64_Custom_Block`

```cpp
static bool CC_AArch64_Custom_Block(
    unsigned &ValNo, MVT &ValVT, MVT &LocVT,
    CCValAssign::LocInfo &LocInfo,
    ISD::ArgFlagsTy &ArgFlags, CCState &State)
```

**What it does:**

Handles passing of **consecutive register blocks** — arrays and structs
that must be passed in a contiguous sequence of registers.

For example, a `[4 x i64]` array should go in `{x0, x1, x2, x3}` as a
block, not scattered across available registers.

**How it works:**

1. Determines which register list to use based on the element type:
   ```
   i64       -> XRegList (x0-x7)
   f16/bf16  -> HRegList (h0-h7)
   f32       -> SRegList (s0-s7)
   f64       -> DRegList (d0-d7)
   f128      -> QRegList (q0-q7)
   SVE mask  -> PRegList (p0-p3)
   SVE vec   -> ZRegList (z0-z7)
   ```

2. Collects all members of the block into `PendingMembers`

3. When the last member arrives, tries to allocate a **contiguous block**
   of registers using `State.AllocateRegBlock`

4. If a contiguous block is available:
   - Assigns each member to its register
   - Clears `PendingMembers`

5. If no contiguous block is available:
   - Marks all registers of that type as used (so no partial allocation)
   - Falls through to stack placement via `finishStackBlock`

**Darwin ILP32 special case:**

On Darwin arm64_32, `i32` values are packed two-per-register into `x`
registers (since pointers are 32-bit but registers are 64-bit):

```
[i32, i32] -> X0 (low 32 bits = first i32, high 32 bits = second i32)
```

This uses `CCValAssign::ZExt` and `CCValAssign::AExtUpper` to describe
the packing.

---

### `#include "AArch64GenCallingConv.inc"`

The last line of the `.cpp` file includes the **TableGen-generated code**.
This is where all the `CC_AArch64_AAPCS`, `RetCC_AArch64_AAPCS`, etc.
functions are actually defined (generated from the `.td` file).

---

## 10. All Calling Conventions Defined in This Backend

### Standard conventions

| Convention | Platform | Key characteristics |
|------------|----------|---------------------|
| `CC_AArch64_AAPCS` | Linux/ELF | Standard AArch64 PCS. 8 int regs, 8 FP regs, 8-byte stack slots |
| `CC_AArch64_DarwinPCS` | macOS/iOS | Like AAPCS but smaller stack slots, relaxed i128 alignment |
| `CC_AArch64_Win64PCS` | Windows | Like AAPCS plus CFGuard target in x9 |

### Variadic conventions

| Convention | Platform | Key characteristics |
|------------|----------|---------------------|
| `CC_AArch64_DarwinPCS_VarArg` | macOS/iOS | All anonymous args on stack, promoted to i64/f64 |
| `CC_AArch64_DarwinPCS_ILP32_VarArg` | macOS arm64_32 | Like Darwin VarArg but 4-byte slots |
| `CC_AArch64_Win64_VarArg` | Windows | Floats passed as integers (x64 compatibility) |
| `CC_AArch64_Arm64EC_VarArg` | Windows Arm64EC | x64-compatible layout, only 4 arg regs |

### Arm64EC conventions

| Convention | Purpose |
|------------|---------|
| `CC_AArch64_Arm64EC_Thunk` | x64 CC mapped to AArch64 registers |
| `CC_AArch64_Arm64EC_Thunk_Native` | Native AArch64 side of EC thunks |
| `CC_AArch64_Arm64EC_CFGuard_Check` | Arm64EC Control Flow Guard |

### Special-purpose conventions

| Convention | Purpose |
|------------|---------|
| `CC_AArch64_GHC` | Glasgow Haskell Compiler STG machine |
| `CC_AArch64_Preserve_None` | All registers volatile, max argument registers |
| `CC_AArch64_Win64_CFGuard_Check` | Windows CFGuard: single arg in x15 |

### Return value conventions

| Convention | Purpose |
|------------|---------|
| `RetCC_AArch64_AAPCS` | Standard return values |
| `RetCC_AArch64_Arm64EC_Thunk` | Arm64EC thunk return values (x64 layout) |
| `RetCC_AArch64_Arm64EC_CFGuard_Check` | Arm64EC CFGuard return |

---

## 11. Callee-Saved Register Sets (CSR)

The `.td` file also defines **callee-saved register sets** using
`CalleeSavedRegs`. These tell the register allocator which registers
must be preserved across function calls.

### Standard sets

| CSR name | Registers saved |
|----------|----------------|
| `CSR_AArch64_AAPCS` | x19-x28, lr, fp, d8-d15 |
| `CSR_Darwin_AArch64_AAPCS` | lr, fp, x19-x28, d8-d15 (fp/lr first for Darwin) |
| `CSR_Win_AArch64_AAPCS` | x19-x28, fp, lr, d8-d15 (fp before lr for Win unwind) |

### Why Darwin puts LR/FP first

Darwin's compact unwind format requires the frame record `(fp, lr)` to be
at the **top** of the callee-save area. Other platforms put it at the bottom
to allow SVE objects to be accessed directly from the frame pointer.

### Extended sets

| CSR name | Extra registers | Purpose |
|----------|----------------|---------|
| `CSR_AArch64_AAPCS_X18` | + x18 | When x18 must be preserved |
| `CSR_AArch64_AAVPCS` | + q8-q23 | Vector PCS (full SIMD preservation) |
| `CSR_AArch64_SVE_AAPCS` | + z8-z23, p4-p15 | SVE calling convention |
| `CSR_AArch64_AAPCS_SwiftError` | - x21 | Swift error register is volatile |
| `CSR_AArch64_AAPCS_SwiftTail` | - x20, x22 | Swift tail call variant |
| `CSR_AArch64_AAPCS_ThisReturn` | + x0 | iOS C++ constructors return `this` |

### Shadow call stack variants

Every major CSR has an `_SCS` variant that additionally preserves `x18`
(used as the shadow call stack pointer):

```
CSR_AArch64_AAPCS_SCS
CSR_AArch64_AAVPCS_SCS
CSR_AArch64_SVE_AAPCS_SCS
... etc.
```

### Special-purpose sets

| CSR name | Purpose |
|----------|---------|
| `CSR_AArch64_NoRegs` | No registers saved (e.g. naked functions) |
| `CSR_AArch64_AllRegs` | All registers saved (e.g. signal handlers) |
| `CSR_AArch64_TLS_ELF` | TLS descriptor stub (saves almost everything) |
| `CSR_AArch64_SME_ABI_Support_Routines_*` | SME runtime routines (preserve most regs) |
| `CSR_AArch64_StackProbe_Windows` | Windows stack probe helper |
| `CSR_Win_AArch64_CFGuard_Check` | CFGuard check (also preserves x0-x8, q0-q7) |
| `CSR_Win_AArch64_Arm64EC_Thunk` | Arm64EC thunk (preserves q6-q15 for x64 compat) |

---

## 12. TableGen CC Actions Reference

Complete reference of all TableGen calling convention actions used in this file:

### Assignment actions

| Action | Syntax | Effect |
|--------|--------|--------|
| `CCAssignToReg` | `CCAssignToReg<[R0, R1, ...]>` | Assign to next free register from list |
| `CCAssignToStack` | `CCAssignToStack<size, align>` | Assign to stack slot |
| `CCAssignToRegWithShadow` | `CCAssignToRegWithShadow<regs, shadows>` | Assign to reg, mark shadows used |
| `CCAssignToStackWithShadow` | `CCAssignToStackWithShadow<size, align, shadows>` | Stack + mark shadow regs used |
| `CCPassByVal` | `CCPassByVal<size, align>` | Copy directly to stack (byval) |
| `CCPassIndirect` | `CCPassIndirect<T>` | Pass pointer to value |
| `CCCustom` | `CCCustom<"fn_name">` | Call custom C++ function |
| `CCDelegateTo` | `CCDelegateTo<CC>` | Fall through to another CC |

### Type conversion actions

| Action | Syntax | Effect |
|--------|--------|--------|
| `CCPromoteToType` | `CCPromoteToType<T>` | Promote to larger type |
| `CCBitConvertToType` | `CCBitConvertToType<T>` | Reinterpret bits as different type |
| `CCTruncToType` | `CCTruncToType<T>` | Truncate to smaller type |

### Conditional wrappers

| Condition | Syntax | Triggers when |
|-----------|--------|---------------|
| `CCIfType` | `CCIfType<[types], action>` | Value type matches |
| `CCIf` | `CCIf<"expr", action>` | C++ expression is true |
| `CCIfSRet` | `CCIfSRet<action>` | Argument has sret attribute |
| `CCIfByVal` | `CCIfByVal<action>` | Argument has byval attribute |
| `CCIfInReg` | `CCIfInReg<action>` | Argument has inreg attribute |
| `CCIfNest` | `CCIfNest<action>` | Argument has nest attribute |
| `CCIfSwiftSelf` | `CCIfSwiftSelf<action>` | Argument has swiftself attribute |
| `CCIfSwiftError` | `CCIfSwiftError<action>` | Argument has swifterror attribute |
| `CCIfSwiftAsync` | `CCIfSwiftAsync<action>` | Argument has swiftasync attribute |
| `CCIfCFGuardTarget` | `CCIfCFGuardTarget<action>` | Argument has cfguardtarget attribute |
| `CCIfConsecutiveRegs` | `CCIfConsecutiveRegs<action>` | Part of consecutive register block |
| `CCIfSplit` | `CCIfSplit<action>` | This is a split value (e.g. i128 half) |
| `CCIfVarArg` | `CCIfVarArg<action>` | Function is variadic |
| `CCIfBigEndian` | `CCIfBigEndian<action>` | Target is big-endian |
| `CCIfILP32` | `CCIfILP32<action>` | Pointer size is 32 bits |
| `CCIfSubtarget` | `CCIfSubtarget<"feat", action>` | Subtarget has feature |
| `CCIfPtr` | `CCIfPtr<action>` | Value is a pointer type |

---

## 13. Full Conceptual Map

```
Calling Convention System
│
├── Why it exists
│     Functions compiled separately must agree on how to communicate.
│     Without rules: arguments in wrong registers, crashes, wrong results.
│
├── What it defines
│     - Which registers hold arguments (x0-x7, d0-d7, z0-z7, p0-p3)
│     - What happens when registers run out (stack overflow)
│     - Which register holds the return value (x0, d0)
│     - Which registers the callee must preserve (x19-x28, d8-d15)
│     - Stack alignment (16 bytes at call boundaries)
│     - Special registers (x8=sret, x15=nest, x18=platform)
│
├── AArch64CallingConvention.td
│     Declarative rules in TableGen language.
│     TableGen generates C++ from these rules.
│
│     ├── AArch64_Common (shared AAPCS rules)
│     │     Nest, sret, byval, Swift, SVE, integer, float, stack overflow
│     │
│     ├── CC_AArch64_AAPCS (Linux/ELF standard)
│     ├── RetCC_AArch64_AAPCS (return values)
│     ├── CC_AArch64_DarwinPCS (macOS/iOS)
│     ├── CC_AArch64_Win64PCS (Windows)
│     ├── CC_AArch64_Win64_VarArg (Windows variadic)
│     ├── CC_AArch64_DarwinPCS_VarArg (Darwin variadic)
│     ├── CC_AArch64_Arm64EC_* (x64 interop on Windows)
│     ├── CC_AArch64_GHC (Haskell runtime)
│     ├── CC_AArch64_Preserve_None (all regs volatile)
│     │
│     └── CalleeSavedRegs definitions
│           CSR_AArch64_AAPCS, CSR_Darwin_*, CSR_Win_*,
│           SVE variants, Swift variants, SCS variants, etc.
│
├── AArch64CallingConvention.h
│     Declares all CC entry point function signatures.
│     Used by AArch64ISelLowering.cpp to call the right CC.
│
├── AArch64CallingConvention.cpp
│     Custom C++ for complex cases TableGen cannot express.
│
│     ├── Register lists (XRegList, DRegList, ZRegList, etc.)
│     │
│     ├── finishStackBlock
│     │     Finalizes pending block on stack.
│     │     SVE tuples: force indirect if no register block available.
│     │     Fixed types: assign each member to a stack slot.
│     │
│     ├── CC_AArch64_Custom_Stack_Block
│     │     Darwin variadic: anonymous args in 8-byte stack slots.
│     │
│     ├── CC_AArch64_Custom_Block
│     │     Consecutive register blocks (arrays, structs).
│     │     Try to allocate contiguous register sequence.
│     │     If not possible, mark all regs used and go to stack.
│     │
│     └── #include "AArch64GenCallingConv.inc"
│           TableGen-generated CC functions included here.
│
└── How it connects to the rest of the backend
      AArch64ISelLowering.cpp calls CC functions during:
        - LowerFormalArguments (reading incoming args)
        - LowerCall (setting up outgoing args)
        - LowerReturn (setting up return values)
      AArch64FrameLowering.cpp uses CSR lists to decide which
        registers to save/restore in prologue/epilogue.
```

---

## 14. Key Concepts Quick Reference

### Argument passing summary (AAPCS64)

```
Type          Registers used          Stack slot size
-----------   ---------------------   ---------------
i1, i8, i16   promoted to i32 -> w0-w7    8 bytes
i32           w0-w7                        8 bytes
i64           x0-x7                        8 bytes
i128          x0+x1, x2+x3, x4+x5, x6+x7  16 bytes (aligned)
f16, bf16     h0-h7                        8 bytes
f32           s0-s7                        8 bytes
f64           d0-d7                        8 bytes
f128          q0-q7                        16 bytes
v2i32, v4i16  d0-d7                        8 bytes
v4i32, v2i64  q0-q7                        16 bytes
SVE vector    z0-z7 (or indirect)          indirect via pointer
SVE predicate p0-p3 (or indirect)          indirect via pointer
```

### Special argument registers

```
x8   = sret pointer (indirect return value location)
x15  = nest parameter (trampoline function pointer)
x20  = Swift self
x21  = Swift error
x22  = Swift async context
```

### Callee-saved registers (must be preserved)

```
GPR:  x19, x20, x21, x22, x23, x24, x25, x26, x27, x28, fp(x29), lr(x30)
FPR:  d8, d9, d10, d11, d12, d13, d14, d15
SVE:  z8-z23, p4-p15  (only in SVE calling convention)
```

### Caller-saved registers (volatile, may be destroyed)

```
GPR:  x0-x17 (x18 is platform-reserved on some OSes)
FPR:  d0-d7, d16-d31
```

### Platform differences

| Feature | Linux/ELF | macOS/iOS | Windows |
|---------|-----------|-----------|---------|
| i128 alignment | even reg pairs | any two regs | even reg pairs |
| Stack slot min | 8 bytes | actual size | 8 bytes |
| Frame record order | LR, FP (bottom of CSR) | FP, LR (top of CSR) | FP, LR (top of CSR) |
| x18 | available | reserved | reserved |
| Variadic floats | in FP regs | on stack | as integers in int regs |
| sret register | x8 | x8 | x8 (or x0/x1 with inreg) |
