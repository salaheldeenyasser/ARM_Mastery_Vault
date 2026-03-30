# Context Switch Mechanism

#advanced #arm #rtos #cortex-m #assembly

> A context switch is the mechanism by which an RTOS swaps the CPU from running one task to running another. On ARM Cortex-M, it's a carefully orchestrated dance between the PendSV handler, the stack, and a few assembly instructions.

---

## 📖 Concept Explanation

Each RTOS task thinks it owns the CPU. When the RTOS decides to switch tasks, it must:
1. **Save** the current task's CPU state (all registers) to its stack
2. **Switch** the stack pointer to the next task's stack
3. **Restore** the next task's registers from its stack
4. **Resume** the next task as if it was never interrupted

This must happen in microseconds, transparently to both tasks.

---

## 🔬 Deep Dive

### What Is a "Context"?

A task's context is its complete CPU state at any moment:
- R0–R12 (general registers)
- SP (stack pointer — each task has its own stack)
- LR (link register)
- PC (program counter — where the task was executing)
- xPSR (status flags)
- (On FPU cores) S0–S31, FPSCR

When this is saved, the task can be "frozen" and later "thawed" exactly where it left off.

### ARM Cortex-M Hardware Help

The Cortex-M hardware automatically saves and restores **8 registers** on interrupt entry/exit:

```
Automatic hardware stacking on interrupt entry:
Stack grows downward ↓

Before exception:    After exception entry (hardware saved):
                     ┌─────────────┐
                     │    xPSR     │  ← SP after hardware push
                     │     PC      │  (Return address)
                     │     LR      │
                     │     R12     │
                     │     R3      │
                     │     R2      │
                     │     R1      │
SP ─────────────►    │     R0      │  ← New SP (8 words pushed = 32 bytes)
                     └─────────────┘
```

This means when PendSV fires, R0-R3, R12, LR, PC, and xPSR are already on the task's stack. We only need to manually save R4–R11.

### Why PendSV for Context Switching?

PendSV (Pendable Service Call) is a Cortex-M exception designed specifically for RTOS context switching:
- Always set to the **lowest priority** (0xFF on most systems)
- Can be "pended" (requested to fire later) from any context
- Only fires when no other exceptions are pending
- Ensures context switch happens at the lowest priority — doesn't interfere with real-time ISRs

```
Time ──────────────────────────────────────────►

SysTick ISR fires (high priority):
  - Determines: time to switch tasks
  - Pends PendSV: SCB->ICSR |= PENDSVSET
  - Returns from SysTick

UART ISR fires (medium priority):
  - Processes received byte
  - Returns

[No more ISRs pending]

PendSV fires (lowest priority):
  - Saves current task context
  - Switches to next task's stack
  - Restores next task's context
  - Returns → next task runs
```

### The Context Switch in Assembly

```asm
; FreeRTOS PendSV handler for Cortex-M4 (simplified and annotated)
; Saved in port.c / portasm.S

PendSV_Handler:
    ; ─── Phase 1: Save current task context ───
    
    MRS     R0, PSP              ; Get Process Stack Pointer
                                 ; (hardware saved R0-R3,R12,LR,PC,xPSR on PSP)
    
    ISB                          ; Instruction Sync Barrier
    
    LDR     R3, =pxCurrentTCB   ; R3 = address of current TCB pointer
    LDR     R2, [R3]             ; R2 = current TCB pointer
    
    ; Save remaining registers R4-R11 onto task's stack
    ; (hardware already saved R0-R3, R12, LR, PC, xPSR)
    STMDB   R0!, {R4-R11, R14}  ; Push R4-R11 and LR (EXC_RETURN) onto PSP
                                 ; PSP -= 9 words (36 bytes)
    
    STR     R0, [R2]             ; Save updated PSP into TCB->pxTopOfStack
                                 ; Current task's stack pointer is now saved
    
    ; ─── Phase 2: Select next task ───
    
    STMDB   SP!, {R3, R14}      ; Save R3 and LR on MSP (we'll use MSP briefly)
    MOV     R0, #0               ; Argument: highest priority = 0
    BL      vTaskSwitchContext   ; C function: updates pxCurrentTCB to next task
                                 ; (returns with pxCurrentTCB pointing to new task)
    LDMIA   SP!, {R3, R14}      ; Restore R3 and LR from MSP
    
    ; ─── Phase 3: Restore next task context ───
    
    LDR     R1, [R3]             ; R1 = new current TCB pointer
    LDR     R0, [R1]             ; R0 = new task's saved PSP (pxTopOfStack)
    
    LDMIA   R0!, {R4-R11, R14}  ; Pop R4-R11 and LR (EXC_RETURN) from new task's stack
    
    MSR     PSP, R0              ; Restore PSP to point to new task's stack
                                 ; (hardware will restore R0-R3,R12,LR,PC,xPSR on return)
    
    ISB                          ; Instruction Sync Barrier
    
    BX      R14                  ; Return via EXC_RETURN
                                 ; Hardware unstacks R0-R3,R12,LR,PC,xPSR from new task's PSP
                                 ; → New task resumes exactly where it left off
```

### Task Stack Layout After Context Save

```
High Address ┌─────────────────────┐ ← Initial SP when task created
             │    Task Stack Area  │
             │        ...          │
             ├─────────────────────┤
             │      xPSR           │  ┐
             │      PC             │  │ Hardware-saved
             │      LR             │  │ on exception entry
             │      R12            │  │
             │      R3             │  │
             │      R2             │  │
             │      R1             │  │
             │      R0             │  ┘
             ├─────────────────────┤
             │      R14 (LR/EXC)   │  ┐
             │      R11            │  │ Manually saved
             │      R10            │  │ in PendSV handler
             │      R9             │  │
             │      R8             │  │
             │      R7             │  │
             │      R6             │  │
             │      R5             │  │
Low Address  │      R4             │  ┘ ← pxTopOfStack points here
             └─────────────────────┘
```

---

## 💻 Code Example

### Task Control Block (TCB) Structure

```c
// Simplified Task Control Block
typedef struct {
    volatile uint32_t *pxTopOfStack; // MUST be first member — assembly accesses it
    uint32_t          ulNotifiedValue;
    uint8_t           ucNotifyState;
    char              pcTaskName[16];
    uint32_t          *pxStack;       // Bottom of stack (for overflow detection)
    uint16_t          uxPriority;
    // ... more fields in real FreeRTOS
} TCB_t;

// When creating a task, FreeRTOS initializes the stack
// to look as if the task was already interrupted once:
uint32_t *pxPortInitialiseStack(uint32_t *pxTopOfStack, TaskFunction_t pxCode) {
    // Hardware frame (bottom of initial context):
    pxTopOfStack--;
    *pxTopOfStack = 0x01000000;              // xPSR: T bit = 1 (Thumb mode)
    pxTopOfStack--;
    *pxTopOfStack = (uint32_t)pxCode;        // PC: task entry point
    pxTopOfStack--;
    *pxTopOfStack = (uint32_t)prvTaskExitError; // LR: if task returns, call this
    pxTopOfStack--;
    *pxTopOfStack = 0;                       // R12
    pxTopOfStack--;
    *pxTopOfStack = 0;                       // R3
    pxTopOfStack--;
    *pxTopOfStack = 0;                       // R2
    pxTopOfStack--;
    *pxTopOfStack = 0;                       // R1
    pxTopOfStack--;
    *pxTopOfStack = (uint32_t)pvParameters;  // R0: task parameter

    // Manual save frame:
    pxTopOfStack -= 8;   // R11-R4
    memset(pxTopOfStack, 0, 8 * sizeof(uint32_t));
    
    return pxTopOfStack;  // → pxTopOfStack in TCB
}
```

---

## 🌍 Real-World Use Case

### Context Switch Timing

On STM32F407 (Cortex-M4 @ 168 MHz):
- Hardware stacking (interrupt entry): ~12 cycles = ~71 ns
- PendSV handler (save R4-R11 + switch + restore): ~50-100 cycles = ~0.6 µs
- Total context switch overhead: ~0.7 µs

For comparison, a typical RTOS tick period is 1 ms (1000 µs). Context switch overhead is **0.07% of CPU time** — negligible for most applications.

---

## ⚠️ Common Pitfalls

- **pxTopOfStack must be first TCB member**: The assembly code uses a fixed offset (0) to access `pxTopOfStack`. If you put anything before it in the struct, the assembly breaks silently.
- **FPU context not saved by default (lazy stacking)**: On Cortex-M4F with FPU, the floating point registers (S0-S31, FPSCR) are saved lazily. If your task uses FPU, ensure `configUSE_TASK_FPU_SUPPORT = 1` in FreeRTOS.
- **Stack overflow before detection**: FreeRTOS stack overflow detection only catches it when it checks, not instantly. Use watermark-based stack monitoring: `uxTaskGetStackHighWaterMark()`.
- **BASEPRI and context switching**: FreeRTOS uses `BASEPRI` to enter critical sections. If BASEPRI is accidentally left set, the SysTick (which triggers context switches) is masked — system appears to freeze.

---

## 🔗 Related Notes

- [[../Task_Scheduling/Scheduling_Algorithms|Task Scheduling Algorithms]]
- [[../../05_Cortex_M_Deep_Dive/NVIC/NVIC_Deep_Dive|NVIC Deep Dive]]
- [[../../05_Cortex_M_Deep_Dive/Exception_Model/Exception_Model_Overview|Exception Model]]
- [[../FreeRTOS/FreeRTOS_on_Cortex_M|FreeRTOS on Cortex-M]]
- [[../../02_ARM_Architecture/Registers/ARM_Register_Set|ARM Register Set]]

---

## 🎯 Interview Questions

1. **Why is PendSV used for context switching instead of SysTick?**
   *PendSV is always set to the lowest priority. This ensures context switching only happens when no other ISRs are running, preventing the switch from interfering with real-time interrupt handlers.*

2. **How many registers need to be manually saved in PendSV, and why?**
   *R4-R11 (8 registers, plus LR for EXC_RETURN = 9 words). The hardware automatically saves R0-R3, R12, LR, PC, xPSR on exception entry.*

3. **What is EXC_RETURN?**
   *A special 32-bit value placed in LR when entering an exception. It encodes: which stack was in use (MSP/PSP), whether FPU state is included, and return mode. Writing it to PC triggers exception return.*

4. **What happens to a task's stack pointer during a context switch?**
   *It's saved in the task's TCB (pxTopOfStack field). When the task is next scheduled, the RTOS loads PSP from the TCB and the hardware restores the saved registers.*

5. **What is lazy FPU stacking?**
   *An optimization where the CPU reserves space for FPU registers on the stack when entering an exception, but doesn't actually save them until the handler tries to use the FPU. This avoids the overhead when most ISRs don't use floating point.*

---

*← [[../MOC_RTOS|Stage 8: RTOS on ARM]]*
