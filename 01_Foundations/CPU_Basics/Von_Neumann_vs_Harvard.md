# Von Neumann vs Harvard Architecture

#beginner #arm #foundations #architecture

> These two architectural models define **where code and data live** — a distinction that is fundamental to how every ARM chip is designed.

---

## 📖 Concept Explanation

When building a CPU, you must answer: *"Where do I store instructions and data, and can they share the same bus?"*

Two answers emerged:

| Architecture | Code & Data | Bus |
|---|---|---|
| **Von Neumann** | Same memory space | Shared bus |
| **Harvard** | Separate memories | Separate buses |

---

## 🔬 Deep Dive

### Von Neumann Architecture

In the classic Von Neumann model, instructions and data live in the **same memory** and travel on the **same bus**.

```
┌──────────┐    Single Bus    ┌────────────────┐
│   CPU    │◄────────────────►│  Unified Memory│
│          │                  │  (Code + Data) │
└──────────┘                  └────────────────┘
```

**Key Properties:**
- Simple design — one memory controller
- Code and data compete for the bus (**Von Neumann bottleneck**)
- Flexible — you can store code in any memory region
- Used by: x86 PCs, early ARM chips

**The Bottleneck Problem:**
```
Time →
[Fetch Instruction] [Fetch Data] [Fetch Instruction] [Fetch Data]
         ↑ Only one of these can happen at a time on one bus
```

### Harvard Architecture

Harvard uses **separate memories** for instructions and data, with **independent buses**.

```
              Instruction Bus
┌──────────┐◄──────────────────►┌──────────────┐
│   CPU    │                    │  Code Memory │
│          │                    │   (Flash)    │
│          │◄──────────────────►└──────────────┘
└──────────┘    Data Bus         ┌──────────────┐
                                 │  Data Memory │
                                 │    (SRAM)    │
                                 └──────────────┘
```

**Key Properties:**
- Can fetch instruction AND data **simultaneously**
- Higher throughput — enables pipelining
- Code memory (Flash) can be read-only, improving security
- Used by: ARM Cortex-M series, AVR, PIC

### Modified Harvard (Used by ARM Cortex-M)

Modern ARM microcontrollers use a **Modified Harvard** architecture — physically separate buses for code and data, but a **unified address space**.

```
ARM Cortex-M4 Internal Architecture:

  ┌─────────────────────────────────────────┐
  │              Cortex-M4 Core             │
  │  ┌─────────┐    ┌─────────┐            │
  │  │  I-Code │    │  D-Code │            │
  │  │   Bus   │    │   Bus   │            │
  │  │ (Instr) │    │ (Data)  │            │
  └──┴────┬────┴────┴────┬────┴────────────┘
          │              │
  ┌───────▼──────┐  ┌────▼─────────┐
  │ Code Flash   │  │  Data SRAM   │
  │ 0x08000000   │  │  0x20000000  │
  └──────────────┘  └──────────────┘
         Both regions visible in unified address space
```

**Why it matters:**
- The CPU can read two 32-bit values per cycle (one instruction, one data)
- This is what enables the Cortex-M4 to achieve ~1 DMIPS/MHz
- You can *also* execute code from SRAM (Modified Harvard — not possible in strict Harvard)

---

## 💻 Code Example

```c
// This is stored in Flash (Code Memory) - 0x08000000 on STM32
const uint32_t lookup_table[] = {0, 1, 4, 9, 16};  // .rodata -> Flash

// This is stored in SRAM (Data Memory) - 0x20000000 on STM32
uint32_t counter = 0;  // .data / .bss -> SRAM

int main(void) {
    // CPU fetches instructions from Flash via I-Code bus
    // CPU reads/writes counter via D-Code bus  
    // BOTH happen in the same clock cycle — Harvard advantage!
    counter = lookup_table[2];  // counter = 4
    return 0;
}
```

To verify memory layout after compilation:
```bash
arm-none-eabi-objdump -h your_firmware.elf
# Shows .text in Flash (0x08000000), .data in SRAM (0x20000000)
```

---

## 🌍 Real-World Use Case

**STM32F407 (Cortex-M4) Memory System:**

| Bus | Access | Speed |
|-----|--------|-------|
| I-Code | Flash instruction fetch | 0 wait states up to 24 MHz |
| D-Code | Flash data read (.rodata) | 0-5 wait states |
| System Bus | SRAM read/write | 0 wait states |

When you run at 168 MHz, the Flash has wait states but the CPU uses the **ART Accelerator** (prefetch + cache) to hide this latency. This is a direct consequence of the Modified Harvard design.

---

## ⚠️ Common Pitfalls

- **"ARM uses Von Neumann"** — False. Cortex-M uses Modified Harvard. The *address space* is unified, but the *buses* are physically separate. You'll fail interviews saying this.
- **Executing from SRAM**: Strict Harvard can't do this. Modified Harvard CAN — this is important for bootloaders and firmware updates.
- **Flash wait states**: Because code and constants share Flash on some chips, accessing large `const` tables while running fast code creates contention. Use SRAM for frequently accessed tables.

---

## 🔗 Related Notes

- [[What_Is_A_CPU|What is a CPU?]]
- [[Fetch_Decode_Execute|Fetch-Decode-Execute Cycle]]
- [[../../04_Memory_System/Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../07_ARM_C_Integration/Memory_Sections/Text_Data_BSS_Sections|.text, .data, .bss Sections]]

---

## 🎯 Interview Questions

1. **Does ARM Cortex-M use Von Neumann or Harvard?**
   *Modified Harvard — separate buses for instruction and data, but unified address space.*

2. **What is the Von Neumann bottleneck?**
   *When instruction fetch and data access compete for the same bus, creating a throughput limitation.*

3. **Why can't strict Harvard execute code from RAM?**
   *Because instruction and data memories are completely separate — the CPU cannot point its instruction fetch bus at data memory.*

4. **What advantage does Modified Harvard give the Cortex-M4?**
   *Simultaneous instruction fetch and data access in the same clock cycle, enabling high throughput without stalls.*

---

**Next →** [[Fetch_Decode_Execute|Fetch-Decode-Execute Cycle]]

*← [[../MOC_Foundations|Stage 1 Foundations]] | [[What_Is_A_CPU|← What is a CPU?]]*
