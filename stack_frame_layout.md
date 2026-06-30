> **File:** `AArch64FrameLowering.cpp`  
> **Total lines:** 3645  
> **Purpose:** Implements the AArch64-specific stack frame layout, prologue/epilogue
> generation, callee-saved register spilling/restoring, alignment handling,
> and all related stack management for the LLVM AArch64 backend.

---

## Table of Contents

1. [Block 1 — File Header and Stack Frame Layout Diagram (Lines 1–303)](#block-1--file-header-and-stack-frame-layout-diagram-lines-1303)
2. [Block 2 — Includes and Command-Line Options (Lines 170–303)](#block-2--includes-and-command-line-options-lines-170303)
3. [Block 3 — `getArgumentStackToRestore` (Lines 306–332)](#block-3--getargumentstacktorestore-lines-306332)
4. [Block 4 — SVE Stack Size Helpers (Lines 344–397)](#block-4--sve-stack-size-helpers-lines-344397)
5. [Block 5 — `homogeneousPrologEpilog` and `producePairRegisters` (Lines 399–455)](#block-5--homogeneousprologepilog-and-producepairregisters-lines-399455)
6. [Block 6 — `estimateRSStackSizeLimit` and `DefaultSafeSPDisplacement` (Lines 457–491)](#block-6--estimatersstacksizelimit-and-defaultsafespdisplacement-lines-457491)
7. [Block 7 — `getFixedObjectSize` (Lines 493–531)](#block-7--getfixedobjectsize-lines-493531)
8. [Block 8 — `canUseRedZone` (Lines 533–561)](#block-8--canuseredzone-lines-533561)
9. [Block 9 — `hasFPImpl` — When Does a Function Need a Frame Pointer? (Lines 563–617)](#block-9--hafpimpl--when-does-a-function-need-a-frame-pointer-lines-563617)
10. [Block 10 — `isFPReserved` (Lines 619–642)](#block-10--isfpreserved-lines-619642)
11. [Block 11 — `hasReservedCallFrame` (Lines 644–652)](#block-11--hasreservedcallframe-lines-644652)
12. [Block 12 — `eliminateCallFramePseudoInstr` (Lines 654–715)](#block-12--eliminatecallframepseudoinstr-lines-654715)
13. [Block 13 — `resetCFIToInitialState` (Lines 717–748)](#block-13--resetcfitoinitialstate-lines-717748)
14. [Block 14 — `getRegisterOrZero` helper (Lines 750–825)](#block-14--getregisteorzero-helper-lines-750825)
15. [Block 15 — `emitZeroCallUsedRegs` (Lines 827–876)](#block-15--emitzerocallusedregs-lines-827876)
16. [Block 16 — `windowsRequiresStackProbe` (Lines 878–886)](#block-16--windowsrequiresstackprobe-lines-878886)
17. [Block 17 — `findScratchNonCalleeSaveRegister` (Lines 888–930)](#block-17--findscratchnoncalleesaveregister-lines-888930)
18. [Block 18 — `canUseAsPrologue` (Lines 932–971)](#block-18--canuseasprolouge-lines-932971)
19. [Block 19 — `needsWinCFI` and `shouldSignReturnAddressEverywhere` (Lines 973–990)](#block-19--needswincfi-and-shouldsignreturnaddresseverywhere-lines-973990)
20. [Block 20 — `insertSEH` (Lines 992–1167)](#block-20--insertseh-lines-9921167)
21. [Block 21 — `requiresSaveVG` (Lines 1169–1179)](#block-21--requiressavevg-lines-11691179)
22. [Block 22 — `emitPacRetPlusLeafHardening` (Lines 1181–1212)](#block-22--emitpacretplusleafhardening-lines-11811212)
23. [Block 23 — `emitPrologue` and `emitEpilogue` (Lines 1214–1229)](#block-23--emitprologue-and-emitepilogue-lines-12141229)
24. [Block 24 — CFI Fixup Helpers (Lines 1226–1235)](#block-24--cfi-fixup-helpers-lines-12261235)
25. [Block 25 — Frame Index Reference Functions (Lines 1237–1359)](#block-25--frame-index-reference-functions-lines-12371359)
26. [Block 26 — `resolveFrameOffsetReference` (Lines 1361–1536)](#block-26--resolveframeoffsetreference-lines-13611536)
27. [Block 27 — Register Pairing Helpers (Lines 1538–1667)](#block-27--register-pairing-helpers-lines-15381667)
28. [Block 28 — `computeCalleeSaveRegisterPairs` (Lines 1669–1954)](#block-28--computecalleesaveregisterpairs-lines-16691954)
29. [Block 29 — `spillCalleeSavedRegisters` (Lines 1956–2173)](#block-29--spillcalleesavedregisters-lines-19562173)
30. [Block 30 — `restoreCalleeSavedRegisters` (Lines 2175–2318)](#block-30--restorecalleesavedregisters-lines-21752318)
31. [Block 31 — Frame ID Helpers (Lines 2320–2354)](#block-31--frame-id-helpers-lines-23202354)
32. [Block 32 — `determineStackHazardSlot` (Lines 2356–2500)](#block-32--determinestackhazardslot-lines-23562500)
33. [Block 33 — `determineCalleeSaves` (Lines 2502–2896)](#block-33--determinecalleesaves-lines-25022896)
34. [Block 34 — `determineSVEStackSizes` (Lines 2898–3511)](#block-34--determinesvestablesizes-lines-28983511)
35. [Block 35 — `getFrameIndexReferencePreferSP` (Lines 3513–3645)](#block-35--getframeindexreferencepreferssp-lines-35133645)
36. [Full Conceptual Map](#full-conceptual-map)

---

## Block 1 — File Header and Stack Frame Layout Diagram (Lines 1–303)

**Lines:** `1 – 303`

This is the most important comment in the entire file. It draws the exact memory
layout of an AArch64 stack frame and explains every region.

### What a stack frame is

When a function is called, it needs memory for local variables, registers it must
preserve, and space to call other functions. That memory is called the **stack frame**.
The stack grows **downward** on AArch64 — lower addresses are at the bottom.

### The layout (after prologue runs)

```
Higher address (top of memory)
|-----------------------------------|
| arguments passed on the stack     |  <- caller put these here before calling
|-----------------------------------|
| (Win64 only) varargs from reg     |  <- Windows: register args saved to stack
|-----------------------------------|
| (Win64 only) callee-saved SVE reg |  <- Windows SVE callee saves
|-----------------------------------|
| callee-saved GPR registers        |  <- x19-x28 saved by this function
|- - - - - - - - - - - - - - - - - -|
| prev_lr  (link register)          |  <- return address
| prev_fp  (frame pointer)          |  <- previous frame pointer = frame record
|-----------------------------------| <- fp (x29) points here
| callee-saved FP/SIMD/SVE regs     |  <- d8-d15, z8-z15, p4-p15
|-----------------------------------|
| SVE stack objects                 |  <- scalable-size local variables
|-----------------------------------|
| alignment padding                 |  <- padding to meet alignment requirements
|-----------------------------------|
| local variables (fixed size)      |  <- int x, arrays, spill slots
|   GPR spills                      |
|   FPR spills                      |
|-----------------------------------| <- bp (base pointer, x19 if needed)
| VLAs (variable-length arrays)     |  <- size unknown at compile time
|-----------------------------------| <- sp (stack pointer)
Lower address (bottom of memory)
```

### Key pointers

| Pointer | Register | Purpose |
|---------|----------|---------|
| `sp` | stack pointer | Always points to current top of stack |
| `fp` | x29 | Points to the frame record (saved fp + lr) |
| `bp` | x19 (LLVM choice) | Stable base when VLAs exist |
| `lr` | x30 | Holds the return address |

### Why three pointers?

- `sp` accesses local variables of fixed size
- `fp` accesses arguments and callee-saved registers
- `bp` is needed when VLAs exist because `sp` moves around dynamically

### Prologue example from the comment

```asm
stp  x28, x27, [sp, #offset-32]
stp  fp,  lr,  [sp, #offset-16]
add  fp,  sp,  #offset - 16
sub  sp,  sp,  #1360
```

---

## Block 2 — Includes and Command-Line Options (Lines 170–303)

**Lines:** `170 – 303`

### Includes

The file includes LLVM infrastructure headers for:

- `MachineFunction`, `MachineBasicBlock`, `MachineInstr` — the machine-level IR
- `MachineFrameInfo` — stores per-function frame layout data
- `RegisterScavenging` — finds free registers at runtime
- `LivePhysRegs` — tracks which physical registers are live
- `CFIInstBuilder` — builds DWARF CFI directives
- `WinEHFuncInfo` — Windows exception handling data

### Command-line options

These are LLVM flags that control backend behavior at compile time.
Pass them like: `llc -aarch64-redzone myfile.ll`

| Flag | Default | Purpose |
|------|---------|---------|
| `aarch64-redzone` | off | Enable 128-byte red zone for leaf functions |
| `stack-tagging-merge-settag` | on | Merge `settag` instructions in epilogue |
| `aarch64-order-frame-objects` | on | Sort stack objects to improve layout |
| `aarch64-split-sve-objects` | on | Separate ZPR and PPR stack regions |
| `homogeneous-prolog-epilog` | off | Emit single helper call for prolog/epilog |
| `aarch64-stack-hazard-remark-size` | 0 | Threshold for hazard remarks |
| `aarch64-stack-hazard-in-non-streaming` | off | Insert hazard padding in non-streaming functions |
| `aarch64-disable-multivector-spill-fill` | off | Disable LD/ST pairs for SME2/SVE2p1 |

---

## Block 3 — `getArgumentStackToRestore` (Lines 306–332)

**Lines:** `306 – 332`  
**Function:** `AArch64FrameLowering::getArgumentStackToRestore`

### What it does

When a function returns, it may need to pop arguments off the stack that were
pushed by the caller. This function calculates how many bytes to restore.

### Two cases

**Case 1 — Tail call return:**
The tail call instruction carries the stack adjustment amount as an operand.
Read it directly from the instruction.

**Case 2 — Normal return:**
The amount was pre-calculated during argument lowering and stored in
`AArch64FunctionInfo`. Just read it back.

### Visual

```
Before return:
  sp -> [local vars] [callee saves] [arguments from caller]

After return:
  sp -> [arguments from caller]   <- sp moved up by ArgumentPopSize
```

---

## Block 4 — SVE Stack Size Helpers (Lines 344–397)

**Lines:** `344 – 397`  
**Functions:** `getStackHazardSize`, `getZPRStackSize`, `getPPRStackSize`,
`isLikelyToHaveSVEStack`, `isTargetWindows`, `hasSVECalleeSavesAboveFrameRecord`

### What SVE stack sizes are

SVE (Scalable Vector Extension) registers have sizes that are not known at
compile time. They are expressed as multiples of `vscale` (the runtime vector
length).

- **ZPR** — SVE vector registers (z0–z31), each `vscale * 16` bytes
- **PPR** — SVE predicate registers (p0–p15), each `vscale * 2` bytes

`StackOffset` has two components:
- **Fixed** — known at compile time (bytes)
- **Scalable** — known only at runtime (multiples of `vscale * 16`)

### `getStackHazardSize`

Returns the SME stack hazard padding size from the subtarget.
Used to insert a buffer between GPR and FPR stack regions.

### `isLikelyToHaveSVEStack`

Conservative check: returns `true` if the function probably has SVE objects
on the stack. Safe to call before callee-saves are determined.

---

## Block 5 — `homogeneousPrologEpilog` and `producePairRegisters` (Lines 399–455)

**Lines:** `399 – 455`

### `homogeneousPrologEpilog` (Lines 399–447)

Returns `true` if the function can use a single helper call for its
prologue/epilogue instead of individual spill/restore instructions.

**Purpose:** Code size optimization. Instead of emitting:
```asm
stp x22, x21, [sp, #0]
stp x20, x19, [sp, #16]
stp fp,  lr,  [sp, #32]
```
It emits a single call to a shared helper routine.

**Conditions that prevent this:**
- Function does not have `minsize` attribute
- Flag `homogeneous-prolog-epilog` is not set
- Red zone is enabled
- Target is Windows (not yet supported)
- SVE stack objects exist (not yet supported)
- VLAs or stack realignment exist
- Swift async context or streaming mode changes exist
- Odd number of GPRs before LR/FP in the callee-save list

### `producePairRegisters` (Lines 449–455)

Returns `true` if callee-saved registers should be stored in pairs.
Required for MachO compact unwind format and homogeneous prolog/epilog.

---

## Block 6 — `estimateRSStackSizeLimit` and `DefaultSafeSPDisplacement` (Lines 457–491)

**Lines:** `457 – 491`

### `DefaultSafeSPDisplacement = 255`

The maximum byte offset from `sp` that can be encoded directly in an AArch64
load/store instruction without needing a scratch register.

```
ldr x0, [sp, #255]   <- fits in immediate field, no scratch needed
ldr x0, [sp, #256]   <- too large, needs scratch register
```

### `estimateRSStackSizeLimit`

Scans all instructions that reference stack frame objects and returns the
safe stack size limit. If any instruction cannot encode its frame offset
legally, returns 0 (forcing a scratch register to be reserved).

---

## Block 7 — `getFixedObjectSize` (Lines 493–531)

**Lines:** `493 – 531`  
**Function:** `AArch64FrameLowering::getFixedObjectSize`

### What fixed objects are

Fixed objects are stack slots at **positive offsets from the frame pointer** —
they are above the frame record. These include:

- Tail call reserved stack space
- Windows varargs GPR save area
- Windows EH catch object slots
- Windows EH unwind helper object (8 bytes)

### What this function does

Calculates the total size of the fixed object area, aligned to 16 bytes.
On non-Windows or funclet frames, this is just the tail call reserved stack.
On Windows, it also includes varargs and EH catch objects.

---

## Block 8 — `canUseRedZone` (Lines 533–561)

**Lines:** `533 – 561`  
**Function:** `AArch64FrameLowering::canUseRedZone`

### What the red zone is

The AArch64 ABI defines a **128-byte red zone** below the stack pointer.

```
sp  ->  [current frame]
        [red zone: 128 bytes, safe to use without adjusting sp]
        [undefined below]
```

Leaf functions can use this area for small local variables **without adjusting
`sp` at all**, saving the `sub sp, sp, #N` instruction in the prologue.

### When it cannot be used

| Condition | Reason |
|-----------|--------|
| Function makes calls | `sp` might be clobbered by callee |
| Function has a frame pointer | `fp` setup changes `sp` semantics |
| Local data exceeds 128 bytes | Does not fit |
| SVE stack objects exist | Scalable size, cannot fit in red zone |
| NEON copy requires memory spill | Specific hardware limitation |
| `EnableRedZone` flag is off | Disabled by default |

---

## Block 9 — `hasFPImpl` — When Does a Function Need a Frame Pointer? (Lines 563–617)

**Lines:** `563 – 617`  
**Function:** `AArch64FrameLowering::hasFPImpl`

### What the frame pointer is

The frame pointer (`x29` / `fp`) points to a fixed location in the stack frame —
the **frame record** (saved `fp` and `lr`).

```
fp -> [saved lr]
      [saved fp]
```

It gives a stable reference point for accessing the stack even when `sp` moves.

### When a frame pointer is required

| Condition | Reason |
|-----------|--------|
| EH funclets exist | Windows EH requires FP for local access |
| `-fno-omit-frame-pointer` | User/compiler option |
| VLAs exist | `sp` is not stable |
| Stack maps or patch points | Debugger/profiler requirements |
| Stack realignment needed | Dynamic alignment padding between SP and CSR area |
| Streaming mode changes (SME) | VG value may differ during unwind |
| Large call frames | Emergency spill slot may be out of SP range |
| DWARF unwind with SVE objects | CFA expression requires stable base |

### When it can be omitted

- Leaf functions with small fixed frames
- No VLAs, no EH, no stack realignment

---

## Block 10 — `isFPReserved` (Lines 619–642)

**Lines:** `619 – 642`  
**Function:** `AArch64FrameLowering::isFPReserved`

### What this means

A reserved frame pointer means `x29` is **always** treated as the frame pointer
register, even if the current function does not actively use it.

### When FP is reserved

- **Darwin (macOS/iOS):** Always. The OS requires a valid frame chain.
- **Windows:** Always. Required for stack walking.
- **Any function that has FP:** Automatically reserved.
- **Frontend request:** Via `FramePointerIsReserved` target option.

---

## Block 11 — `hasReservedCallFrame` (Lines 644–652)

**Lines:** `644 – 652`  
**Function:** `AArch64FrameLowering::hasReservedCallFrame`

### What a reserved call frame is

When a function calls other functions, it needs to put arguments on the stack.
There are two strategies:

**Strategy 1 — Reserved call frame (no VLAs):**
Allocate space for the largest possible outgoing call at function entry.
No `sp` adjustment needed around each call.

**Strategy 2 — Dynamic call frame (VLAs exist):**
Adjust `sp` before and after each call.
Required when VLAs exist because `sp` is already moving around.

This function returns `true` (use strategy 1) when there are no VLAs.

---

## Block 12 — `eliminateCallFramePseudoInstr` (Lines 654–715)

**Lines:** `654 – 715`  
**Function:** `AArch64FrameLowering::eliminateCallFramePseudoInstr`

### What pseudo instructions are

During instruction selection, LLVM inserts pseudo instructions:

```
ADJCALLSTACKDOWN  #size   <- "reserve this space before a call"
ADJCALLSTACKUP    #size   <- "release this space after a call"
```

These are not real AArch64 instructions. This function **replaces them** with
real `sub sp` / `add sp` instructions.

### Visual

```
Before elimination:
  ADJCALLSTACKDOWN #32
  BL some_function
  ADJCALLSTACKUP #32

After elimination (no reserved call frame):
  sub sp, sp, #32
  BL some_function
  add sp, sp, #32

After elimination (reserved call frame):
  (nothing — space was already reserved in prologue)
```

### Stack probing case

If stack probing is enabled and the call frame is large enough (>= 1024 bytes),
the `sub sp` is replaced with an inline stack probe sequence that touches each
page to trigger OS mapping.

---

## Block 13 — `resetCFIToInitialState` (Lines 717–748)

**Lines:** `717 – 748`  
**Function:** `AArch64FrameLowering::resetCFIToInitialState`

### What CFI is

CFI = **Call Frame Information** — DWARF debug/unwind directives that describe
the stack frame layout. Used by:
- Debuggers (to show stack traces)
- Exception handling (to unwind the stack)
- Profilers

### What this function does

At certain points (e.g. after an exception handler returns), the CFI state needs
to be reset to the function entry state. This emits:

```asm
.cfi_def_cfa sp, 0          ; CFA is now sp+0
.cfi_same_value x30         ; lr is back to its original value
.cfi_same_value x29         ; fp is back to its original value
.cfi_same_value x19         ; etc. for all callee-saved registers
```

It also handles:
- Return address signing state (PAC/PAuth)
- Shadow call stack (`x18`) reset

---

## Block 14 — `getRegisterOrZero` helper (Lines 750–825)

**Lines:** `750 – 825`  
**Function:** `static MCRegister getRegisterOrZero`

### What this does

A helper used by `emitZeroCallUsedRegs`. Given any register (e.g. `w5`, `b3`,
`s7`, `d12`, `q0`), it returns the canonical 64-bit or full-width register to
zero out.

- For GPRs: always returns the 64-bit `x` form (zeroing `x5` also zeros `w5`)
- For FPRs: returns `z` register if SVE is available, otherwise `q` register
- For callee-saved registers (x19–x28, fp, lr): returns 0 (do not zero these)

---

## Block 15 — `emitZeroCallUsedRegs` (Lines 827–876)

**Lines:** `827 – 876`  
**Function:** `AArch64FrameLowering::emitZeroCallUsedRegs`

### What this does

A **security feature**. Before a function returns, zeroes out all registers
that were used during the function to prevent sensitive data leakage.

### How it works

```
For GPRs (x0-x17):
  mov x0, xzr   (or equivalent zero instruction)

For FPRs/NEON (v0-v31):
  movi v0.16b, #0

For SVE predicates (p0-p15):
  pfalse p0.b
```

Controlled by the `-fzero-call-used-regs` compiler flag.

---

## Block 16 — `windowsRequiresStackProbe` (Lines 878–886)

**Lines:** `878 – 886`  
**Function:** `AArch64FrameLowering::windowsRequiresStackProbe`

### What stack probing is

On Windows, the OS only maps one page of stack at a time. If a function
allocates more than one page (4096 bytes) at once, it must **touch each page**
to trigger the OS to map it. This is called **stack probing**.

```asm
; Without probing (dangerous on Windows):
sub sp, sp, #8192   ; jumps over unmapped pages -> crash

; With probing:
sub sp, sp, #4096
str xzr, [sp]       ; touch page 1
sub sp, sp, #4096
str xzr, [sp]       ; touch page 2
```

Returns `true` if the stack is large enough to require probing on Windows.

---

## Block 17 — `findScratchNonCalleeSaveRegister` (Lines 888–930)

**Lines:** `888 – 930`  
**Function:** `AArch64FrameLowering::findScratchNonCalleeSaveRegister`

### What a scratch register is

Some prologue/epilogue operations need a **temporary register** to hold
intermediate values. For example, when the stack frame is too large to encode
in a single immediate:

```asm
mov x9, #large_value
sub sp, sp, x9
```

`x9` here is the scratch register.

### How it finds one

1. If this is the entry block (and not `preserve_none` calling convention), prefer `x9`
2. Otherwise, scan live registers to find one not currently in use
3. Prefer `x9` if available (historically used for this purpose)
4. Fall back to any available `GPR64`
5. Return `AArch64::NoRegister` if none found

---

## Block 18 — `canUseAsPrologue` (Lines 932–971)

**Lines:** `932 – 971`  
**Function:** `AArch64FrameLowering::canUseAsPrologue`

### What this does

Checks whether a given `MachineBasicBlock` can be used as the function prologue.
Some blocks cannot be used as prologues because:

- Swift async context setup clobbers `x16`/`x17` which may be live-in
- Stack probing sequences clobber `NZCV` flags which may be live-in
- Stack realignment or inline stack probing requires a scratch register that is not available
- Saving VG or Windows stack probing requires a scratch register with call support

---

## Block 19 — `needsWinCFI` and `shouldSignReturnAddressEverywhere` (Lines 973–990)

**Lines:** `973 – 990`

### `needsWinCFI` (Lines 973–977)

Returns `true` if the function needs Windows CFI (Structured Exception Handling
unwind opcodes). Required when targeting Windows and the function needs an
unwind table entry.

### `shouldSignReturnAddressEverywhere` (Lines 979–990)

Returns `true` if return address signing (PAC/PAuth) should be applied to
every basic block, not just the entry. Used with the `sign-return-address=all`
attribute. Not supported with WinCFI (ordering constraints).

---

## Block 20 — `insertSEH` (Lines 992–1167)

**Lines:** `992 – 1167`  
**Function:** `AArch64FrameLowering::insertSEH`

### What SEH is

SEH = **Structured Exception Handling** — the Windows exception handling mechanism.

On Windows, every function that modifies the stack must emit **unwind opcodes**
that describe how to reverse the prologue. These are used by the OS to unwind
the stack during exception handling.

### What this function does

For each spill instruction in the prologue, inserts a corresponding **SEH pseudo
instruction** that describes the spill to the Windows unwind system.

### Examples

| Spill instruction | SEH opcode emitted |
|-------------------|--------------------|
| `stp x19, x20, [sp, #-16]!` | `SEH_SaveRegP_X x19, x20, -16` |
| `stp fp, lr, [sp, #-16]!` | `SEH_SaveFPLR_X -16` |
| `str z8, [sp, #offset, mul vl]` | `SEH_SaveZReg z8, offset` |
| `str p4, [sp, #offset, mul vl]` | `SEH_SavePReg p4, offset` |
| `stp q8, q9, [sp, #offset]` | `SEH_SaveAnyRegQP reg0, reg1, offset` |

---

## Block 21 — `requiresSaveVG` (Lines 1169–1179)

**Lines:** `1169 – 1179`  
**Function:** `AArch64FrameLowering::requiresSaveVG`

### What VG is

`VG` is the **SVE vector length register** — a virtual register that holds the
number of 64-bit elements in an SVE vector. It is not a real hardware register
but is treated as one for spilling purposes.

### When VG must be saved

When a function has **streaming mode changes** (SME), the value of VG may differ
between streaming and non-streaming mode. If DWARF unwind info is needed, VG
must be saved so the unwinder can compute the correct CFA.

On Darwin, VG is only saved if SVE is actually available.

---

## Block 22 — `emitPacRetPlusLeafHardening` (Lines 1181–1212)

**Lines:** `1181 – 1212`  
**Function:** `AArch64FrameLowering::emitPacRetPlusLeafHardening`

### What PAC/PAuth is

PAC = **Pointer Authentication Code**. AArch64 supports signing the return
address with a cryptographic code to prevent return-oriented programming (ROP)
attacks.

### What this function does

Emits `PAUTH_PROLOGUE` (sign return address) at function entry and EH funclet
entries, and `PAUTH_EPILOGUE` (authenticate return address) before each return.

This is the "pac-ret plus leaf hardening" variant which applies signing even
to leaf functions.

---

## Block 23 — `emitPrologue` and `emitEpilogue` (Lines 1214–1229)

**Lines:** `1214 – 1229`  
**Functions:** `AArch64FrameLowering::emitPrologue`, `AArch64FrameLowering::emitEpilogue`

### What these do

These are the **entry points** for generating the function prologue and epilogue.
The actual work is delegated to `AArch64PrologueEmitter` and
`AArch64EpilogueEmitter` in `AArch64PrologueEpilogue.cpp`.

### What a prologue does

```
1. Save callee-saved registers (stp instructions)
2. Adjust sp downward to allocate the frame
3. Set up fp (if needed)
4. Set up bp (if needed)
5. Emit CFI directives for unwinding
6. Sign return address (if pointer authentication is enabled)
7. Save VG (if streaming mode changes exist)
```

### What an epilogue does

```
1. Restore sp to pre-frame value
2. Restore callee-saved registers (ldp instructions)
3. Authenticate return address (if pointer authentication is enabled)
4. Return (ret)
```

### Visual: full prologue example

```asm
; Function entry: sp = 0x10030

stp x28, x27, [sp, #-48]!   ; save x28/x27, sp = 0x10000
stp fp,  lr,  [sp, #16]     ; save fp/lr at sp+16
add fp,  sp,  #16           ; fp = sp + 16 = 0x10010
sub sp,  sp,  #1360         ; allocate locals

; After prologue:
; sp = 0x0FAB0  (local frame)
; fp = 0x10010  (frame record)
```

---

## Block 24 — CFI Fixup Helpers (Lines 1226–1235)

**Lines:** `1226 – 1235`  
**Functions:** `enableCFIFixup`, `enableFullCFIFixup`

### What CFI fixup is

After the prologue/epilogue inserter runs, some CFI directives may be incorrect
or missing (e.g. in blocks that are reached via exception handling paths).
The CFI fixup pass corrects these.

- `enableCFIFixup` — enables basic CFI fixup if DWARF unwind info is needed
- `enableFullCFIFixup` — enables full async DWARF fixup for async unwind info

---

## Block 25 — Frame Index Reference Functions (Lines 1237–1359)

**Lines:** `1237 – 1359`  
**Functions:** `getFrameIndexReference`, `getFrameIndexReferenceFromSP`,
`getNonLocalFrameIndexReference`, `getFPOffset`, `getStackOffset`,
`getSEHFrameIndexOffset`, `resolveFrameIndexReference`

### What a frame index is

During compilation, before register allocation, stack objects are referred to
by **frame indices** (integers like `FI = 0`, `FI = 1`) rather than actual
byte offsets.

```
%0 = alloca i32    -> frame index 0
%1 = alloca i64    -> frame index 1
```

### What these functions do

Given a frame index, they return:
- `FrameReg` — which register to use as base (`sp`, `fp`, or `bp`)
- Return value — byte offset from that register

| Function | Purpose |
|----------|---------|
| `getFrameIndexReference` | General-purpose FI resolution (used by debug info) |
| `getFrameIndexReferenceFromSP` | SP-relative offset for stack layout analysis |
| `getNonLocalFrameIndexReference` | For funclets accessing parent frame (Windows) |
| `getFPOffset` | Compute offset relative to frame pointer |
| `getStackOffset` | Compute offset relative to stack pointer |
| `getSEHFrameIndexOffset` | Compute offset for Windows SEH unwind data |
| `resolveFrameIndexReference` | Delegates to `resolveFrameOffsetReference` |

---

## Block 26 — `resolveFrameOffsetReference` (Lines 1361–1536)

**Lines:** `1361 – 1536`  
**Function:** `AArch64FrameLowering::resolveFrameOffsetReference`

### What this does

The core function for deciding **which base register to use** and **what offset**
when accessing any stack object.

### Decision logic

```
Fixed objects (function arguments):
  Use fp if available.

CSR objects (callee-saved register spills):
  Use fp if stack is being realigned.

Local variables:
  Use fp if:
    - FP offset is positive (always better than SP)
    - VLAs exist and no base pointer is available
    - FP offset fits and is preferred
  Use bp if:
    - VLAs exist and base pointer is available
  Use sp otherwise.

SVE objects:
  Use fp if available and beneficial (scalable offsets are cheaper from fp).
  Use sp or bp otherwise.
```

### Why this matters

AArch64 load/store instructions have limited immediate offset ranges:

```asm
ldr x0, [sp, #offset]   ; offset must fit in 12 bits (0..4095 scaled)
ldur x0, [sp, #offset]  ; offset must fit in 9 bits signed (-256..255)
```

If the offset is too large, a scratch register is needed. This function picks
the base register that gives the smallest offset.

---

## Block 27 — Register Pairing Helpers (Lines 1538–1667)

**Lines:** `1538 – 1667`  
**Functions:** `getPrologueDeath`, `produceCompactUnwindFrame`,
`invalidateWindowsRegisterPairing`, `invalidateRegisterPairing`,
`struct RegPairInfo`, `findFreePredicateReg`, `enableMultiVectorSpillFill`

### `getPrologueDeath` (Lines 1538–1546)

Returns the kill flag state for a register being spilled in the prologue.
Registers that are also live-in (e.g. arguments passed in callee-saved registers)
must **not** have the kill flag set.

### `produceCompactUnwindFrame` (Lines 1548–1558)

Returns `true` if the function should use MachO compact unwind format.
Requires: MachO target, no SwiftError, no SwiftTail calling convention,
no VG save, no SVE calling convention.

### `invalidateRegisterPairing` (Lines 1603–1621)

Returns `true` if two registers **cannot** be paired into a single `stp`/`ldp`.

Reasons for invalidation:
- Windows CFI requires consecutive registers only
- LR must be paired with FP (not any other register) when a frame record is needed
- ARM64EC alignment requirements

### `struct RegPairInfo` (Lines 1624–1639)

Describes one register (or pair of registers) to be spilled/restored.

```cpp
struct RegPairInfo {
  Register Reg1;       // first register
  Register Reg2;       // second register (invalid if unpaired)
  int FrameIdx;        // stack slot index
  int Offset;          // offset from sp/fp in units of register size
  enum RegType {
    GPR,     // x19-x28, fp, lr
    FPR64,   // d8-d15
    FPR128,  // q8-q15
    PPR,     // p4-p15 (SVE predicates)
    ZPR,     // z8-z23 (SVE vectors)
    VG       // virtual vector length register
  } Type;
};
```

### `findFreePredicateReg` (Lines 1641–1650)

Finds a free predicate register (`p8`–`p15`) to use as the governing predicate
for multi-vector SVE spill/fill instructions (`ST1B_2Z_IMM`, `LD1B_2Z_IMM`).

### `enableMultiVectorSpillFill` (Lines 1652–1667)

Returns `true` if multi-vector spill/fill instructions can be used.
Requires SVE2.1 or SME2 (and not locally-streaming mode for SME2).

---

## Block 28 — `computeCalleeSaveRegisterPairs` (Lines 1669–1954)

**Lines:** `1669 – 1954`  
**Function:** `computeCalleeSaveRegisterPairs`

### What callee-saved registers are

The AArch64 ABI defines which registers a function must preserve:

| Category | Registers | Who saves them |
|----------|-----------|----------------|
| Caller-saved (volatile) | x0–x17, v0–v7, v16–v31 | Caller saves before call |
| Callee-saved (non-volatile) | x19–x28, x29(fp), x30(lr), v8–v15 | Callee saves on entry |

### Why pairs?

AArch64 has `stp`/`ldp` instructions that store/load **two registers at once**:

```asm
stp x19, x20, [sp, #-16]!   ; save x19 and x20 together, decrement sp
ldp x19, x20, [sp], #16     ; restore x19 and x20 together, increment sp
```

This is more efficient than two separate stores/loads.

### What this function does

Iterates over the callee-saved register list and groups them into `RegPairInfo`
structures. Handles:

- GPR pairs (`stp xi, xj`)
- FPR64 pairs (`stp di, dj`)
- FPR128 pairs (`stp qi, qj`)
- ZPR pairs (`st1b {zi, zj}`) — only with SVE2.1/SME2
- PPR singles (`str pi`) — predicates cannot be paired
- VG singles — virtual register, needs special handling
- Stack hazard padding insertion between GPR and FPR CSRs
- Alignment gaps to ensure 16-byte alignment
- Windows vs non-Windows fill direction (top-down vs bottom-up)
- Swift async context extra 8-byte slot

---

## Block 29 — `spillCalleeSavedRegisters` (Lines 1956–2173)

**Lines:** `1956 – 2173`  
**Function:** `AArch64FrameLowering::spillCalleeSavedRegisters`

### What this does

Emits the actual **store instructions** in the prologue to save callee-saved
registers to the stack.

### Example output

```asm
stp x22, x21, [sp, #0]
stp x20, x19, [sp, #16]
stp fp,  lr,  [sp, #32]
```

### Instruction selection per register type

| Register type | Paired instruction | Single instruction |
|---------------|-------------------|--------------------|
| GPR | `STPXi` | `STRXui` |
| FPR64 | `STPDi` | `STRDui` |
| FPR128 | `STPQi` | `STRQui` |
| ZPR (SVE2.1/SME2) | `ST1B_2Z_IMM` | `STR_ZXI` |
| PPR | — | `STR_PXI` |
| VG | — | `STRXui` (after reading VG value) |

### VG special case

VG is not a real hardware register. To save it:
1. If SVE is available: use `CNTD` instruction to read the vector length
2. If SVE is not available: call the `__arm_get_current_vg` runtime function
3. Store the result to the VG stack slot

### Homogeneous prolog shortcut

If `homogeneousPrologEpilog` returns true, emits a single `HOM_Prolog`
pseudo instruction instead of individual spills.

---

## Block 30 — `restoreCalleeSavedRegisters` (Lines 2175–2318)

**Lines:** `2175 – 2318`  
**Function:** `AArch64FrameLowering::restoreCalleeSavedRegisters`

### What this does

Mirror of `spillCalleeSavedRegisters`. Emits **load instructions** in the
epilogue to restore callee-saved registers from the stack.

### Example output

```asm
ldp fp,  lr,  [sp, #32]
ldp x20, x19, [sp, #16]
ldp x22, x21, [sp, #0]
```

Note the **reverse order** from spilling — restores happen bottom-up.

### SVE restore order

For performance reasons, SVE registers are restored in **increasing** order
(reversed from the spill order). This is done by reversing the PPR and ZPR
sub-ranges of the `RegPairs` list before iterating.

### VG special case

VG is never restored — it is a virtual register that reflects the current
hardware state. The `continue` statement skips VG entries during restore.

---

## Block 31 — Frame ID Helpers (Lines 2320–2354)

**Lines:** `2320 – 2354`  
**Functions:** `getMMOFrameID`, `getLdStFrameID`, `isPPRAccess`

### `getMMOFrameID`

Given a `MachineMemOperand` (a memory access annotation on a machine instruction),
returns the frame index of the stack object being accessed, if any.

Handles two cases:
- Fixed stack pseudo source values (spill slots)
- `alloca` instructions mapped to frame objects

### `getLdStFrameID`

Wrapper around `getMMOFrameID` for load/store instructions.
Returns the frame index of the first memory operand.

### `isPPRAccess`

Returns `true` if a load/store instruction accesses a PPR (SVE predicate)
register. Used to classify stack slots as PPR-only vs ZPR/FPR.

---

## Block 32 — `determineStackHazardSlot` (Lines 2356–2500)

**Lines:** `2356 – 2500`  
**Function:** `AArch64FrameLowering::determineStackHazardSlot`

### What a stack hazard is

Under SME (Scalable Matrix Extension), the CPU has a separate coprocessor for
FP/vector operations. If the CPU and the SME coprocessor both access the **same
memory region** (including the stack), it can cause a **cache hazard** —
incorrect or slow behavior.

### The solution: hazard padding

Insert a **padding slot** between GPR and FPR stack regions:

```
[GPR callee saves]
[hazard padding]    <- empty space, acts as a buffer
[FPR callee saves]
```

### What this function does

1. Check if hazard padding is needed (streaming functions only, unless `StackHazardInNonStreaming` is set)
2. Scan saved registers to see if any FPR/ZPR/PPR registers are saved
3. Scan stack objects to classify them as GPR-only, FPR-only, or PPR-only
4. If FPR/ZPR objects exist, create a stack object of size `StackHazardSize` (must be multiple of 16)
5. Store its frame index in `AArch64FunctionInfo` for later use
6. Optionally enable `SplitSVEObjects` to separate PPR and ZPR regions

### `SplitSVEObjects` decision

When `SplitSVEObjects` is enabled, the hazard padding is placed **between the
PPR area and the ZPR/FPR area** instead of between GPR and FPR CSRs. This
avoids hazards between both GPR↔FPR and ZPR↔PPR.

FPR callee-saves are promoted to ZPR callee-saves when `SplitSVEObjects` is
active, to ensure they land in the ZPR region.

---

## Block 33 — `determineCalleeSaves` (Lines 2502–2896)

**Lines:** `2502 – 2896`  
**Function:** `AArch64FrameLowering::determineCalleeSaves`

### What this does

Called **before** the prologue is emitted. Decides which registers need to be
saved and computes the total callee-save stack size.

### Steps

```
1.  Call base class to get initial set of used callee-saved registers
2.  Handle base pointer register (x19 if needed)
3.  Skip user-reserved registers (set via +reserve-x#i)
4.  Handle register pairing (force pairs for compact unwind)
5.  Handle ZPR pairs (find a predicate register for multi-vector spill)
6.  Handle Windows x18 preservation (for Win64 CC on non-Windows OS)
7.  Determine hazard slot placement (calls determineStackHazardSlot)
8.  Calculate total callee-save stack size (GPR, ZPR, PPR separately)
9.  Add VG save if streaming mode changes exist
10. Force LR save if VG save requires a runtime call (no SVE)
11. Force FP and LR to be saved if a frame pointer is needed
12. Estimate if the frame can be eliminated entirely
13. Determine if a base pointer is needed
14. Reserve emergency spill slot if needed
```

### Key outputs

After this function runs, `AArch64FunctionInfo` contains:
- `CalleeSavedStackSize` — total bytes for GPR/FPR callee saves
- `ZPRCalleeSavedStackSize` — bytes for ZPR callee saves
- `PPRCalleeSavedStackSize` — bytes for PPR callee saves
- `StackHazardSlotIndex` — frame index of hazard padding slot (if any)
- `PredicateRegForFillSpill` — predicate register for multi-vector spill

---

## Block 34 — `determineSVEStackSizes` (Lines 2898–3511)

**Lines:** `2898 – 3511`  
**Function:** `static SVEStackSizes determineSVEStackSizes`

### What this does

Processes all SVE stack objects (scalable-size local variables and spill slots)
and computes:
- Total ZPR (vector) stack size in `vscale` units
- Total PPR (predicate) stack size in `vscale` units

When `AssignOffsets::Yes` is passed, also assigns the actual offsets to each
SVE stack object in `MachineFrameInfo`.

### Why SVE sizes are special

SVE object sizes are **not known at compile time**. They are expressed as
multiples of `vscale * 16` bytes (for vectors) or `vscale * 2` bytes (for
predicates). The actual size is only known at runtime.

This means SVE stack objects cannot be mixed with fixed-size objects in the
same offset calculation — they need separate tracking.

### Split SVE objects layout

When `SplitSVEObjects` is enabled:

```
[PPR callee saves]
[PPR stack objects]
[hazard padding]
[ZPR/FPR callee saves]
[ZPR stack objects]
```

When disabled (default):

```
[hazard padding]
[ZPR + FPR + PPR callee saves]
[ZPR + PPR stack objects]
```

---

## Block 35 — `getFrameIndexReferencePreferSP` (Lines 3513–3645)

**Lines:** `3513 – 3645`  
**Function:** `AArch64FrameLowering::getFrameIndexReferencePreferSP`

### What this does

A variant of `getFrameIndexReference` that **prefers `sp`** as the base register
when possible, even if `fp` is available.

Used by the stack frame layout analysis pass and by some debug info generation
paths where SP-relative offsets are more useful for analysis.

### Difference from `getFrameIndexReference`

`getFrameIndexReference` may prefer `fp` for certain objects (arguments, CSRs).
`getFrameIndexReferencePreferSP` always tries `sp` first and only falls back to
`fp` when `sp` cannot reach the object (e.g. VLA area, SVE objects with
realignment).

---

## Full Conceptual Map

```
AArch64FrameLowering.cpp
│
├── Stack Frame Layout (Lines 1–303)
│     Stack grows down. Regions: args, callee saves, frame record,
│     SVE objects, locals, VLAs. Three pointers: sp, fp, bp.
│
├── Command-line options (Lines 170–303)
│     Control red zone, object ordering, SVE splitting, hazard padding.
│
├── SVE stack size helpers (Lines 344–397)
│     ZPR/PPR sizes in scalable units. Hazard size from subtarget.
│
├── Homogeneous prolog/epilog (Lines 399–455)
│     Code-size optimization: single helper call instead of individual spills.
│
├── Stack size limit estimation (Lines 457–491)
│     DefaultSafeSPDisplacement = 255. Scan instructions for legal offsets.
│
├── Fixed object size (Lines 493–531)
│     Tail call space, Windows varargs, EH catch objects.
│
├── Red zone (Lines 533–561)
│     128-byte area below sp. Leaf functions only. Many exclusions.
│
├── Frame pointer decision (Lines 563–617)
│     When is x29 needed? VLAs, realignment, EH, large frames, SME.
│
├── FP reserved check (Lines 619–642)
│     Darwin and Windows always reserve FP for frame chain validity.
│
├── Reserved call frame (Lines 644–652)
│     Pre-allocate outgoing argument space vs. adjust sp per call.
│
├── Call frame pseudo elimination (Lines 654–715)
│     Replace ADJCALLSTACKDOWN/UP with real sub/add sp instructions.
│
├── CFI reset (Lines 717–748)
│     Reset DWARF unwind state after EH handler returns.
│
├── Zero call-used registers (Lines 827–876)
│     Security: zero registers before return to prevent data leakage.
│
├── Windows stack probing (Lines 878–886)
│     Touch each page when allocating large stacks on Windows.
│
├── Scratch register finder (Lines 888–930)
│     Find a free non-callee-save register for prologue use.
│
├── Windows SEH (Lines 992–1167)
│     Emit Windows unwind opcodes alongside spill instructions.
│
├── PAC/PAuth hardening (Lines 1181–1212)
│     Sign/authenticate return address for ROP protection.
│
├── Prologue/epilogue entry points (Lines 1214–1229)
│     Delegate to AArch64PrologueEpilogue.cpp.
│
├── Frame index resolution (Lines 1237–1536)
│     For each stack object: which base register? What offset?
│     Core logic in resolveFrameOffsetReference (Lines 1361–1536).
│
├── Register pairing helpers (Lines 1538–1667)
│     RegPairInfo struct. Pairing validation. Multi-vector spill check.
│
├── computeCalleeSaveRegisterPairs (Lines 1669–1954)
│     Group callee-saved registers into stp/ldp pairs.
│     Handle hazard padding, alignment gaps, Windows vs non-Windows.
│
├── spillCalleeSavedRegisters (Lines 1956–2173)
│     Emit actual stp/str instructions for callee saves in prologue.
│
├── restoreCalleeSavedRegisters (Lines 2175–2318)
│     Emit actual ldp/ldr instructions for callee restores in epilogue.
│
├── Stack hazard slot (Lines 2356–2500)
│     SME cache hazard padding between GPR and FPR stack regions.
│     SplitSVEObjects decision for PPR/ZPR separation.
│
├── determineCalleeSaves (Lines 2502–2896)
│     Decide which registers to save. Compute all stack sizes.
│     Reserve emergency spill slot. Handle base pointer.
│
├── determineSVEStackSizes (Lines 2898–3511)
│     Assign offsets to scalable SVE stack objects.
│     Handle split vs non-split SVE layout.
│
└── getFrameIndexReferencePreferSP (Lines 3513–3645)
      SP-preferred variant of frame index resolution.
      Used by stack layout analysis and some debug info paths.
```

---

## Key Concepts Quick Reference

### Stack pointers

| Register | Name | Purpose |
|----------|------|---------|
| `sp` (x31) | Stack pointer | Current top of stack |
| `fp` (x29) | Frame pointer | Points to frame record |
| `lr` (x30) | Link register | Return address |
| `bp` (x19) | Base pointer | Stable base when VLAs exist |

### Register categories

| Category | Registers | Saved by |
|----------|-----------|----------|
| Caller-saved | x0–x17, v0–v7, v16–v31 | Caller |
| Callee-saved GPR | x19–x28, fp, lr | Callee |
| Callee-saved FPR | d8–d15 (or q8–q15) | Callee |
| SVE vector | z8–z23 | Callee |
| SVE predicate | p4–p15 | Callee |

### Important data structures

| Type | Purpose |
|------|---------|
| `MachineFrameInfo` | Stores all stack object metadata |
| `AArch64FunctionInfo` | AArch64-specific per-function state |
| `RegPairInfo` | Describes one register pair for spill/restore |
| `StackOffset` | Offset with fixed + scalable components |
| `CalleeSavedInfo` | One callee-saved register and its frame slot |

### Key alignment rules

| Rule | Value |
|------|-------|
| Default stack alignment | 16 bytes |
| Red zone size | 128 bytes |
| Max safe SP displacement | 255 bytes |
| Hazard slot alignment | 16 bytes (must be multiple of 16) |
| Tail call reserved stack | Must be multiple of 16 bytes |
