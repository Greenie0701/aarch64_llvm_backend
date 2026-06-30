# AArch64 Registers in LLVM - Comprehensive Guide

## Table of Contents
1. [Introduction to Registers](#introduction-to-registers)
2. [What are Registers?](#what-are-registers)
3. [AArch64 Register Types](#aarch64-register-types)
4. [Register Classes](#register-classes)
5. [Register Banks](#register-banks)
6. [Register Definition Files](#register-definition-files)
7. [Register Usage in LLVM](#register-usage-in-llvm)
8. [Advanced Topics](#advanced-topics)

---

## Introduction to Registers

This document provides a comprehensive overview of how registers are defined, organized, and used in the LLVM AArch64 backend. It covers the register architecture, TableGen definitions, register allocation, and the role of registers in code generation.

---

## What are Registers?

**Registers** are small, fast storage locations directly accessible by the CPU. They are the fastest form of memory in the computer hierarchy and are used to:

- Store operands for arithmetic and logical operations
- Hold intermediate computation results
- Store memory addresses for load/store operations
- Maintain program state (stack pointer, frame pointer, etc.)
- Hold function arguments and return values

### Key Characteristics:
- **Speed**: Fastest memory access (typically 1 CPU cycle)
- **Size**: Limited number available (32-64 registers in AArch64)
- **Types**: Different registers for different purposes (integer, floating-point, vector, etc.)
- **Width**: Varying bit widths (8, 16, 32, 64, 128, or scalable bits)

---

## AArch64 Register Types

AArch64 (ARM 64-bit architecture) provides several categories of registers:

### 1. General Purpose Registers (GPRs)

#### 32-bit Registers (W0-W30, WSP, WZR)
- **W0-W30**: 31 general-purpose 32-bit registers
- **WSP**: 32-bit Stack Pointer
- **WZR**: 32-bit Zero Register (always reads as 0, writes are discarded)

```tablegen
def W0    : AArch64Reg<0,   "w0" >, DwarfRegNum<[0]>;
def W1    : AArch64Reg<1,   "w1" >, DwarfRegNum<[1]>;
// ... W2-W30
def WSP   : AArch64Reg<31, "wsp">, DwarfRegNum<[31]>;
def WZR   : AArch64Reg<31, "wzr">, DwarfRegAlias<WSP>;
```

#### 64-bit Registers (X0-X30, SP, XZR, FP, LR)
- **X0-X28**: 29 general-purpose 64-bit registers
- **X29 (FP)**: Frame Pointer
- **X30 (LR)**: Link Register (return address)
- **SP**: 64-bit Stack Pointer
- **XZR**: 64-bit Zero Register

```tablegen
def X0    : AArch64Reg<0,   "x0",  [W0,  W0_HI]>, DwarfRegAlias<W0>;
def FP    : AArch64Reg<29, "x29", [W29, W29_HI]>, DwarfRegAlias<W29>;
def LR    : AArch64Reg<30, "x30", [W30, W30_HI]>, DwarfRegAlias<W30>;
def SP    : AArch64Reg<31, "sp",  [WSP, WSP_HI]>, DwarfRegAlias<WSP>;
```

**Usage**:
- Function arguments: X0-X7
- Return values: X0-X1
- Callee-saved: X19-X28
- Special: X29 (FP), X30 (LR), X31 (SP/XZR)

### 2. Floating-Point and Vector Registers

AArch64 provides a unified register file for floating-point and SIMD operations:

#### Scalar Floating-Point Registers
- **B0-B31**: 8-bit (byte)
- **H0-H31**: 16-bit (half-precision float)
- **S0-S31**: 32-bit (single-precision float)
- **D0-D31**: 64-bit (double-precision float)
- **Q0-Q31**: 128-bit (quad-word, SIMD vectors)

```tablegen
def B0    : AArch64Reg<0,   "b0">, DwarfRegNum<[64]>;
def H0    : AArch64Reg<0,   "h0", [B0, B0_HI]>, DwarfRegAlias<B0>;
def S0    : AArch64Reg<0,   "s0", [H0, H0_HI]>, DwarfRegAlias<B0>;
def D0    : AArch64Reg<0,   "d0", [S0, S0_HI], ["v0", ""]>, DwarfRegAlias<B0>;
def Q0    : AArch64Reg<0,   "q0", [D0, D0_HI], ["v0", ""]>, DwarfRegAlias<B0>;
```

**Hierarchy**: Each register is a view of the same physical register:
```
Q0 (128-bit): [127:0]
  ├─ D0 (64-bit): [63:0]
  │   ├─ S0 (32-bit): [31:0]
  │   │   ├─ H0 (16-bit): [15:0]
  │   │   │   └─ B0 (8-bit): [7:0]
```

### 3. SVE (Scalable Vector Extension) Registers

SVE introduces scalable vector registers whose size is determined at runtime:

#### SVE Vector Registers (Z0-Z31)
- **Z0-Z31**: Scalable vector registers (128-2048 bits, in 128-bit increments)
- Each Z register contains a corresponding Q register as its lower 128 bits

```tablegen
def Z0    : AArch64Reg<0,   "z0",  [Q0, Q0_HI]>, DwarfRegNum<[96]>;
def Z1    : AArch64Reg<1,   "z1",  [Q1, Q1_HI]>, DwarfRegNum<[97]>;
// ... Z2-Z31
```

#### SVE Predicate Registers (P0-P15)
- **P0-P15**: Predicate registers for masked operations
- Each bit controls whether an element participates in an operation

```tablegen
def P0    : AArch64Reg<0,   "p0", [PN0]>, DwarfRegAlias<PN0>;
def P1    : AArch64Reg<1,   "p1", [PN1]>, DwarfRegAlias<PN1>;
// ... P2-P15
```

#### SVE Predicate-as-Counter Registers (PN0-PN15)
- **PN0-PN15**: Predicate-as-counter registers (SVE2.1 feature)

```tablegen
def PN0    : AArch64Reg<0,   "pn0">, DwarfRegNum<[48]>;
def PN1    : AArch64Reg<1,   "pn1">, DwarfRegNum<[49]>;
// ... PN2-PN15
```

### 4. SME (Scalable Matrix Extension) Registers

SME introduces matrix registers for matrix operations:

#### Matrix Accumulator (ZA)
- **ZA**: The entire matrix accumulator array
- Can be accessed as tiles: ZAB0, ZAH0-ZAH1, ZAS0-ZAS3, ZAD0-ZAD7, ZAQ0-ZAQ15

```tablegen
def ZAB0 : AArch64Reg<0, "za0.b", [ZAH0, ZAH1]>;
def ZAH0 : AArch64Reg<0, "za0.h", [ZAS0, ZAS2]>;
def ZAS0 : AArch64Reg<0, "za0.s", [ZAD0, ZAD4]>;
def ZAD0 : AArch64Reg<0, "za0.d", [ZAQ0, ZAQ8]>;
def ZAQ0  : AArch64Reg<0,  "za0.q">;
```

#### Lookup Table Register (ZT0)
- **ZT0**: 512-bit lookup table register (SME2)

```tablegen
def ZT0 : AArch64Reg<0, "zt0">;
```

### 5. System and Control Registers

#### Condition Flags (NZCV)
- **NZCV**: Negative, Zero, Carry, oVerflow flags
- Updated by comparison and arithmetic operations

```tablegen
def NZCV  : AArch64Reg<0, "nzcv">;
```

#### Floating-Point Control Registers
- **FPCR**: Floating-Point Control Register
- **FPSR**: Floating-Point Status Register
- **FPMR**: Floating-Point Mode Register

```tablegen
def FPCR : AArch64Reg<0, "fpcr">;
def FPSR : AArch64Reg<0, "fpsr">;
def FPMR : AArch64Reg<0, "fpmr">;
```

#### SVE Control Registers
- **FFR**: First Fault Register (SVE)
- **VG**: Vector Granule (virtual register for DWARF)

```tablegen
def FFR : AArch64Reg<0, "ffr">, DwarfRegNum<[47]>;
def VG : AArch64Reg<0, "vg">, DwarfRegNum<[46]>;
```

---

## Register Classes

Register classes group registers with similar properties for register allocation:

### GPR Register Classes

```tablegen
// GPR32: 32-bit general purpose registers (W0-W30, WZR)
def GPR32 : RegisterClass<"AArch64", [i32], 32, (add GPR32common, WZR)>;

// GPR64: 64-bit general purpose registers (X0-X28, FP, LR, XZR)
def GPR64 : RegisterClass<"AArch64", [i64], 64, (add GPR64common, XZR)>;

// GPR32sp: 32-bit registers including stack pointer
def GPR32sp : RegisterClass<"AArch64", [i32], 32, (add GPR32common, WSP)>;

// GPR64sp: 64-bit registers including stack pointer
def GPR64sp : RegisterClass<"AArch64", [i64], 64, (add GPR64common, SP)>;
```

**Key Distinctions**:
- **GPR32/GPR64**: Include zero register (WZR/XZR), exclude stack pointer
- **GPR32sp/GPR64sp**: Include stack pointer (WSP/SP), exclude zero register
- **GPR32common/GPR64common**: Exclude both zero register and stack pointer

### FPR Register Classes

```tablegen
// FPR8: 8-bit floating-point registers
def FPR8  : RegisterClass<"AArch64", [i8, aarch64mfp8], 8, (sequence "B%u", 0, 31)>;

// FPR16: 16-bit floating-point registers
def FPR16 : RegisterClass<"AArch64", [f16, bf16, i16], 16, (sequence "H%u", 0, 31)>;

// FPR32: 32-bit floating-point registers
def FPR32 : RegisterClass<"AArch64", [f32, i32], 32, (sequence "S%u", 0, 31)>;

// FPR64: 64-bit floating-point/vector registers
def FPR64 : RegisterClass<"AArch64", [f64, i64, v2f32, v1f64, v8i8, v4i16, v2i32,
                                      v1i64, v4f16, v4bf16], 64, (sequence "D%u", 0, 31)>;

// FPR128: 128-bit vector registers
def FPR128 : RegisterClass<"AArch64", [v16i8, v8i16, v4i32, v2i64, v4f32, v2f64, 
                                       f128, v8f16, v8bf16], 128, (sequence "Q%u", 0, 31)>;
```

### SVE Register Classes

```tablegen
// ZPR: SVE vector registers (scalable)
def ZPR : RegisterClass<"AArch64", [nxv16i8, nxv8i16, nxv4i32, nxv2i64,
                                    nxv2f16, nxv4f16, nxv8f16,
                                    nxv2bf16, nxv4bf16, nxv8bf16,
                                    nxv2f32, nxv4f32, nxv2f64],
                        128, (sequence "Z%u", 0, 31)>;

// PPR: SVE predicate registers
def PPR : RegisterClass<"AArch64", [nxv16i1, nxv8i1, nxv4i1, nxv2i1, nxv1i1], 
                        16, (sequence "P%u", 0, 15)>;

// PNR: SVE predicate-as-counter registers
def PNR : RegisterClass<"AArch64", [aarch64svcount], 16, (sequence "PN%u", 0, 15)>;
```

### Special Purpose Register Classes

```tablegen
// CCR: Condition code register
def CCR : RegisterClass<"AArch64", [FlagsVT], 32, (add NZCV)> {
  let CopyCost = -1;  // Don't allow copying of status registers
  let isAllocatable = 0;
}

// tcGPR64: Tail call registers (exclude callee-saved)
def tcGPR64 : RegisterClass<"AArch64", [i64], 64, 
                           (sub GPR64common, X19, X20, X21, X22, X23, X24, 
                                X25, X26, X27, X28, FP, LR)>;
```

---

## Register Banks

Register banks are a higher-level abstraction used in the GlobalISel instruction selection framework. They group register classes by functional unit:

### Definition (AArch64RegisterBanks.td)

```tablegen
/// General Purpose Registers: W, X
def GPRRegBank : RegisterBank<"GPR", [XSeqPairsClass]>;

/// Floating Point, Vector, Scalable Vector Registers: B, H, S, D, Q, Z
def FPRRegBank : RegisterBank<"FPR", [QQQQ, ZPR]>;

/// Conditional register: NZCV
def CCRegBank : RegisterBank<"CC", [CCR]>;
```

### Register Bank Characteristics

#### GPR Register Bank
- **Purpose**: Integer arithmetic, logical operations, address calculations
- **Registers**: W0-W30, X0-X30, SP, ZR
- **Maximum Size**: 128 bits (for paired registers)
- **Operations**: ADD, SUB, MUL, DIV, AND, OR, XOR, shifts, loads, stores

#### FPR Register Bank
- **Purpose**: Floating-point arithmetic, SIMD operations, vector operations
- **Registers**: B0-B31, H0-H31, S0-S31, D0-D31, Q0-Q31, Z0-Z31
- **Maximum Size**: 512 bits (QQQQ sequence) or scalable (SVE)
- **Operations**: FADD, FMUL, vector operations, SIMD instructions

#### CC Register Bank
- **Purpose**: Condition flags for conditional execution
- **Registers**: NZCV
- **Maximum Size**: 32 bits
- **Operations**: Comparisons, conditional branches

### Register Bank Selection

The `AArch64RegisterBankInfo` class determines which register bank to use:

```cpp
const RegisterBank &
AArch64RegisterBankInfo::getRegBankFromRegClass(const TargetRegisterClass &RC,
                                                LLT Ty) const {
  // Logic to determine register bank based on:
  // - Register class
  // - Type (vector, scalar, floating-point)
  // - Instruction constraints
}
```

**Selection Criteria**:
1. **Type-based**: Vectors and floats → FPR, Integers → GPR
2. **Instruction-based**: Floating-point ops → FPR, Integer ops → GPR
3. **Cost-based**: Minimize cross-bank copies (expensive)

---

## Register Definition Files

### 1. AArch64RegisterInfo.td

**Purpose**: TableGen file defining all registers, register classes, and their properties.

**Key Sections**:

#### Register Definitions
```tablegen
class AArch64Reg<bits<16> enc, string n, list<Register> subregs = [],
               list<string> altNames = []>
        : Register<n, altNames> {
  let HWEncoding = enc;
  let Namespace = "AArch64";
  let SubRegs = subregs;
}
```

#### SubRegister Indices
```tablegen
// For GPRs
def sub_32   : SubRegIndex<32>;      // W register in X register
def sub_32_hi : SubRegIndex<32, 32>; // Upper 32 bits

// For FPRs
def bsub    : SubRegIndex<8, 0>;     // B in H/S/D/Q
def hsub    : SubRegIndex<16, 0>;    // H in S/D/Q
def ssub    : SubRegIndex<32, 0>;    // S in D/Q
def dsub    : SubRegIndex<64, 0>;    // D in Q
def zsub    : SubRegIndex<128, 0>;   // Q in Z
```

#### Register Tuples
```tablegen
// Pairs of consecutive registers
def DSeqPairs : RegisterTuples<[dsub0, dsub1], 
                               [(rotl FPR64, 0), (rotl FPR64, 1)]>;

// Quads of consecutive registers
def QSeqQuads : RegisterTuples<[qsub0, qsub1, qsub2, qsub3],
                               [(rotl FPR128, 0), (rotl FPR128, 1),
                                (rotl FPR128, 2), (rotl FPR128, 3)]>;
```

### 2. AArch64RegisterInfo.h

**Purpose**: C++ header declaring the `AArch64RegisterInfo` class.

**Key Methods**:

```cpp
class AArch64RegisterInfo final : public AArch64GenRegisterInfo {
public:
  // Get callee-saved registers for a function
  const MCPhysReg *getCalleeSavedRegs(const MachineFunction *MF) const override;
  
  // Get call-preserved register mask
  const uint32_t *getCallPreservedMask(const MachineFunction &MF,
                                       CallingConv::ID) const override;
  
  // Get reserved registers (not available for allocation)
  BitVector getReservedRegs(const MachineFunction &MF) const override;
  
  // Get frame register (FP or SP)
  Register getFrameRegister(const MachineFunction &MF) const override;
  
  // Eliminate frame index references
  bool eliminateFrameIndex(MachineBasicBlock::iterator II, int SPAdj,
                           unsigned FIOperandNum,
                           RegScavenger *RS = nullptr) const override;
};
```

### 3. AArch64RegisterInfo.cpp

**Purpose**: Implementation of register information queries and transformations.

**Key Functionality**:

#### Reserved Registers
```cpp
BitVector AArch64RegisterInfo::getReservedRegs(const MachineFunction &MF) const {
  BitVector Reserved(getNumRegs());
  
  // Always reserved
  markSuperRegs(Reserved, AArch64::WSP);  // Stack pointer
  markSuperRegs(Reserved, AArch64::WZR);  // Zero register
  
  // Conditionally reserved
  if (TFI->isFPReserved(MF))
    markSuperRegs(Reserved, AArch64::W29);  // Frame pointer
  
  if (hasBasePointer(MF))
    markSuperRegs(Reserved, AArch64::W19);  // Base pointer
  
  // Platform-specific reservations
  if (MF.getSubtarget<AArch64Subtarget>().isWindowsArm64EC()) {
    markSuperRegs(Reserved, AArch64::W13);
    markSuperRegs(Reserved, AArch64::W14);
    // ... more platform-specific reservations
  }
  
  return Reserved;
}
```

#### Callee-Saved Registers
```cpp
const MCPhysReg *
AArch64RegisterInfo::getCalleeSavedRegs(const MachineFunction *MF) const {
  // Different calling conventions have different callee-saved registers
  switch (F.getCallingConv()) {
  case CallingConv::C:
    return CSR_AArch64_AAPCS_SaveList;  // X19-X28, D8-D15
  case CallingConv::PreserveMost:
    return CSR_AArch64_RT_MostRegs_SaveList;  // Most registers
  case CallingConv::AArch64_VectorCall:
    return CSR_AArch64_AAVPCS_SaveList;  // Vector calling convention
  // ... more calling conventions
  }
}
```

### 4. AArch64RegisterBanks.td

**Purpose**: Define register banks for GlobalISel.

```tablegen
/// General Purpose Registers: W, X
def GPRRegBank : RegisterBank<"GPR", [XSeqPairsClass]>;

/// Floating Point, Vector, Scalable Vector Registers: B, H, S, D, Q, Z
def FPRRegBank : RegisterBank<"FPR", [QQQQ, ZPR]>;

/// Conditional register: NZCV
def CCRegBank : RegisterBank<"CC", [CCR]>;
```

### 5. AArch64RegisterBankInfo.h / .cpp

**Purpose**: Implement register bank selection logic for GlobalISel.

**Key Methods**:

```cpp
class AArch64RegisterBankInfo final : public AArch64GenRegisterBankInfo {
  // Get instruction mapping (which register bank for each operand)
  const InstructionMapping &
  getInstrMapping(const MachineInstr &MI) const override;
  
  // Get alternative mappings (e.g., GPR vs FPR for bitcast)
  InstructionMappings
  getInstrAlternativeMappings(const MachineInstr &MI) const override;
  
  // Cost of copying between register banks
  unsigned copyCost(const RegisterBank &A, const RegisterBank &B,
                    TypeSize Size) const override;
};
```

**Example Mapping Logic**:
```cpp
// For G_ADD: all operands should be in the same bank
if (Opc == TargetOpcode::G_ADD) {
  LLT Ty = MRI.getType(MI.getOperand(0).getReg());
  if (Ty.isVector())
    return getFPRMapping();  // Vectors go to FPR
  else
    return getGPRMapping();  // Scalars go to GPR
}

// For G_BITCAST: may need cross-bank copy
if (Opc == TargetOpcode::G_BITCAST) {
  // Provide alternatives: GPR→GPR, FPR→FPR, GPR→FPR, FPR→GPR
  return getAlternativeMappings();
}
```

### 6. AArch64DeadRegisterDefinitions.cpp

**Purpose**: Optimize away dead register definitions.

**Functionality**:
- Identifies instructions that define registers that are never used
- Replaces them with zero register (XZR/WZR) to save encoding space
- Example: `MOV X0, #0` → use XZR directly

---

## Register Usage in LLVM

### 1. Register Allocation

LLVM uses several register allocation algorithms:

#### Fast Register Allocator
- **Speed**: Very fast
- **Quality**: Lower quality
- **Use**: Debug builds, quick compilation

#### Greedy Register Allocator (Default)
- **Speed**: Moderate
- **Quality**: High quality
- **Algorithm**: 
  1. Calculate live ranges for virtual registers
  2. Assign physical registers greedily based on spill cost
  3. Split live ranges if necessary
  4. Insert spill/reload code

#### PBQP Register Allocator
- **Speed**: Slower
- **Quality**: Highest quality
- **Algorithm**: Partitioned Boolean Quadratic Programming

### 2. Calling Conventions

AArch64 defines several calling conventions:

#### AAPCS64 (ARM Architecture Procedure Call Standard)
```
Arguments:
  - Integer/Pointer: X0-X7
  - Floating-point: D0-D7 (or Q0-Q7 for vectors)
  - Additional args: Stack

Return values:
  - Integer: X0-X1
  - Floating-point: D0-D3

Callee-saved:
  - X19-X28
  - D8-D15 (lower 64 bits of Q8-Q15)
  - FP (X29), LR (X30)
```

#### Vector Calling Convention
```
Arguments:
  - Vectors: Q0-Q7
  - Scalars: X0-X7

Callee-saved:
  - Q8-Q15 (full 128 bits)
```

#### SVE Calling Convention
```
Arguments:
  - SVE vectors: Z0-Z7
  - SVE predicates: P0-P3

Callee-saved:
  - Z8-Z23 (lower 128 bits only)
  - P4-P15
```

### 3. Register Pressure

Register pressure is the demand for registers at a given point:

```cpp
unsigned AArch64RegisterInfo::getRegPressureLimit(
    const TargetRegisterClass *RC, MachineFunction &MF) const {
  switch (RC->getID()) {
  case AArch64::GPR64RegClassID:
    return 32 - 1                    // XZR/SP
              - (hasFP(MF) ? 1 : 0)  // FP
              - (hasBasePointer(MF) ? 1 : 0)  // X19
              - NumReservedRegs;     // Platform-specific
  case AArch64::FPR128RegClassID:
    return 32;  // All Q registers available
  // ...
  }
}
```

### 4. Spilling and Reloading

When register pressure exceeds available registers:

```
Original:
  %vreg1 = ADD %vreg2, %vreg3
  %vreg4 = MUL %vreg1, %vreg5

After spilling %vreg1:
  %vreg1 = ADD %vreg2, %vreg3
  STR %vreg1, [SP, #offset]    // Spill to stack
  ...
  %vreg1_reload = LDR [SP, #offset]  // Reload from stack
  %vreg4 = MUL %vreg1_reload, %vreg5
```

---

## Advanced Topics

### 1. SubRegister Liveness Tracking

LLVM can track liveness at the subregister level:

```cpp
// Enable subregister liveness
MRI.enableSubRegLiveness(true);

// Example: Only lower 32 bits (W0) are live, not full X0
// This allows better optimization
```

### 2. Register Coalescing

Combines copy instructions by assigning the same physical register:

```
Before coalescing:
  %vreg1 = ...
  %vreg2 = COPY %vreg1
  ... = USE %vreg2

After coalescing (if %vreg1 and %vreg2 get same physical register):
  %vreg1 = ...
  ... = USE %vreg1  // COPY eliminated
```

### 3. Register Hints

Guide register allocation to reduce copies:

```cpp
bool AArch64RegisterInfo::getRegAllocationHints(
    Register VirtReg, ArrayRef<MCPhysReg> Order,
    SmallVectorImpl<MCPhysReg> &Hints, const MachineFunction &MF,
    const VirtRegMap *VRM, const LiveRegMatrix *Matrix) const {
  
  // For SVE predicated instructions, prefer reusing source register
  // to avoid MOVPRFX instruction
  if (isSVEPredicatedInstr(MI)) {
    Hints.push_back(getPhysReg(MI.getOperand(2).getReg()));
  }
  
  // For SME multi-vector instructions, prefer strided registers
  if (isSMEMultiVectorInstr(MI)) {
    // Suggest Z0_Z8, Z1_Z9, etc. instead of Z0_Z1
    Hints.push_back(getStridedRegister(...));
  }
}
```

### 4. Register Scavenging

Finding a free register during frame lowering:

```cpp
// Need a scratch register for large frame offset
Register ScratchReg = RS->FindUnusedReg(&AArch64::GPR64RegClass);
if (!ScratchReg) {
  // No free register, use emergency spill slot
  ScratchReg = RS->scavengeRegister(&AArch64::GPR64RegClass, II, SPAdj);
}
```

### 5. Register Pairs and Tuples

Some instructions require consecutive registers:

```tablegen
// Load pair: LDP X0, X1, [SP]
def XSeqPairs : RegisterTuples<[sube64, subo64],
                               [(decimate (rotl GPR64, 0), 2),
                                (decimate (rotl GPR64, 1), 2)]>;

// SME strided registers: Z0_Z8, Z1_Z9, etc.
def ZStridedPairs : RegisterTuples<[zsub0, zsub1], [
  (trunc (rotl ZPR, 0), 8), (trunc (rotl ZPR, 8), 8)
]>;
```

### 6. Platform-Specific Reservations

Different platforms reserve different registers:

```cpp
// Windows ARM64EC
if (MF.getSubtarget<AArch64Subtarget>().isWindowsArm64EC()) {
  // X13, X14, X23, X24, X28 clobbered by async signals
  markSuperRegs(Reserved, AArch64::W13);
  markSuperRegs(Reserved, AArch64::W14);
  // V16-V31 also clobbered
  for (unsigned i = AArch64::B16; i <= AArch64::B31; ++i)
    markSuperRegs(Reserved, i);
}

// Linux with LFI (Load-Fence-Instruction)
if (MF.getSubtarget<AArch64Subtarget>().isLFI()) {
  // X25-X28 reserved for LFI
  markSuperRegs(Reserved, AArch64::W25);
  markSuperRegs(Reserved, AArch64::W26);
  markSuperRegs(Reserved, AArch64::W27);
  markSuperRegs(Reserved, AArch64::W28);
}
```

### 7. Register Aliasing

Understanding register aliasing is crucial:

```
Physical Register Aliasing:
  X0 aliases W0 (lower 32 bits)
  Q0 aliases D0, S0, H0, B0 (different views)
  Z0 aliases Q0 (lower 128 bits)

Implications:
  - Writing to W0 zeros upper 32 bits of X0
  - Writing to S0 zeros upper 96 bits of Q0
  - Writing to Q0 preserves upper bits of Z0 (SVE)
```

### 8. Register Pressure Tracking

LLVM tracks register pressure to guide optimizations:

```cpp
// Calculate register pressure at each program point
RegPressureTracker RPTracker;
RPTracker.init(&MF, &RegClassInfo, &LIS, &MBB, MBB.begin());

for (MachineInstr &MI : MBB) {
  RPTracker.advance();
  
  // Get current pressure
  RegisterPressure Pressure = RPTracker.getPressure();
  
  // Check if we're exceeding limits
  if (Pressure.getMaxPressure(GPRRegBank) > GPRLimit) {
    // Consider spilling or code motion
  }
}
```

---

## Summary

### Key Takeaways

1. **Registers are the fastest storage** in the CPU hierarchy, used for operands, results, and state.

2. **AArch64 provides multiple register types**:
   - GPRs (W/X) for integer operations
   - FPRs (B/H/S/D/Q) for floating-point and SIMD
   - SVE (Z/P) for scalable vectors
   - SME (ZA/ZT) for matrix operations

3. **Register classes** group registers with similar properties for allocation.

4. **Register banks** (GPR, FPR, CC) are high-level abstractions for GlobalISel.

5. **Register allocation** is a critical optimization that maps virtual registers to physical registers.

6. **Calling conventions** define how registers are used for function calls.

7. **Platform-specific considerations** affect which registers are available.

### File Organization

```
AArch64/
├── AArch64RegisterInfo.td          # TableGen definitions
├── AArch64RegisterBanks.td         # Register bank definitions
├── AArch64RegisterInfo.h           # C++ interface
├── AArch64RegisterInfo.cpp         # Implementation
├── AArch64DeadRegisterDefinitions.cpp  # Optimization pass
└── GISel/
    ├── AArch64RegisterBankInfo.h   # GlobalISel register banks
    └── AArch64RegisterBankInfo.cpp # Bank selection logic
```

### Further Reading

- [ARM Architecture Reference Manual](https://developer.arm.com/documentation/)
- [LLVM Register Allocation Documentation](https://llvm.org/docs/CodeGenerator.html#register-allocation)
- [AArch64 Procedure Call Standard](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst)
- [LLVM GlobalISel Documentation](https://llvm.org/docs/GlobalISel/)

---

*This document provides a comprehensive overview of AArch64 registers in LLVM. For specific implementation details, refer to the source files in the LLVM repository.*
