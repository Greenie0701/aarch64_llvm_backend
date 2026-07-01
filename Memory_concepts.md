# ARM64 Memory System Learning Roadmap

A structured roadmap for learning the ARM64 memory system, including the MMU, caches, memory ordering, and AArch64 memory operations. This roadmap is intended for systems programmers and LLVM AArch64 backend developers.

---

## 1. Memory Hierarchy

Understand how data moves through the memory subsystem and the performance characteristics of each level.

### Topics

- Registers
- L1 Cache
- L2 Cache
- L3 Cache
- DRAM (Main Memory)

### Learn

- Memory latency
- Cache hierarchy
- Data movement between cache levels
- CPU vs Memory performance gap

---

## 2. Virtual Memory

Learn how ARM64 separates virtual and physical memory.

### Topics

- Virtual Addresses
- Physical Addresses
- Why Virtual Memory Exists

### Learn

- Process address spaces
- Address translation
- Memory protection
- Address space isolation

---

## 3. Memory Management Unit (MMU)

Study how the MMU translates virtual addresses into physical addresses.

### Topics

- Page Tables
- Address Translation
- Translation Lookaside Buffer (TLB)
- Page Faults

### Learn

- Multi-level page tables
- TTBR0 / TTBR1
- TCR_EL1
- MAIR_EL1
- Page table walk
- TLB maintenance

---

## 4. Cache Architecture

Understand how caches improve memory performance.

### Topics

- Cache Lines
- Set Associativity
- Write-back vs Write-through
- Cache Coherence

### Learn

- Cache maintenance
- Cache invalidation
- Cache cleaning
- Cache coherency protocols

---

## 5. Memory Attributes

Learn how ARM64 classifies different types of memory.

### Topics

- Normal Memory
- Device Memory
- Shareability
- Cacheability

### Learn

- Inner vs Outer Shareable
- Memory attribute encoding
- MAIR_EL1 configuration

---

## 6. Memory Ordering

Modern ARM processors can execute memory operations out of order. Learn how ordering is controlled.

### Topics

- Memory Reordering
- Data Memory Barrier (`DMB`)
- Data Synchronization Barrier (`DSB`)
- Instruction Synchronization Barrier (`ISB`)

### Learn

- Weak memory ordering
- Acquire/Release semantics
- Synchronization primitives

---

## 7. AArch64 Memory Instructions

Become familiar with the core memory-related instructions.

### Basic Load/Store

- `LDR`
- `STR`
- `LDP`
- `STP`

### Acquire / Release

- `LDAR`
- `STLR`

### Exclusive (Atomic)

- `LDXR`
- `STXR`

### Prefetch

- `PRFM`

### Learn

- Addressing modes
- Atomic operations
- Load/store pair instructions
- Prefetching

---

# Recommended Learning Order

1. Memory Hierarchy
2. Virtual Memory
3. MMU
4. Cache Architecture
5. Memory Attributes
6. Memory Ordering
7. AArch64 Memory Instructions

---

# Official References

## 1. Arm Learn the Architecture (Recommended Starting Point)

A beginner-friendly series of tutorials that introduce ARM architecture concepts before diving into the architecture specification.

https://www.arm.com/architecture/learn-the-architecture/a-profile

---

## 2. AArch64 Memory Management Guide

Official guide covering:

- Virtual memory
- MMU
- Page tables
- TLB
- Memory attributes
- Translation

https://documentation-service.arm.com/static/670e4dc89fbc7343d3e4cee1

---

## 3. Arm Architecture Reference Manual (ARM ARM)

The definitive reference for the ARMv8-A / AArch64 architecture.

Topics include:

- AArch64 instruction set
- Memory model
- MMU
- System registers
- Cache maintenance
- Memory barriers
- Exception levels

Available through Arm Developer documentation:

https://developer.arm.com/documentation

> Search for:
> **Arm® Architecture Reference Manual for A-profile architecture**

---
