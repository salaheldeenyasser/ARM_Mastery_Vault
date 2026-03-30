# What is a CPU?

#beginner #arm #foundations

> A CPU (Central Processing Unit) is the digital brain of a computer — it fetches instructions from memory, decodes them, and executes operations on data.

---

## 📖 Concept Explanation

Every microcontroller, microprocessor, and SoC has a CPU at its core. Understanding the CPU is the first step to understanding ARM.

A CPU does exactly three things, over and over:
1. **Fetch** — Read the next instruction from memory
2. **Decode** — Understand what the instruction wants
3. **Execute** — Do it (compute, move data, branch)

This is the **Fetch-Decode-Execute cycle** — the heartbeat of every processor. See → [[Fetch_Decode_Execute]]

---

## 🔬 Deep Dive

### Core CPU Components

```
┌─────────────────────────────────────────────┐
│                    CPU                      │
│                                             │
│  ┌──────────┐    ┌──────────┐               │
│  │  Control │    │   ALU    │               │
│  │   Unit   │    │(Arithmetic│              │
│  │  (CU)    │    │  Logic   │               │
│  └────┬─────┘    │  Unit)   │               │
│       │          └──────────┘               │
│  ┌────▼──────────────────────┐              │
│  │     Register File         │              │
│  │  R0, R1, R2 ... R15, PC   │              │
│  └───────────────────────────┘              │
└────────────────────┬────────────────────────┘
                     │ Bus Interface
              ┌──────▼──────┐
              │    Memory   │
              │  (RAM/Flash)│
              └─────────────┘
```

### Key Subsystems

**Control Unit (CU)**
- Reads (fetches) instructions from memory
- Decodes what each instruction means
- Generates control signals to drive the ALU and registers

**Arithmetic Logic Unit (ALU)**
- Performs all arithmetic: ADD, SUB, MUL
- Performs all logic: AND, OR, XOR, NOT
- Computes comparisons and updates status flags

**Register File**
- Small, ultra-fast on-chip storage
- Much faster than even L1 cache
- ARM has 16 general-purpose registers (R0–R15)
- See → [[../../../02_ARM_Architecture/Registers/ARM_Register_Set|ARM Register Set]]

**Program Counter (PC)**
- Holds the address of the NEXT instruction to fetch
- In ARM Cortex-M: this is register R15
- See → [[../../../02_ARM_Architecture/Registers/Program_Counter_LR_SP|PC, LR, and SP]]

**Bus Interface**
- Connects the CPU to external memory and peripherals
- ARM uses AHB (Advanced High-performance Bus) and APB
- Transfers instruction and data between CPU and memory

---

## 🖼️ Diagram: CPU in a Microcontroller

```
┌──────────────────────────────────────────┐
│            Microcontroller               │
│                                          │
│  ┌────────┐   ┌────────┐  ┌──────────┐   │
│  │  CPU   │   │ Flash  │  │   SRAM   │   │
│  │(Cortex │◄──│(Code)  │  │ (Data)   │   │
│  │  -M4)  │   │128 KB  │  │  32 KB   │   │
│  └───┬────┘   └────────┘  └──────────┘   │
│      │                                   │
│  ┌───▼────────────────────────────────┐  │
│  │         Bus Matrix (AHB)           │  │
│  └────┬───────────┬────────────┬──────┘  │
│  ┌────▼──┐  ┌─────▼──┐   ┌─────▼─────┐   │
│  │  GPIO │  │  UART  │   │  Timers   │   │
│  └───────┘  └────────┘   └───────────┘   │
└──────────────────────────────────────────┘
```

---

## 💻 Code Example

When you write this C code:

```c
int a = 5;
int b = 3;
int c = a + b;
```

The CPU actually does this at register level:

```asm
; ARM Cortex-M Assembly (simplified)
MOV  R0, #5      ; Load 5 into register R0 (a)
MOV  R1, #3      ; Load 3 into register R1 (b)
ADD  R2, R0, R1  ; R2 = R0 + R1 = 8 (c)
STR  R2, [SP]    ; Store result to memory (stack)
```

The CPU fetches each line from Flash, decodes it, and executes it using the ALU and registers.

---

## 🌍 Real-World Use Case

Every embedded system you encounter has a CPU:

| Product | CPU | Core |
|---------|-----|------|
| STM32F4 | ARM Cortex-M4 | 168 MHz |
| Raspberry Pi 4 | ARM Cortex-A72 | 1.8 GHz |
| iPhone 15 | Apple A17 (ARM) | ~3.7 GHz |
| Arduino Uno | AVR ATmega328P | 16 MHz |
| ESP32 | Xtensa LX6 | 240 MHz |

ARM dominates embedded and mobile because of its **power efficiency and licensing model** — chip makers license the ARM ISA and build their own silicon around it.

---

## ⚠️ Common Pitfalls

- **Confusing CPU and MCU**: A CPU is just the processing core. An MCU (microcontroller) is the CPU + RAM + Flash + peripherals — all on one chip.
- **Thinking registers are memory**: Registers are NOT memory addresses. They are on-chip storage inside the CPU with no addresses (you access them by name: R0, R1...).
- **Ignoring the pipeline**: Modern CPUs don't execute one instruction at a time — they overlap fetch/decode/execute in a pipeline. This matters for timing. See → [[../../../09_Performance_Optimization/Pipeline/ARM_Pipeline|ARM Pipeline]]

---

## 🔗 Related Notes

- [[Von_Neumann_vs_Harvard|Von Neumann vs Harvard Architecture]]
- [[Fetch_Decode_Execute|Fetch-Decode-Execute Cycle]]
- [[Registers_and_ALU|Registers & ALU]]
- [[../Embedded_Intro/What_Is_Embedded_Systems|What is Embedded Systems?]]
- [[../../02_ARM_Architecture/Registers/ARM_Register_Set|ARM Register Set]]

---

## 🎯 Interview Questions

1. **What are the three main components of a CPU?**
   *Control Unit, ALU, Register File — plus the bus interface.*

2. **What is the difference between a CPU and an MCU?**
   *CPU is the processing core only. MCU integrates CPU + Flash + RAM + peripherals on one die.*

3. **What does the Program Counter do?**
   *Holds the address of the next instruction to be fetched. On ARM Cortex-M it is R15.*

4. **Why are registers faster than RAM?**
   *Registers are on-chip, inside the CPU die itself, with direct wiring to the ALU — zero bus latency. RAM requires bus transactions.*

---

**Next →** [[Von_Neumann_vs_Harvard|Von Neumann vs Harvard Architecture]]

*← [[../MOC_Foundations|Stage 1 Foundations]]*
