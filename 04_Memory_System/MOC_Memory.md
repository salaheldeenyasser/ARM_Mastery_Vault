# 🧠 Stage 4: Memory System & Organization — Map of Content

#intermediate #arm #memory #roadmap

> Memory is where everything lives. Stack, heap, code, constants, registers — understanding where each byte sits and how the CPU accesses it is the foundation of every debug session you'll ever have.

---

## Memory Map

- [[Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map (4GB Address Space)]]
- [[Memory_Map/STM32_Memory_Map|STM32F4 Concrete Memory Map]]
- [[Memory_Map/Memory_Regions_and_Attributes|Memory Region Types & Attributes]]

## Stack vs Heap

- [[Stack_vs_Heap/Stack_Memory_Model|Stack Memory Model]]
- [[Stack_vs_Heap/Heap_Memory_Model|Heap Memory Model]]
- [[Stack_vs_Heap/Stack_vs_Heap_Comparison|Stack vs Heap — When to Use Which]]
- [[Stack_vs_Heap/Stack_Overflow_Detection|Stack Overflow Detection]]

## Alignment

- [[Alignment/Memory_Alignment|Memory Alignment Rules]]
- [[Alignment/Alignment_Faults|Alignment Faults on Cortex-M]]
- [[Alignment/Struct_Padding|Structure Padding & Packing]]

## Endianness

- [[Endianness/Little_vs_Big_Endian|Little-Endian vs Big-Endian]]
- [[Endianness/Endianness_in_Practice|Endianness Bugs in Practice]]

## Cache Basics

- [[Cache_Basics/Cache_Fundamentals|Cache Fundamentals (L1/L2)]]
- [[Cache_Basics/Cache_Hit_Miss|Cache Hits, Misses & Write Policies]]
- [[Cache_Basics/Cache_on_Cortex_M7|Cache on Cortex-M7]]

## MMU vs MPU

- [[MMU_vs_MPU/MPU_on_Cortex_M|MPU on Cortex-M (Memory Protection Unit)]]
- [[MMU_vs_MPU/MMU_on_Cortex_A|MMU on Cortex-A (Memory Management Unit)]]
- [[MMU_vs_MPU/Virtual_vs_Physical_Memory|Virtual vs Physical Memory]]

---

**Previous Stage →** [[../03_ARM_Assembly/MOC_Assembly|Stage 3: ARM Assembly]]

**Next Stage →** [[../05_Cortex_M_Deep_Dive/MOC_CortexM|Stage 5: Cortex-M Deep Dive]]

*← [[../_Dashboard/🏠 ARM Mastery Vault Home|Home]]*
