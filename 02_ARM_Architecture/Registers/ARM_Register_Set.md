# ARM Register Set

#beginner #intermediate #arm #registers

> Registers are the CPU's working memory — ultra-fast storage inside the chip itself. Mastering the ARM register set is the foundation of assembly programming and interrupt debugging.

---

## 📖 Concept Explanation

The ARM Cortex-M CPU has **16 registers**, each 32 bits wide. These are the only storage the CPU can directly operate on. All computations happen in registers. Memory is just a warehouse — registers are the workbench.

---

## 🔬 Deep Dive

### The ARM Cortex-M Register File

```
┌────────────────────────────────────────────┐
│          ARM Cortex-M Register File        │
├──────┬─────────────┬───────────────────────┤
│ Name │  Alias      │  Purpose              │
├──────┼─────────────┼───────────────────────┤
│  R0  │  a1         │  General purpose /    │
│  R1  │  a2         │  function arguments / │
│  R2  │  a3         │  return values        │
│  R3  │  a4         │  (caller-saved)       │
├──────┼─────────────┼───────────────────────┤
│  R4  │  v1         │  General purpose      │
│  R5  │  v2         │  (callee-saved)       │
│  R6  │  v3         │  Preserved across     │
│  R7  │  v4         │  function calls       │
│  R8  │  v5         │                       │
│  R9  │  v6/SB      │  Platform register    │
│  R10 │  v7/SL      │  Stack limit (opt)    │
│  R11 │  v8/FP      │  Frame pointer (opt)  │
│  R12 │  IP         │  Intra-procedure call │
├──────┼─────────────┼───────────────────────┤
│  R13 │  SP         │  Stack Pointer        │
│  R14 │  LR         │  Link Register        │
│  R15 │  PC         │  Program Counter      │
├──────┼─────────────┼───────────────────────┤
│  xPSR│  (special)  │  Status Register      │
└──────┴─────────────┴───────────────────────┘
```

### The Special-Purpose Registers in Detail

#### R13 — Stack Pointer (SP)
- Points to the top of the current stack
- ARM Cortex-M uses a **full descending stack** (SP points to last used word, grows toward lower addresses)
- Two physical SP registers: **MSP** (Main Stack Pointer) and **PSP** (Process Stack Pointer)
- MSP = used by exception handlers and privileged code
- PSP = used by RTOS tasks (unprivileged code)

```
High Address  0x20010000  ┐
                          │  ← MSP initial value (top of SRAM)
              0x2000FFE0  │  ← After 8 bytes pushed: SP = 0x2000FFF8
              0x2000FFE4  │
              0x2000FFE8  │
              0x2000FFEC  │
              0x2000FFF0  │
              0x2000FFF4  │
              0x2000FFF8  ├─ SP (after PUSH {R4-R7, LR})
Low Address   0x2000FFFC  └
```

#### R14 — Link Register (LR)
- Stores the **return address** when a function is called (BL instruction)
- When you call a function: `LR = address of instruction after the BL`
- When you return: `MOV PC, LR` or `POP {PC}`
- In exception entry: LR gets a special **EXC_RETURN** value

```asm
; Function call example
0x08000100: BL  my_function     ; LR = 0x08000104, PC = address of my_function
0x08000104: MOV R0, #1          ; ← This is where LR points (return address)

; Inside my_function:
my_function:
    PUSH {R4, LR}               ; Save LR (we'll call another function)
    ; ... do work ...
    POP  {R4, PC}               ; Restore, write old LR into PC = return
```

#### R15 — Program Counter (PC)
- Always holds the address of the **next instruction to fetch**
- **IMPORTANT:** On ARM, reads of PC return `current_address + 4` (pipeline effect)
- Writing to PC causes a branch (jump)
- Bit 0 must be 0 (word-aligned for 32-bit ARM) or 1 (Thumb mode switch)

#### xPSR — Program Status Register
The xPSR is a 32-bit register that packs three separate registers:

```
Bit:  31 30 29 28 27  26:25  24  23:20  19:16  15:10   9    8    7    6    5    4:0
      N  Z  C  V  Q  IT[1:0] T   [res] GE[3:0] IT[7:2] [res] [res] [res] [res] [res] ISR_NUMBER

      ↑─────────── APSR ───────────↑
                                  ↑T─ EPSR ──↑
                                                              ↑──────── IPSR ───────────↑
```

**N (Negative):** Set if result is negative (bit 31 = 1)
**Z (Zero):** Set if result is zero
**C (Carry):** Set if operation produced a carry out
**V (Overflow):** Set if signed overflow occurred
**T (Thumb bit):** Must always be 1 on Cortex-M (always in Thumb mode)
**ISR_NUMBER:** Current exception number (0 = thread mode, non-zero = handler mode)

---

## 💻 Code Example

### Viewing Registers with GDB

```bash
(gdb) info registers
r0             0x0                 0
r1             0x20000200          536871424
r2             0x0                 0
r3             0x0                 0
r4             0x0                 0
r5             0x0                 0
r6             0x0                 0
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x0                 0
r11            0x0                 0
r12            0x0                 0
sp             0x20010000          0x20010000
lr             0xffffffff          -1       ← Initial LR value
pc             0x08000200          0x8000200 <main>
xpsr           0x01000000          16777216  ← T bit set = Thumb mode
```

### Register Usage in Practice (C to Assembly)

```c
// C code
int add(int a, int b) {
    return a + b;
}

int main(void) {
    int result = add(3, 5);
}
```

Compiled assembly (ARM Cortex-M, GCC):
```asm
; add function:
; a is in R0 (first argument), b is in R1 (second argument)
add:
    ADD  R0, R0, R1     ; R0 = R0 + R1 (result goes in R0 = return value)
    BX   LR             ; Return (branch to address in LR)

; main:
main:
    PUSH {R7, LR}       ; Save LR (we're calling a function)
    MOV  R0, #3         ; First argument: a = 3
    MOV  R1, #5         ; Second argument: b = 5
    BL   add            ; Call add, LR = address of next instruction
    ; R0 now = 8 (return value)
    POP  {R7, PC}       ; Return from main
```

### Checking xPSR Flags

```c
// After this code, what flags are set?
int a = 5;
int b = 5;
int c = a - b;  // Result = 0

// Z flag = 1 (Zero), N flag = 0, C flag = 1 (borrow = no borrow), V flag = 0
```

```asm
MOV  R0, #5
MOV  R1, #5
SUBS R2, R0, R1    ; R2 = 0, sets flags: Z=1, N=0, C=1, V=0
BEQ  equal         ; Branch taken because Z=1
```

---

## 🌍 Real-World Use Case

### Examining a Hard Fault

When your system crashes with a HardFault, the first thing you do is dump all registers:

```bash
(gdb) info registers
pc    0x08001234   ← Where the crash happened
lr    0x08000abc   ← What called the crashed function  
sp    0x2000ff00   ← Stack pointer (check if it overflowed!)
xpsr  0x61000003   ← ISR_NUMBER = 3 = HardFault handler is running
```

Reading these registers tells you exactly:
- What instruction crashed (PC)
- What function called it (LR → trace back)
- Whether the stack is valid (SP within SRAM bounds)

→ See [[../../11_Debugging_Tooling/Hard_Fault_Debug/Debugging_Hard_Faults|Debugging Hard Faults]]

---

## ⚠️ Common Pitfalls

- **PC reads return PC+4**: If you read PC in an instruction, the value is current instruction address + 4 (pipeline prefetch). Critical for position-independent code.
- **LR has EXC_RETURN in exceptions**: In an interrupt handler, LR does NOT contain a return address — it contains a special bit pattern (0xFFFFFFF9, 0xFFFFFFFD, etc.). Writing this to PC triggers the exception return mechanism.
- **Corrupting callee-saved registers**: If your function modifies R4–R11, you MUST push/pop them. Forgetting this causes incredibly hard-to-find bugs.
- **SP must be 8-byte aligned on function entry**: Required by AAPCS. Failure causes hard faults on some architectures.

---

## 🔗 Related Notes

- [[Program_Counter_LR_SP|PC, LR, and SP — Deep Dive]]
- [[CPSR_APSR_xPSR|Status Registers (xPSR)]]
- [[../Programmers_Model/Processor_Modes|Processor Operating Modes]]
- [[../../03_ARM_Assembly/Calling_Conventions/AAPCS_Convention|AAPCS Calling Convention]]
- [[../../05_Cortex_M_Deep_Dive/Exception_Model/Exception_Model_Overview|Exception Model]]
- [[../../11_Debugging_Tooling/Hard_Fault_Debug/Debugging_Hard_Faults|Debugging Hard Faults]]

---

## 🎯 Interview Questions

1. **How many registers does ARM Cortex-M have?**
   *16 general-purpose registers (R0–R15), plus the xPSR status register.*

2. **What is the Link Register used for?**
   *Stores the return address when a function is called via BL instruction. When the function returns, the PC is loaded from LR.*

3. **What are MSP and PSP?**
   *Main Stack Pointer (used by privileged/handler code and exceptions) and Process Stack Pointer (used by RTOS tasks in unprivileged mode). They are two physical banks of the R13 register.*

4. **If you corrupt R5 inside a function without saving it, what happens?**
   *R5 is callee-saved (AAPCS). The caller expects R5 to be unchanged after the call. Corrupting it causes silent data corruption — one of the hardest bugs to track down.*

5. **What does the T bit in xPSR do?**
   *Indicates Thumb mode. On Cortex-M it must always be 1 — the CPU only executes Thumb/Thumb-2 instructions. Clearing it causes a UsageFault.*

---

**Next →** [[Program_Counter_LR_SP|PC, LR, and SP — Deep Dive]]

*← [[../MOC_ARM_Architecture|Stage 2 ARM Architecture]]*
