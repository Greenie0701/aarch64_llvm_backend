# AArch64 Backend: LLVM IR → Assembly

This document traces the complete pipeline from LLVM IR input to AArch64
assembly output, mapping each stage to the files responsible for it.

---

```
LLVM IR (.ll / .bc)
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        STAGE 1: TARGET REGISTRATION                 │
│                                                                     │
│  AArch64TargetMachine.cpp                                           │
│  ├── LLVMInitializeAArch64Target()                                  │
│  │     Registers AArch64leTargetMachine and AArch64beTargetMachine  │
│  │     with the LLVM target registry.                               │
│  │     Initializes all passes (ISel, AsmPrinter, optimizers, etc.)  │
│  │                                                                  │
│  AArch64TargetMachine.h                                             │
│  ├── AArch64TargetMachine        → base class for LE/BE variants    │
│  ├── AArch64leTargetMachine      → little-endian target             │
│  └── AArch64beTargetMachine      → big-endian target                │
│                                                                     │
│  TargetInfo/AArch64TargetInfo.cpp                                   │
│  └── getTheAArch64leTarget(), getTheAArch64beTarget()               │
│        Provides the Target objects used during registration.        │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     STAGE 2: SUBTARGET CREATION                     │
│                                                                     │
│  AArch64TargetMachine.cpp                                           │
│  └── getSubtargetImpl(Function &F)                                  │
│        Reads per-function attributes (target-cpu, target-features,  │
│        streaming mode, SVE vector sizes) and creates or reuses a    │
│        cached AArch64Subtarget for that function.                   │
│                                                                     │
│  AArch64Subtarget.h / AArch64Subtarget.cpp                         │
│  ├── Holds all CPU feature flags (hasNEON, hasSVE, hasSME, etc.)   │
│  ├── Owns instances of:                                             │
│  │     AArch64FrameLowering   → stack frame management             │
│  │     AArch64InstrInfo       → instruction descriptions            │
│  │     AArch64RegisterInfo    → register file description           │
│  │     AArch64TargetLowering  → IR → DAG lowering rules            │
│  │     AArch64SelectionDAGInfo→ DAG memory operation helpers        │
│  └── Owns GlobalISel components (CallLowering, Legalizer, etc.)    │
│                                                                     │
│  AArch64Features.td / AArch64Processors.td                         │
│  └── TableGen sources that define all CPU features and processors.  │
│        Auto-generates AArch64GenSubtargetInfo.inc used by           │
│        ParseSubtargetFeatures().                                    │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STAGE 3: PASS PIPELINE SETUP                     │
│                                                                     │
│  AArch64TargetMachine.cpp → AArch64PassConfig                       │
│  ├── createPassConfig()   → builds the full pass pipeline           │
│  │                                                                  │
│  │  IR-Level Passes (addIRPasses):                                  │
│  │  ├── AtomicExpandPass         → expand atomics to ll/sc loops    │
│  │  ├── SVEIntrinsicOptsPass     → optimize SVE intrinsics          │
│  │  ├── ComplexDeinterleavingPass→ match complex arithmetic         │
│  │  ├── InterleavedAccessPass    → match ldN/stN patterns           │
│  │  ├── SMEABIPass               → SME calling convention setup     │
│  │  └── AArch64StackTaggingPass  → MTE stack tagging                │
│  │                                                                  │
│  │  Pre-ISel Passes (addPreISel):                                   │
│  │  ├── AArch64PromoteConstantPass → hoist constants                │
│  │  └── GlobalMergePass           → merge small globals             │
│  │                                                                  │
│  │  (Two ISel paths exist: SelectionDAG and GlobalISel)             │
└─────────────────────────────────────────────────────────────────────┘
      │
      ├──────────────────────────┐
      │  SelectionDAG Path       │  GlobalISel Path
      ▼                          ▼
┌───────────────────┐   ┌────────────────────────────────────────────┐
│   STAGE 4A:       │   │  STAGE 4B: GLOBALISEL PATH                 │
│  SELECTIONDAG     │   │                                            │
│  ISEL PATH        │   │  GISel/AArch64IRTranslator.cpp             │
│                   │   │  └── IRTranslator pass                     │
│  AArch64ISelLow-  │   │       Translates LLVM IR → generic MIR     │
│  ering.h/.cpp     │   │       (G_ADD, G_LOAD, G_STORE, etc.)       │
│  ├── Inherits     │   │                                            │
│  │   TargetLower- │   │  GISel/AArch64LegalizerInfo.cpp            │
│  │   ing          │   │  └── Legalizer pass                        │
│  ├── LowerOpera-  │   │       Makes generic ops legal for AArch64  │
│  │   tion()       │   │                                            │
│  │   Handles all  │   │  GISel/AArch64RegisterBankInfo.cpp         │
│  │   ISD opcodes  │   │  └── RegBankSelect pass                    │
│  │   (LOAD, STORE │   │       Assigns register banks (GPR vs FPR)  │
│  │   ADD, CALL,   │   │                                            │
│  │   etc.)        │   │  GISel/AArch64InstructionSelector.cpp      │
│  │                │   │  └── InstructionSelect pass                │
│  ├── LowerCall()  │   │       Selects real AArch64 instructions    │
│  ├── LowerReturn()│   │       from generic MIR                     │
│  └── PerformDAG-  │   │                                            │
│      Combine()    │   │  GISel/AArch64PreLegalizerCombiner.cpp     │
│                   │   │  GISel/AArch64PostLegalizerCombiner.cpp    │
│  AArch64ISelDAG-  │   │  └── Combiner passes before/after          │
│  ToDAG.cpp        │   │       legalization                         │
│  └── AArch64DAG-  │   │                                            │
│      ToDAGISel    │   │  GISel/AArch64CallLowering.cpp             │
│      Select()     │   │  └── Lowers calls in GlobalISel path       │
│      Matches DAG  │   └────────────────────────────────────────────┘
│      patterns and │
│      emits real   │
│      MachineInstrs│
│                   │
│  AArch64FastISel  │
│  .cpp             │
│  └── Fast path    │
│      ISel for -O0 │
└───────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STAGE 5: CALLING CONVENTION LOWERING                   │
│                                                                     │
│  AArch64CallingConvention.td / AArch64CallingConvention.h/.cpp      │
│  ├── Defines rules for argument/return value placement:             │
│  │     CC_AArch64_AAPCS   → standard C calling convention          │
│  │     CC_AArch64_Win64   → Windows AArch64 calling convention     │
│  │     CC_AArch64_SVE_*   → SVE vector calling conventions         │
│  └── Auto-generates CCAssignFnForCall / CCAssignFnForReturn         │
│                                                                     │
│  AArch64ISelLowering.cpp                                            │
│  ├── LowerFormalArguments() → incoming args → virtual registers     │
│  ├── LowerCall()            → outgoing call args + return values    │
│  └── LowerReturn()          → function return value placement       │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STAGE 6: MACHINE IR OPTIMIZATION PASSES                │
│                                                                     │
│  These passes operate on MachineInstr / MachineBasicBlock level.    │
│                                                                     │
│  ── SSA Optimization Phase ──────────────────────────────────────   │
│  AArch64MIPeepholeOpt.cpp     → peephole opts on MIR               │
│  AArch64ConditionOptimizer.cpp→ optimize condition codes            │
│  AArch64ConditionalCompares.cpp→ form CCMP/CCMN instructions        │
│  AArch64CondBrTuning.cpp      → tune conditional branches           │
│  AArch64AdvSIMDScalarPass.cpp → use scalar SIMD where profitable    │
│  AArch64SIMDInstrOpt.cpp      → SIMD instruction optimizations      │
│  SMEABIPass.cpp               → SME ABI lowering                   │
│  MachineSMEABIPass.cpp        → machine-level SME ABI              │
│  SMEPeepholeOpt.cpp           → SME peephole optimizations          │
│  SVEIntrinsicOpts.cpp         → SVE intrinsic optimizations         │
│                                                                     │
│  ── Pre-Register Allocation ─────────────────────────────────────   │
│  AArch64DeadRegisterDefinitions.cpp → replace dead defs with XZR   │
│  AArch64StorePairSuppress.cpp       → suppress unprofitable STP     │
│  AArch64StackTaggingPreRA.cpp       → pre-RA MTE stack tagging      │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   STAGE 7: REGISTER ALLOCATION                      │
│                                                                     │
│  AArch64RegisterInfo.h / AArch64RegisterInfo.cpp                    │
│  ├── Inherits AArch64GenRegisterInfo (from AArch64RegisterInfo.td)  │
│  ├── getCalleeSavedRegs()   → which regs must be saved/restored     │
│  ├── getReservedRegs()      → SP, FP, XZR, platform-reserved regs  │
│  ├── eliminateFrameIndex()  → replace frame indices with real offsets│
│  └── getRegAllocationHints()→ hints for the register allocator      │
│                                                                     │
│  AArch64RegisterInfo.td                                             │
│  └── TableGen source defining all registers:                        │
│       X0-X30, W0-W30, SP, FP, LR                                   │
│       V0-V31 (NEON), Z0-Z31 (SVE), P0-P15 (SVE predicates)        │
│       ZA, ZT0 (SME matrix registers)                                │
│                                                                     │
│  AArch64PBQPRegAlloc.h / AArch64PBQPRegAlloc.cpp                   │
│  └── PBQP-based register allocation constraints for AArch64        │
│                                                                     │
│  AArch64PostCoalescerPass.cpp                                       │
│  └── Post-coalescer cleanup pass                                    │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                STAGE 8: PROLOGUE / EPILOGUE INSERTION               │
│                                                                     │
│  AArch64FrameLowering.h / AArch64FrameLowering.cpp                  │
│  ├── emitPrologue()                                                 │
│  │     Emits: STP x29, x30, [sp, #-N]!                             │
│  │            MOV x29, sp                                           │
│  │            SUB sp, sp, #LocalSize                                │
│  ├── emitEpilogue()                                                 │
│  │     Emits: ADD sp, sp, #LocalSize                                │
│  │            LDP x29, x30, [sp], #N                                │
│  │            RET                                                   │
│  ├── spillCalleeSavedRegisters() → emit STP/STR for callee-saves    │
│  ├── restoreCalleeSavedRegisters()→ emit LDP/LDR for callee-saves   │
│  ├── determineCalleeSaves()      → decide which regs need saving    │
│  └── getFrameIndexReference()    → compute SP/FP-relative offsets   │
│                                                                     │
│  AArch64PrologueEpilogue.h / AArch64PrologueEpilogue.cpp           │
│  └── Helpers for prologue/epilogue emission                         │
│                                                                     │
│  AArch64MachineFunctionInfo.h / AArch64MachineFunctionInfo.cpp      │
│  └── Per-function state: stack sizes, varargs info, LOH directives, │
│       SVE/SME stack sizes, PAC-RET signing info, etc.               │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STAGE 9: POST-RA OPTIMIZATION PASSES                   │
│                                                                     │
│  AArch64RedundantCopyElimination.cpp → remove redundant copies      │
│  AArch64A57FPLoadBalancing.cpp       → FP load balancing for A57    │
│  AArch64LoadStoreOptimizer.cpp       → merge LDR/STR → LDP/STP     │
│  AArch64ExpandPseudoInsts.cpp        → expand pseudo instructions   │
│  AArch64LowerHomogeneousPrologEpilog.cpp → homogeneous prolog/epilog│
│  AArch64CompressJumpTables.cpp       → compress jump table entries  │
│  AArch64RedundantCondBranchPass.cpp  → remove redundant cond branches│
│  AArch64A53Fix835769.cpp             → Cortex-A53 erratum 835769 fix│
│  AArch64FalkorHWPFFix.cpp            → Falkor HW prefetch fix       │
│  AArch64SpeculationHardening.cpp     → Spectre mitigations          │
│  AArch64SLSHardening.cpp             → Straight-line speculation fix │
│  AArch64PointerAuth.cpp              → PAC-RET pointer auth         │
│  AArch64BranchTargets.cpp            → BTI branch target insertion  │
│  AArch64StackTagging.cpp             → MTE stack tagging            │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  STAGE 10: INSTRUCTION SCHEDULING                   │
│                                                                     │
│  AArch64MachineScheduler.h / AArch64MachineScheduler.cpp            │
│  └── AArch64PostRASchedStrategy                                     │
│       Custom post-RA scheduling strategy for AArch64               │
│                                                                     │
│  AArch64MacroFusion.h / AArch64MacroFusion.cpp                      │
│  └── createAArch64MacroFusionDAGMutation()                          │
│       Fuses instruction pairs that execute as one on modern CPUs    │
│       (e.g. ADRP+ADD, CMP+branch, AES pairs)                       │
│                                                                     │
│  AArch64Sched*.td (many files)                                      │
│  ├── AArch64SchedA53.td, AArch64SchedA57.td, AArch64SchedA55.td    │
│  ├── AArch64SchedNeoverseN1.td, AArch64SchedNeoverseV1.td, etc.    │
│  └── Define per-CPU pipeline models, latencies, and resource usage  │
│       used by the MachineScheduler to order instructions.           │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STAGE 11: MACHINEINSTR → MC LOWERING                   │
│                                                                     │
│  AArch64MCInstLower.h / AArch64MCInstLower.cpp                      │
│  └── AArch64MCInstLower::Lower()                                    │
│       Converts each MachineInstr → MCInst                           │
│       Handles: global addresses, block addresses, jump tables,      │
│       external symbols, target flags (MO_PAGE, MO_PAGEOFF, etc.)   │
│                                                                     │
│  AArch64AsmPrinter.cpp                                              │
│  ├── runOnMachineFunction() → entry point per function              │
│  ├── emitInstruction()      → dispatches each MachineInstr          │
│  │     Handles pseudo expansions:                                   │
│  │     ├── LowerJumpTableDest()    → ADR + LDR + ADD sequence       │
│  │     ├── LowerMOPS()            → MOPS memory operation sequences │
│  │     ├── LowerKCFI_CHECK()      → KCFI type hash check            │
│  │     ├── LowerHWASAN_CHECK_MEMACCESS() → HWASan checks            │
│  │     ├── emitPtrauthBranch()    → PAC authenticated branches      │
│  │     ├── emitPtrauthSign()      → PAC pointer signing             │
│  │     └── emitFMov0()            → zero FP register sequences      │
│  ├── emitFunctionEntryLabel()  → function symbol + variant PCS      │
│  ├── emitStartOfAsmFile()      → ELF build attributes, GNU props    │
│  ├── emitEndOfAsmFile()        → HWASan stubs, LOH directives       │
│  └── emitLOHs()               → Linker Optimization Hints (MachO)  │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  STAGE 12: MC LAYER (ASSEMBLY EMISSION)             │
│                                                                     │
│  MCTargetDesc/AArch64MCTargetDesc.cpp                               │
│  └── Registers MC components: MCAsmInfo, MCInstrInfo,               │
│       MCRegisterInfo, MCSubtargetInfo, MCInstPrinter                │
│                                                                     │
│  MCTargetDesc/AArch64MCAsmInfo.cpp                                  │
│  └── Defines assembly syntax rules:                                 │
│       comment characters, directive names, label format, etc.       │
│                                                                     │
│  MCTargetDesc/AArch64InstPrinter.cpp                                │
│  └── Converts MCInst → text assembly string                         │
│       e.g. MCInst{ADD, X0, X1, X2} → "add x0, x1, x2"             │
│                                                                     │
│  MCTargetDesc/AArch64MCCodeEmitter.cpp                              │
│  └── Converts MCInst → binary machine code bytes                    │
│       Used when emitting object files directly (not .s text)        │
│                                                                     │
│  MCTargetDesc/AArch64AsmBackend.cpp                                 │
│  └── Handles relocations, fixups, and relaxation                    │
│       e.g. branch offset fixups, ADRP page relocations              │
│                                                                     │
│  MCTargetDesc/AArch64AddressingModes.h                              │
│  └── Encodes/decodes shift amounts, extend types, logical immediates│
│                                                                     │
│  MCTargetDesc/AArch64MCAsmInfo.h                                    │
│  └── MCAsmInfo subclass for AArch64 assembly syntax                 │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                STAGE 13: OBJECT FILE / SECTION LAYOUT               │
│                                                                     │
│  AArch64TargetObjectFile.h / AArch64TargetObjectFile.cpp            │
│  └── Decides which ELF/MachO/COFF section each global goes into     │
│       e.g. .text, .rodata, .data, .bss, __auth_ptr (MachO PAC)     │
│                                                                     │
│  AArch64CollectLOH.cpp                                              │
│  └── Collects Linker Optimization Hints for MachO:                  │
│       Identifies ADRP+LDR / ADRP+ADD pairs for the linker          │
└─────────────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        FINAL OUTPUT                                 │
│                                                                     │
│   .s  file  → AArch64 text assembly (via MCStreamer → raw_ostream)  │
│   .o  file  → ELF/MachO/COFF object (via MCObjectStreamer)          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Supporting Infrastructure (Used Across All Stages)

```
AArch64InstrInfo.h / AArch64InstrInfo.cpp
├── Instruction properties: size, latency, scheduling class
├── analyzeCompare(), optimizeCompareInstr() → flag optimization
├── analyzeBranch(), insertBranch(), removeBranch() → branch management
├── copyPhysReg(), storeRegToStackSlot(), loadRegFromStackSlot()
├── foldMemoryOperandImpl() → fold spills into instructions
├── getMachineCombinerPatterns() → mul-add fusion patterns
└── getOutliningCandidateInfo() → machine outliner support

AArch64InstrInfo.td / AArch64InstrFormats.td
├── TableGen definitions of every AArch64 instruction
├── Encoding, operand types, scheduling class, flags
└── Pattern matching rules for SelectionDAG ISel

AArch64InstrAtomics.td   → atomic instruction definitions
AArch64SVEInstrInfo.td   → SVE instruction definitions
AArch64SMEInstrInfo.td   → SME instruction definitions
SVEInstrFormats.td       → SVE instruction format templates
SMEInstrFormats.td       → SME instruction format templates
AArch64InstrGISel.td     → GlobalISel-specific patterns

AArch64SelectionDAGInfo.h / AArch64SelectionDAGInfo.cpp
└── Custom lowering for memcpy/memset/memmove using NEON/SVE

AArch64TargetTransformInfo.h / AArch64TargetTransformInfo.cpp
└── Cost model for vectorization, unrolling, inlining decisions
     Used by the middle-end optimizer (not code generation itself)

AArch64ExpandImm.h / AArch64ExpandImm.cpp
└── Expands large immediates into MOV/MOVK instruction sequences

AArch64SMEAttributes.h / AArch64SMEAttributes.cpp
└── Parses and stores SME function attributes
     (streaming mode, ZA state, ZT0 state)

Utils/AArch64BaseInfo.h
└── Shared enums and utilities: system registers, PAC keys,
     condition codes, shift/extend types, barrier options

AArch64.h
└── Forward declarations and pass creation functions
     (createAArch64ISelDag, createAArch64LoadStoreOptimizationPass, etc.)

AArch64.td
└── Top-level TableGen file that includes all other .td files
     and defines the AArch64 target

AArch64Combine.td
└── TableGen-driven DAG combine patterns for GlobalISel

AArch64RegisterBanks.td / AArch64GenRegisterBankInfo.def
└── Register bank definitions for GlobalISel (GPR, FPR)

AArch64SystemOperands.td
└── Definitions of system registers (FPCR, FPSR, TPIDR_EL0, etc.)

AArch64PerfectShuffle.h
└── Lookup table for optimal NEON shuffle sequences

AArch64PointerAuth.h / AArch64PointerAuth.cpp
└── PAC-RET pointer authentication pass and utilities

AArch64CleanupLocalDynamicTLSPass.cpp
└── Combines multiple _TLS_MODULE_BASE_ references (ELF TLS)

AArch64PromoteConstant.cpp
└── Promotes constant vectors to global variables to avoid
     repeated materialization in hot code

AArch64MIPeepholeOpt.cpp
└── Machine IR peephole: combines shifts, extends, bitfield ops

AArch64PostCoalescerPass.cpp
└── Cleans up after register coalescing

AArch64Arm64ECCallLowering.cpp
└── Windows ARM64EC (x64 emulation) call lowering

AArch64PBQPRegAlloc.h / AArch64PBQPRegAlloc.cpp
└── PBQP register allocation constraints (e.g. A57 FP pairing)

AArch64MacroFusion.h / AArch64MacroFusion.cpp
└── Macro-fusion mutation for the machine scheduler

AArch64MachineScheduler.h / AArch64MachineScheduler.cpp
└── Custom post-RA scheduling strategy

AArch64SelectionDAGInfo.h / AArch64SelectionDAGInfo.cpp
└── Custom SelectionDAG memory operation lowering

AArch64ExpandImm.h / AArch64ExpandImm.cpp
└── Immediate expansion into MOV/MOVK sequences

Disassembler/AArch64Disassembler.cpp
└── Decodes binary AArch64 instructions back to MCInst
     (used by llvm-objdump, not in the compilation pipeline)

AsmParser/AArch64AsmParser.cpp
└── Parses AArch64 text assembly → MCInst
     (used by llvm-mc / inline asm, not in the compilation pipeline)
```

---

## Key Data Structure Flow

```
LLVM IR (Function, BasicBlock, Instruction)
    │
    │  [SelectionDAGBuilder / IRTranslator]
    ▼
SelectionDAG (SDNode, SDValue)          ← AArch64ISelLowering.cpp operates here
    │
    │  [AArch64DAGToDAGISel::Select()]
    ▼
MachineInstr / MachineBasicBlock        ← AArch64InstrInfo.cpp operates here
    │
    │  [AArch64FrameLowering, Register Allocator, Optimizers]
    ▼
MachineInstr (physical registers, real offsets)
    │
    │  [AArch64MCInstLower::Lower()]
    ▼
MCInst                                  ← MCTargetDesc layer operates here
    │
    │  [AArch64InstPrinter]             │  [AArch64MCCodeEmitter]
    ▼                                   ▼
Text Assembly (.s)                  Binary Object (.o)
```
 
