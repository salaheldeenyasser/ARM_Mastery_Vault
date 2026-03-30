# ARM History and Evolution

#beginner #arm #history

> ARM didn't start as a world-dominating architecture. It started as a skunkworks project at Acorn Computers to build a cheap processor for a BBC computer — and accidentally created the most widely deployed CPU architecture in history.

---

## 📖 The ARM Story

### Timeline

```
1983 ──► Acorn Computers needs a 2nd-source processor for BBC Micro
         VLSI chips are too expensive. Decision: build their own.

1985 ──► ARM1 tape-out (Acorn RISC Machine)
         3 engineers, 12 months, 25,000 transistors
         First instruction executed: April 26, 1985
         (For comparison: Intel 80286 = 134,000 transistors)

1987 ──► ARM2 ships in Acorn Archimedes — first commercial ARM product
         8 MHz, 4 MIPS — outperforms Intel 286

1990 ──► Apple + Acorn + VLSI form ARM Ltd (Advanced RISC Machines)
         Apple Newton uses ARM610 — first mobile ARM device

1993 ──► ARM6 architecture — DEC StrongARM, NEC, Cirrus license it

1997 ──► ARM7TDMI — becomes the dominant embedded core
         Used in: Game Boy Advance, Nokia phones, countless MCUs

2004 ──► ARM Cortex introduced
         Cortex-M3: First "Cortex-M" core — embedded revolution begins

2007 ──► First iPhone — ARM11 core
         ARM's mobile dominance begins

2010 ──► Cortex-M4 — first M-profile with FPU and DSP extensions

2012 ──► ARMv8-A — first 64-bit ARM architecture (AArch64)
         
2016 ──► SoftBank acquires ARM for $32 billion

2021 ──► Cortex-M55, Cortex-M85 — ML at the edge
         ARMv9 architecture announced

2023 ──► ARM IPO on NASDAQ ($54 billion valuation)
         ARM chips in: 95%+ of smartphones, 90%+ of IoT devices
```

### The Licensing Model — Why ARM is Everywhere

ARM does NOT manufacture chips. ARM **licenses** its intellectual property.

```
┌──────────────┐
│   ARM Ltd    │  Designs the CPU architecture (ISA + microarchitecture)
│  (Designer)  │  Charges licensing fees
└──────┬───────┘
       │ License
       ▼
┌──────────────────────────────────────────────────────┐
│  Chip Manufacturers (Licensees)                       │
│  ST Microelectronics → STM32 (Cortex-M)              │
│  NXP → i.MX, LPC (Cortex-M, Cortex-A)               │
│  Texas Instruments → Tiva, OMAP                       │
│  Apple → A-series chips (custom ARM core)             │
│  Qualcomm → Snapdragon (custom ARM core)              │
│  Samsung → Exynos (custom ARM core)                   │
└──────────────────────────────────────────────────────┘
```

This is why you find ARM in everything from a $1 microcontroller to a $1000 iPhone. The same architecture, different implementations optimized for different markets.

---

## 🔬 ARM Architecture Generations

### Architecture → Core Mapping

| Architecture | Example Cores | Key Feature |
|---|---|---|
| ARMv4T | ARM7TDMI | Thumb instruction set |
| ARMv5TE | ARM9, ARM10 | Enhanced DSP |
| ARMv6 | ARM11 | SIMD, VFP, Thumb-2 preview |
| ARMv7-M | Cortex-M3 | Thumb-2 only, exception model |
| ARMv7-M | Cortex-M4 | + DSP + optional FPU |
| ARMv7-M | Cortex-M7 | + dual-issue pipeline, I/D cache |
| ARMv8-M | Cortex-M23/M33 | TrustZone for Cortex-M |
| ARMv8-M | Cortex-M55 | + Helium (M-Profile Vector) |
| ARMv7-A | Cortex-A8/9 | 32-bit application, NEON |
| ARMv8-A | Cortex-A53/57 | 64-bit (AArch64), first |
| ARMv9-A | Cortex-A510/710 | Scalable Vector Extension |

### The Cortex-M Family Specifically

This vault focuses heavily on Cortex-M because it dominates microcontrollers:

```
Cortex-M0/M0+   → Ultra-low power, simple (ARMv6-M)
                   Cost-optimized IoT, wearables
                   
Cortex-M3       → Mainstream (ARMv7-M)
                   First true Cortex-M for embedded
                   
Cortex-M4       → Performance (ARMv7-M + DSP + FPU)
                   Motor control, audio, sensor fusion
                   ← This is the sweet spot for learning
                   
Cortex-M7       → High performance (ARMv7-M, dual-issue)
                   High-speed networking, audio processing
                   
Cortex-M23/M33  → Security (ARMv8-M + TrustZone)
                   IoT security, secure boot
                   
Cortex-M55/M85  → ML at edge (ARMv8.1-M + Helium)
                   TinyML inference, DSP workloads
```

---

## 🌍 ARM in Numbers (2024)

- **280 billion** ARM chips shipped since 1990
- **~30 billion** ARM chips shipped per year
- **95%** of smartphones use ARM
- **67%** of IoT devices use ARM
- **100%** of iPhones/iPads use ARM (Apple Silicon)
- Present in: cars, toothbrushes, satellites, servers (AWS Graviton), supercomputers (Fugaku #1 HPC)

---

## ⚠️ Common Pitfalls

- **ARM ≠ ARM Ltd**: ARM the architecture is used by dozens of companies. When someone says "ARM chip," they mean a chip using the ARM ISA — not necessarily made by ARM Ltd (now ARM Holdings).
- **ARM7TDMI is NOT Cortex-M**: Many engineers confuse these. ARM7TDMI is ARMv4T (classic ARM, 3-stage pipeline). Cortex-M3 is ARMv7-M (completely different exception model, registers, pipeline). Code is NOT compatible at binary level.
- **AArch64 ≠ AArch32**: ARMv8 introduced 64-bit mode. The register file, instruction encoding, calling convention, and exception model all changed. Don't assume your Cortex-M knowledge directly maps to a Cortex-A running Linux in 64-bit mode.

---

## 🔗 Related Notes

- [[../ARM_Profiles/Cortex_M_Profile|Cortex-M Profile]]
- [[../ARM_Profiles/Cortex_A_Profile|Cortex-A Profile]]
- [[../RISC_Principles/RISC_Design_Philosophy|RISC Design Philosophy]]
- [[../../01_Foundations/Architecture_Overview/ARM_vs_x86_vs_MIPS|ARM vs x86 vs MIPS]]

---

## 🎯 Interview Questions

1. **Who designs ARM processors?**
   *ARM Ltd (now ARM Holdings, owned by SoftBank). They design the architecture and license it to chip manufacturers. ARM does not manufacture silicon.*

2. **Why is ARM so dominant in mobile/embedded?**
   *RISC design with excellent performance-per-watt ratio, flexible licensing model allowing many vendors to optimize for their market, and 40 years of ecosystem momentum.*

3. **What's the difference between ARMv7-M and ARMv8-M?**
   *ARMv8-M adds TrustZone security extension for Cortex-M, enabling secure/non-secure world separation. Also adds some instruction set improvements.*

4. **Is Cortex-M4 and ARM7TDMI the same architecture?**
   *No. ARM7TDMI is ARMv4T (classic ARM, 1997 era). Cortex-M4 is ARMv7-M (2010). Completely different exception models, pipeline structures, and Thumb-2 vs Thumb ISA.*

---

*← [[../MOC_ARM_Architecture|Stage 2 ARM Architecture]]*
