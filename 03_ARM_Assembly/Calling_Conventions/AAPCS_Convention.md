# AAPCS — ARM Architecture Procedure Call Standard

#intermediate #arm #assembly #calling-convention

> AAPCS defines the rules for how ARM functions call each other. Violating it causes data corruption that's nearly impossible to find by inspection alone. Know it cold.

---

## 📖 Concept Explanation

When function A calls function B, they need to agree on:
- Which registers carry arguments?
- Where does the return value go?
- Which registers must B preserve for A?
- How is the stack structured?

AAPCS (ARM Architecture Procedure Call Standard) defines all of this. It's enforced by compilers and must be followed by any hand-written assembly that interfaces with C.

---

## 🔬 The AAPCS Rules

### Register Usage

```
Register  | AAPCS Role              | Who Must Save?
──────────┼─────────────────────────┼─────────────────────────
R0        | Arg 1 / Return value    | Caller-saved (corruptible)
R1        | Arg 2 / Return high 32b | Caller-saved (corruptible)
R2        | Arg 3                   | Caller-saved (corruptible)
R3        | Arg 4                   | Caller-saved (corruptible)
R4        | General purpose         | Callee-saved (must preserve)
R5        | General purpose         | Callee-saved (must preserve)
R6        | General purpose         | Callee-saved (must preserve)
R7        | General purpose         | Callee-saved (must preserve)
R8        | General purpose         | Callee-saved (must preserve)
R9        | Platform register       | Callee-saved (must preserve)
R10 (SL)  | Stack limit (optional)  | Callee-saved (must preserve)
R11 (FP)  | Frame pointer (optional)| Callee-saved (must preserve)
R12 (IP)  | Intra-procedure scratch | Caller-saved (corruptible)
R13 (SP)  | Stack Pointer           | Special — must be 8-byte aligned
R14 (LR)  | Link Register           | Caller-saved if you call others
R15 (PC)  | Program Counter         | Not directly used
```

**Memory aid:**
- **R0–R3**: "a1–a4" — **A**rguments. Caller saves them if needed after the call.
- **R4–R11**: "v1–v8" — **V**ariables. Callee saves them before using.
- **R12**: Scratch — corruptible, used by linker for long call veneers.

### Argument Passing Rules

```
Up to 4 arguments → pass in R0, R1, R2, R3
5th argument onwards → push onto stack (right-to-left order)
64-bit arguments (long long, double) → aligned pair: R0:R1 or R2:R3
Structs ≤ 4 bytes → R0
Structs > 4 bytes → pointer passed in register, struct passed on stack
```

### Return Value Rules

```
32-bit value (int, uint32_t, pointer) → R0
64-bit value (long long, double)      → R0:R1
Struct ≤ 4 bytes                      → R0
Struct > 4 bytes                      → caller allocates space, address in R0
```

### Stack Alignment

```
SP must be 8-byte aligned at any public function call boundary.
(Even though push/pop are 4-byte each — AAPCS requires 8 at entry)
Violation: Undefined behavior, possible HardFault on FPU operations
```

---

## 💻 Code Examples

### Simple 4-Argument Function

```c
// C source
int add4(int a, int b, int c, int d) {
    return a + b + c + d;
}
```

```asm
; Assembly output (GCC, ARM Thumb-2, -O0)
add4:
    ; a=R0, b=R1, c=R2, d=R3 — no stack needed for args
    ADD  R0, R0, R1        ; R0 = a + b
    ADD  R0, R0, R2        ; R0 = (a+b) + c
    ADD  R0, R0, R3        ; R0 = (a+b+c) + d  ← result in R0
    BX   LR                ; Return
```

### 5-Argument Function (Stack Spillover)

```c
int add5(int a, int b, int c, int d, int e) {
    return a + b + c + d + e;
}
```

```asm
; Caller's perspective when calling add5(1, 2, 3, 4, 5):
MOV  R0, #1          ; arg1
MOV  R1, #2          ; arg2
MOV  R2, #3          ; arg3
MOV  R3, #4          ; arg4
MOV  R4, #5
PUSH {R4}            ; arg5 → pushed to stack BEFORE the call
BL   add5
ADD  SP, SP, #4      ; Clean up arg5 from stack after call
; Result in R0

; Inside add5:
add5:
    ; R0=a, R1=b, R2=c, R3=d, [SP+0]=e
    LDR  R12, [SP]     ; Load e from stack into R12 (IP = scratch)
    ADD  R0,  R0, R1
    ADD  R0,  R0, R2
    ADD  R0,  R0, R3
    ADD  R0,  R0, R12  ; + e
    BX   LR
```

### Non-Leaf Function (Must Save LR)

```c
int factorial(int n) {       // Recursive — NOT a leaf function
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

```asm
factorial:
    PUSH {R4, LR}        ; Save R4 (we'll use it), save LR (we'll call recursively)
    MOV  R4, R0          ; R4 = n (preserve across recursive call)
    CMP  R0, #1
    BLE  base_case
    SUB  R0, R0, #1      ; R0 = n-1
    BL   factorial        ; Recursive call: LR gets overwritten → good thing we saved it!
    MUL  R0, R0, R4      ; R0 = factorial(n-1) * n
    POP  {R4, PC}        ; Restore R4, pop saved LR directly into PC = return
    ; (POP {R4, PC} is the thumb idiom for "restore and return")
base_case:
    MOV  R0, #1
    POP  {R4, PC}
```

### Callee-Saved Register Violation (Common Bug)

```asm
; BAD: using R4-R7 without saving them
my_func:
    MOV  R4, #100        ; ← WRONG: R4 is callee-saved!
    ; ... use R4 ...
    BX   LR              ; Caller expects R4 to be unchanged!
                         ; Now caller's R4 is 100 → data corruption

; CORRECT:
my_func:
    PUSH {R4, LR}        ; Save R4 (and LR if we call anything)
    MOV  R4, #100
    ; ... use R4 ...
    POP  {R4, PC}        ; Restore R4, return
```

---

## 🌍 Real-World Use Case

### Disassembling Compiler Output to Verify AAPCS

```bash
# Compile with debug info
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -O0 -g -c func.c -o func.o

# Disassemble to verify calling convention
arm-none-eabi-objdump -d func.o

# Check that:
# - R4-R11 are pushed at function start if used
# - R4-R11 are popped before return
# - Return value is in R0
# - Stack is balanced (same SP at entry and exit)
```

### Interrupt Handler — Exception to AAPCS

```c
// Hardware AUTOMATICALLY saves R0-R3, R12, LR, PC, xPSR on exception entry
// So your ISR can freely use these without push/pop
void UART_IRQHandler(void) {
    // R0-R3, R12 are already saved by hardware — use them freely
    uint32_t status = UART->SR;    // Likely uses R0 or R1
    if (status & UART_SR_RXNE) {
        process_byte(UART->DR);    // Uses R0 as argument — hardware saved it
    }
    // No need to save/restore R0-R3, R12 — hardware handles it
}
```

---

## ⚠️ Common Pitfalls

- **Not saving LR in non-leaf functions**: If your function calls another function via BL, LR gets overwritten. You MUST `PUSH {LR}` at the start (or `PUSH {R4, LR}` to maintain 8-byte alignment).
- **POP {PC} vs BX LR**: Both return, but `POP {PC}` is preferred when you've pushed LR — it combines the pop and branch into one instruction. Using `BX LR` after `POP {LR}` also works but wastes a register.
- **Odd number of pushes**: If you push an odd number of words, SP is only 4-byte aligned. Add a dummy register to maintain 8-byte alignment: `PUSH {R4, R5}` instead of `PUSH {R4}`.
- **Forgetting to clean up stack-passed arguments**: If the caller pushes a 5th argument, it must pop/adjust SP after the call returns: `ADD SP, SP, #4`.

---

## 🔗 Related Notes

- [[../Inline_Assembly/GCC_Inline_Assembly|GCC Inline Assembly]]
- [[../Stack_Operations/Push_Pop_Operations|PUSH and POP Operations]]
- [[../../02_ARM_Architecture/Registers/ARM_Register_Set|ARM Register Set]]
- [[../../08_RTOS_on_ARM/Context_Switching/Context_Switch_Mechanism|Context Switch Mechanism]]

---

## 🎯 Interview Questions

1. **Which ARM registers must a function preserve (callee-saved)?**
   *R4–R11, and SP. R0–R3, R12 are caller-saved. LR is caller-saved unless the function makes other calls, in which case the function should push LR and pop it into PC.*

2. **What happens if a function uses R5 without saving it?**
   *Silent data corruption. The caller expects R5 to be unchanged after the call (it's callee-saved). If R5 held the address of a critical buffer, the program may crash mysteriously much later.*

3. **Why must SP be 8-byte aligned at function entry?**
   *Required by AAPCS to allow 64-bit aligned data (double, long long) to be pushed to the stack. FPU operations may also require 8-byte aligned stack. Violation is undefined behavior.*

4. **How does a function return a 64-bit value?**
   *Low 32 bits in R0, high 32 bits in R1 (R0:R1 pair).*

---

*← [[../MOC_Assembly|Stage 3: ARM Assembly]]*
