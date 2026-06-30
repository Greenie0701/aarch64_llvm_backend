This repo contains high level explanation for llvm AARCH64 backend based on llvm 22.x

```text
┌─────────────────────────────────────────────────────────────────┐
│              RECOMMENDED LEARNING ORDER                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 1: Registers & Instructions                               │
│  ─────────────────────────────────                              │
│  □ Learn all register names and their roles                     │
│  □ Write simple ARM64 assembly by hand                          │
│  □ Understand integer and load/store instructions               │
│  □ Tool: use godbolt.org with -target aarch64 to see output     │
│                                                                 │
│  WEEK 2: Calling Convention & Stack                             │
│  ──────────────────────────────────                             │
│  □ Trace through a function call step by step                   │
│  □ Draw the stack frame for a function you write                │
│  □ Understand caller vs callee saved registers                  │
│  □ Implement a simple function call in assembly                 │
│                                                                 │
│  WEEK 3: Memory & Addressing                                    │
│  ─────────────────────────────                                  │
│  □ Understand all addressing modes                              │
│  □ Learn the ADRP + ADD pattern for global access               │
│  □ Understand alignment requirements                            │
│  □ Read about weak memory model basics                          │
│                                                                 │
│  WEEK 4: Relocation & Linking                                   │
│  ────────────────────────────                                   │
│  □ Use objdump -r to inspect relocation entries                 │
│  □ Understand R_AARCH64_CALL26 and ADRP relocations             │
│  □ Learn GOT and PLT for dynamic linking                        │
│  □ Read ELF spec for AArch64                                    │
│                                                                 │
│  WEEK 5: LLVM Backend Internals                                 │
│  ──────────────────────────────                                 │
│  □ Read AArch64 .td files in llvm/lib/Target/AArch64/           │
│  □ Understand TableGen pattern matching                         │
│  □ Trace SelectionDAG for a simple function                     │
│  □ Read AArch64FrameLowering.cpp for stack layout               │
│                                                                 │
│  REFERENCE FILES IN LLVM SOURCE:                                │
│  llvm/lib/Target/AArch64/                                       │
│    AArch64.td                 ; top-level TableGen              │
│    AArch64RegisterInfo.td     ; register definitions            │
│    AArch64InstrInfo.td        ; instruction definitions         │
│    AArch64ISelLowering.cpp    ; IR → SelectionDAG lowering      │
│    AArch64ISelDAGToDAG.cpp    ; DAG → MachineInstr selection    │
│    AArch64FrameLowering.cpp   ; stack frame management          │
│    AArch64CallingConvention.td; calling convention rules        │
│    AArch64AsmPrinter.cpp      ; emit final assembly/binary      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
