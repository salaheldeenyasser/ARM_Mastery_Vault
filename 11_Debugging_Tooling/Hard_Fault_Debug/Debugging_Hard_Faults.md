# Debugging Hard Faults on ARM Cortex-M

#advanced #debugging #cortex-m #arm

> A HardFault is the CPU's way of saying "something went catastrophically wrong." Learning to decode a HardFault is one of the most valuable embedded engineering skills — it turns a mysterious crash into a precise diagnosis.

---

## 📖 Concept Explanation

A **HardFault** is a CPU exception that fires when a fault cannot be handled by a specific fault handler, or when a fault occurs and the specific fault handler is disabled.

Common causes:
- Null pointer dereference
- Stack overflow (SP into invalid memory)
- Unaligned access (on Cortex-M0/M0+)
- Executing undefined instructions
- Bus fault (accessing non-existent memory)
- Calling a non-Thumb function pointer

---

## 🔬 The HardFault Diagnosis System

### Step 1: Capture the Stacked PC

When a HardFault fires, the CPU automatically pushes registers onto the stack. The **stacked PC** is the address of the instruction that caused the fault.

```c
// A proper HardFault handler that captures the context
// This must be in assembly to get the stack frame correctly

// In your startup or port file:
__attribute__((naked))
void HardFault_Handler(void) {
    __asm volatile(
        " tst   lr, #4      \n"  // Test bit 2 of EXC_RETURN
        " ite   eq          \n"  // If EQ (bit2=0): used MSP
        " mrseq r0, msp     \n"  // R0 = MSP (main stack)
        " mrsne r0, psp     \n"  // R0 = PSP (process/task stack)
        " b     HardFault_HandlerC \n"
    );
}

// C handler receives the stack frame pointer in R0
void HardFault_HandlerC(uint32_t *sp) {
    // Extract the stacked registers
    volatile uint32_t r0  = sp[0];
    volatile uint32_t r1  = sp[1];
    volatile uint32_t r2  = sp[2];
    volatile uint32_t r3  = sp[3];
    volatile uint32_t r12 = sp[4];
    volatile uint32_t lr  = sp[5];   // Link Register at time of fault
    volatile uint32_t pc  = sp[6];   // PC = ADDRESS OF THE FAULTING INSTRUCTION
    volatile uint32_t psr = sp[7];

    // Read fault status registers
    volatile uint32_t cfsr  = SCB->CFSR;   // Configurable Fault Status Register
    volatile uint32_t hfsr  = SCB->HFSR;   // HardFault Status Register
    volatile uint32_t dfsr  = SCB->DFSR;   // Debug Fault Status Register
    volatile uint32_t afsr  = SCB->AFSR;   // Auxiliary Fault Status Register
    volatile uint32_t mmfar = SCB->MMFAR;  // MemManage Fault Address Register
    volatile uint32_t bfar  = SCB->BFAR;   // BusFault Address Register

    // Breakpoint here in GDB — examine all variables above
    __asm volatile("bkpt #01");
    while(1);
}
```

### Step 2: Read the Fault Status Registers

The **CFSR** (Configurable Fault Status Register) at `0xE000ED28` is split into three sub-registers:

```
CFSR Layout:
Bit:  31                  16  15           8  7             0
      ├── UFSR (Usage) ──────┤── BFSR ──────┤── MMFSR ──────┤
```

#### MMFSR — MemManage Fault Status Register (Byte 0)

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | IACCVIOL | Instruction access violation (exec from protected region) |
| 1 | DACCVIOL | Data access violation (R/W to protected region) |
| 3 | MUNSTKERR | Fault on unstacking for exception return |
| 4 | MSTKERR | Fault on stacking for exception entry |
| 5 | MLSPERR | Fault during FP lazy state preservation |
| 7 | MMARVALID | MMFAR holds valid address |

#### BFSR — BusFault Status Register (Byte 1)

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | IBUSERR | Bus fault on instruction fetch |
| 1 | PRECISERR | Precise data bus error (BFAR is valid) |
| 2 | IMPRECISERR | Imprecise data bus error (address may not be available) |
| 3 | UNSTKERR | Fault on unstacking |
| 4 | STKERR | Fault on stacking |
| 5 | LSPERR | Fault during FP lazy state preservation |
| 7 | BFARVALID | BFAR holds valid fault address |

#### UFSR — UsageFault Status Register (Halfword bits 31:16)

| Bit | Name | Meaning |
|-----|------|---------|
| 16 | UNDEFINSTR | Undefined instruction executed |
| 17 | INVSTATE | Invalid EPSR state (Thumb bit issue) |
| 18 | INVPC | Invalid PC load (branch to EXC_RETURN outside handler) |
| 19 | NOCP | Coprocessor access fault (FPU not enabled?) |
| 24 | UNALIGNED | Unaligned access trap (if UNALIGN_TRP in CCR set) |
| 25 | DIVBYZERO | Division by zero (if DIV_0_TRP in CCR set) |

---

## 💻 Code Example

### GDB Hard Fault Post-Mortem Analysis

```bash
# Connect GDB to your target
arm-none-eabi-gdb firmware.elf
(gdb) target remote localhost:3333
(gdb) monitor reset halt

# When the HardFault fires (breakpoint in HardFault_HandlerC):
(gdb) info registers
r0             0x00000000  0          ← Suspicious: R0 = NULL
r1             0x20001234  536875572
r2             0x00000005  5
...
sp             0x2000ff80  0x2000ff80
lr             0x08001234  main.c:42  ← What called the faulting function
pc             0x08000abc  BadFunc+16 ← WHERE THE CRASH HAPPENED

# Decode the PC to source line:
(gdb) list *0x08000abc
0x8000abc is in BadFunc (main.c:89).
# This tells you exactly which line crashed!

# Examine the fault status registers:
(gdb) x/xw 0xE000ED28
0xe000ed28: 0x00008200    ← CFSR
# 0x00008200 = BFSR bit 9 (PRECISERR) and bit 15 (BFARVALID)
# → Precise bus fault, and BFAR is valid!

(gdb) x/xw 0xE000ED38
0xe000ed38: 0xdeadbeef   ← BFAR: the address that caused the fault!
# 0xdeadbeef → you dereferenced an uninitialized/corrupted pointer
```

### Stack Overflow Detection

```c
// Check if fault was caused by stack overflow
void HardFault_HandlerC(uint32_t *sp) {
    // Check stack pointer is within valid SRAM range
    extern uint32_t _estack;       // Top of stack (from linker script)
    extern uint32_t _Min_Stack_Size;
    
    uint32_t stack_bottom = (uint32_t)&_estack - (uint32_t)&_Min_Stack_Size;
    uint32_t current_sp = (uint32_t)sp;
    
    if (current_sp < stack_bottom) {
        // Stack overflow! SP is below the stack bottom
        // Don't trust ANY local variables — they may be corrupted
        while(1);  // With a way to identify "STACK_OVERFLOW" fault
    }
    
    // FreeRTOS: Check task stack watermark
    // uxTaskGetStackHighWaterMark(NULL) → words remaining
}
```

### Decoding a Real HardFault

```
Scenario: System crashes intermittently after 10 minutes.

1. Run HardFault handler → capture registers:
   pc = 0x08003456
   r0 = 0x00000000  ← NULL pointer!
   cfsr = 0x00000001 ← MMFSR bit 0: IACCVIOL

2. GDB: list *0x08003456
   → Can't display (stripped binary) or → uart_send.c:45
   uart_send.c:45: pUART->DR = *buffer++;

3. Root cause: pUART is NULL — the UART handle was not initialized
   before uart_send() was called (race condition in task startup order).

Fix: Add NULL check, fix initialization order.
```

---

## 🌍 Real-World Use Case

### Building a Fault Logger to Flash

```c
// Save fault info to non-volatile memory for post-mortem analysis
typedef struct {
    uint32_t magic;    // 0xDEADBEEF = valid entry
    uint32_t pc;
    uint32_t lr;
    uint32_t cfsr;
    uint32_t hfsr;
    uint32_t bfar;
    uint32_t mmfar;
    uint32_t sp_at_fault;
} FaultRecord_t;

// Store in a dedicated Flash sector that survives resets
#define FAULT_LOG_ADDRESS 0x080E0000   // Last sector of 1MB Flash

void HardFault_HandlerC(uint32_t *sp) {
    FaultRecord_t record = {
        .magic        = 0xDEADBEEF,
        .pc           = sp[6],
        .lr           = sp[5],
        .cfsr         = SCB->CFSR,
        .hfsr         = SCB->HFSR,
        .bfar         = SCB->BFAR,
        .mmfar        = SCB->MMFAR,
        .sp_at_fault  = (uint32_t)sp,
    };
    
    Flash_Write(FAULT_LOG_ADDRESS, &record, sizeof(record));
    NVIC_SystemReset();  // Reset and allow recovery
}

// On startup, check for saved fault:
void check_fault_log(void) {
    FaultRecord_t *log = (FaultRecord_t *)FAULT_LOG_ADDRESS;
    if (log->magic == 0xDEADBEEF) {
        // Send log over UART, then erase the sector
        UART_Printf("FAULT: PC=0x%08lX LR=0x%08lX CFSR=0x%08lX\r\n",
                    log->pc, log->lr, log->cfsr);
        Flash_EraseSector(FAULT_LOG_ADDRESS);
    }
}
```

---

## ⚠️ Common Pitfalls

- **Imprecise bus faults**: When using write buffers (DMA or certain buses), the fault may be reported several instructions AFTER the actual bad access. CFSR shows IMPRECISERR. Disable write buffering temporarily (set DISDEFWBUF in ACTLR) to make it precise for debugging.
- **Fault in fault handler (double fault)**: If your HardFault handler itself causes a fault, you get a lockup. Keep HardFault handler simple — don't malloc, don't use UART unless it was already initialized.
- **Nakedness is required**: The assembly thunk MUST use `__attribute__((naked))` — the compiler must not add prologue/epilogue code, which would corrupt the stack frame pointer.
- **Wrong stack used**: Bit 2 of EXC_RETURN tells you MSP vs PSP. In RTOS tasks, the task stack is on PSP, not MSP. Using the wrong one gives garbage register values.

---

## 🔗 Related Notes

- [[../../05_Cortex_M_Deep_Dive/Exception_Model/Exception_Model_Overview|Exception Model]]
- [[../../05_Cortex_M_Deep_Dive/NVIC/NVIC_Deep_Dive|NVIC Deep Dive]]
- [[GDB_for_Embedded|GDB for Embedded Systems]]
- [[JTAG_vs_SWD|JTAG vs SWD]]
- [[../../04_Memory_System/Stack_vs_Heap/Stack_Memory_Model|Stack Memory Model]]
- [[../../07_ARM_C_Integration/Interrupt_Vector_Table/Vector_Table_Structure|Vector Table Structure]]

---

## 🎯 Interview Questions

1. **What is the first thing you do when a HardFault fires?**
   *Read the stacked PC (at sp[6]) to find where it crashed, and read CFSR to understand why.*

2. **What is the difference between PRECISERR and IMPRECISERR in BFSR?**
   *PRECISERR: the fault address is exact and BFAR is valid. IMPRECISERR: due to write buffering, the fault is reported late and the address may not be valid.*

3. **How do you tell if a HardFault was caused by a stack overflow?**
   *Check if SP < stack bottom (from linker script symbols). Also, if MSTKERR in MMFSR is set, the fault occurred during exception stacking — often means stack was already corrupted.*

4. **What does EXC_RETURN bit 2 tell you in the HardFault handler?**
   *Bit 2 = 0 means MSP was in use (privileged/handler code crashed). Bit 2 = 1 means PSP was in use (RTOS task crashed). You use this to read the correct stack.*

5. **How would you debug a HardFault in a deployed product with no JTAG?**
   *Save fault registers (PC, LR, CFSR, BFAR, MMFAR) to non-volatile memory (Flash or EEPROM) in the HardFault handler, then reset. On next boot, read and report the saved fault data.*

---

*← [[../MOC_Debugging|Stage 11: Debugging & Tooling]]*
