# NVIC — Nested Vectored Interrupt Controller

#intermediate #advanced #cortex-m #arm #interrupts

> The NVIC is the interrupt brain of the Cortex-M. It receives interrupt requests, prioritizes them, and hands control to the correct handler — all in hardware, with no software overhead for vector lookup.

---

## 📖 Concept Explanation

In embedded systems, the CPU must respond to asynchronous events: a byte arrived on UART, a timer expired, a button was pressed. Without interrupts, you'd poll every peripheral constantly — wasteful and unreliable.

The NVIC (Nested Vectored Interrupt Controller) is the ARM Cortex-M hardware module that:
1. **Receives** interrupt requests from peripherals
2. **Prioritizes** them (multiple can fire simultaneously)
3. **Nests** them (higher priority can preempt lower priority)
4. **Vectors** automatically to the correct handler (no software lookup)
5. **Saves/restores** CPU state automatically (hardware stacking)

---

## 🔬 Deep Dive

### NVIC Architecture

```
                 IRQ Signals
  ┌─────────┐      │
  │  UART   │──────┤
  │  Timer  │──────┤
  │  SPI    │──────┤    ┌──────────────────────────┐
  │  DMA    │──────┼───►│           NVIC           │
  │  GPIO   │──────┤    │  - Priority registers    │◄─── Software (NVIC->IPR[])
  │  ...    │──────┤    │  - Enable registers      │◄─── Software (NVIC->ISER[])
  └─────────┘      │    │  - Pending registers     │
                        │  - Active register       │
                        └────────────┬─────────────┘
                                     │ Takes CPU
                                     ▼
                              ┌─────────────┐
                              │  Cortex-M   │
                              │    Core     │
                              └─────────────┘
```

### Key NVIC Registers (Base: 0xE000E100)

| Register | Address | Purpose |
|----------|---------|---------|
| ISER[0-7] | 0xE000E100 | Interrupt Set-Enable |
| ICER[0-7] | 0xE000E180 | Interrupt Clear-Enable |
| ISPR[0-7] | 0xE000E200 | Interrupt Set-Pending |
| ICPR[0-7] | 0xE000E280 | Interrupt Clear-Pending |
| IABR[0-7] | 0xE000E300 | Interrupt Active Bit |
| IPR[0-59] | 0xE000E400 | Interrupt Priority (8 bits/interrupt) |

### Interrupt Priority

**Priority Numbering:** **Lower number = higher priority**. Priority 0 is the highest.

**Priority Bits:** Cortex-M allows 3–8 bits of priority. STM32 uses 4 bits = 16 priority levels (0–15).

**Preemption Priority vs Subpriority:**
The priority field is split between preemption priority and subpriority, configured via `SCB->AIRCR` (PRIGROUP field).

```
Priority Register (8 bits, but typically 4 bits used on STM32):
┌──────────┬──────────┐
│ Preempt  │   Sub    │
│  Priority│ Priority │
│  [7:4]   │  [3:0]   │
└──────────┴──────────┘

PRIGROUP = 3 (typical): 4 preemption bits, 0 subpriority bits
                         = 16 preemption priority levels

High-priority interrupt (e.g. priority 1) CAN preempt
low-priority handler (e.g. priority 5).

Same preemption priority: CANNOT preempt each other.
Subpriority only determines order among same-preemption-level IRQs.
```

### Interrupt Lifecycle

```
1. PENDING: Hardware asserts IRQ signal → NVIC sets IRQ as Pending
            (NVIC->ISPR bit set)

2. ACTIVE: CPU accepts IRQ → hardware automatically:
           - Saves R0-R3, R12, LR, PC, xPSR to stack (8 words)
           - Fetches vector from interrupt vector table
           - Sets LR = EXC_RETURN (special value)
           - Jumps to interrupt handler

3. EXECUTING: Handler function runs
              (ISR_NUMBER in xPSR is non-zero)

4. RETURN: Handler executes BX LR (EXC_RETURN value)
           - Hardware unstacks R0-R3, R12, LR, PC, xPSR
           - Returns to preempted code / next pending interrupt

5. TAIL-CHAINING: If another IRQ is pending at end of handler,
                  hardware skips unstack/stack — direct vector fetch
                  (saves 12 cycles!)
```

### Exception Vector Table

```c
// The vector table must be placed at start of Flash (or at VTOR offset)
// Each entry is a 32-bit function pointer (Thumb bit = 1, so value is odd)

__attribute__((section(".isr_vector")))
const uint32_t vector_table[] = {
    // System exceptions (ARM-defined, negative IRQ numbers)
    (uint32_t)&_estack,          // [0]  Initial Stack Pointer
    (uint32_t)Reset_Handler,     // [1]  Reset
    (uint32_t)NMI_Handler,       // [2]  Non-Maskable Interrupt
    (uint32_t)HardFault_Handler, // [3]  Hard Fault
    (uint32_t)MemManage_Handler, // [4]  MemManage Fault  
    (uint32_t)BusFault_Handler,  // [5]  Bus Fault
    (uint32_t)UsageFault_Handler,// [6]  Usage Fault
    0, 0, 0, 0,                  // [7-10] Reserved
    (uint32_t)SVC_Handler,       // [11] SVC (SuperVisor Call)
    (uint32_t)DebugMon_Handler,  // [12] Debug Monitor
    0,                           // [13] Reserved
    (uint32_t)PendSV_Handler,    // [14] PendSV (used by RTOS)
    (uint32_t)SysTick_Handler,   // [15] SysTick
    
    // External interrupts (vendor-specific, IRQ0 onwards)
    (uint32_t)WWDG_IRQHandler,   // [16] IRQ0: Watchdog
    (uint32_t)EXTI0_IRQHandler,  // [17] IRQ1: External interrupt 0
    // ... up to IRQ239 (device-specific)
};
```

---

## 💻 Code Example

### Setting Up an External Interrupt (PA0) on STM32F4

```c
#include "stm32f4xx.h"

// Step 1: Enable clocks
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;   // Enable GPIOA clock
RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN;  // Enable SYSCFG clock

// Step 2: Configure PA0 as input with pull-up
GPIOA->MODER  &= ~(3U << 0);   // PA0 = input (00)
GPIOA->PUPDR  &= ~(3U << 0);
GPIOA->PUPDR  |=  (1U << 0);   // Pull-up

// Step 3: Route PA0 to EXTI0 via SYSCFG
SYSCFG->EXTICR[0] &= ~SYSCFG_EXTICR1_EXTI0;  // Select Port A for EXTI0

// Step 4: Configure EXTI0
EXTI->IMR  |= EXTI_IMR_MR0;    // Unmask EXTI0 (enable)
EXTI->RTSR |= EXTI_RTSR_TR0;   // Trigger on rising edge
EXTI->FTSR &= ~EXTI_FTSR_TR0;  // Disable falling edge

// Step 5: Set NVIC priority (0 = highest, 15 = lowest on STM32)
NVIC_SetPriority(EXTI0_IRQn, 5);  // Priority 5

// Step 6: Enable NVIC
NVIC_EnableIRQ(EXTI0_IRQn);

// Step 7: Enable global interrupts
__enable_irq();  // Clear PRIMASK (I bit)
```

```c
// The interrupt handler - name MUST match vector table
void EXTI0_IRQHandler(void) {
    if (EXTI->PR & EXTI_PR_PR0) {      // Check which EXTI line fired
        // Handle the interrupt
        LED_Toggle();
        
        EXTI->PR |= EXTI_PR_PR0;       // CRITICAL: Clear pending flag
        // Write 1 to clear (write-1-to-clear register)
    }
}
```

### NVIC Priority Configuration (CMSIS)

```c
// Using CMSIS (the proper way)
NVIC_SetPriority(TIM2_IRQn, 3);          // Set priority
NVIC_EnableIRQ(TIM2_IRQn);              // Enable
NVIC_DisableIRQ(TIM2_IRQn);             // Disable
NVIC_SetPendingIRQ(TIM2_IRQn);          // Force pending (testing)
NVIC_ClearPendingIRQ(TIM2_IRQn);        // Clear pending
uint32_t active = NVIC_GetActive(TIM2_IRQn);  // Is it currently executing?

// Special interrupt masking
__disable_irq();    // Set PRIMASK = 1 (blocks all interrupts except NMI/HardFault)
__enable_irq();     // Clear PRIMASK = 0
__set_BASEPRI(5);   // Block all interrupts with priority >= 5 (but allow 0-4)
__set_BASEPRI(0);   // Disable BASEPRI (allow all)
```

---

## 🌍 Real-World Use Case

### RTOS and PendSV

FreeRTOS uses **PendSV** (priority 15, always lowest) for context switching:
- Timer tick fires high-priority interrupt → calls `taskYIELD()` 
- `taskYIELD()` sets PendSV pending: `SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk`
- When all other handlers complete, PendSV fires
- PendSV handler saves current task context, loads next task context
- Context switch happens at lowest priority — minimizes interference

→ See [[../../08_RTOS_on_ARM/Context_Switching/Context_Switch_Mechanism|Context Switch Mechanism]]

---

## ⚠️ Common Pitfalls

- **Forgetting to clear the pending flag**: Most peripheral interrupt flags must be cleared in the ISR. If you don't clear EXTI->PR, the interrupt fires again immediately when the handler returns — infinite loop!
- **Priority inversion**: Setting a driver ISR to priority 0 blocks your RTOS tick. Rule: **RTOS interrupts should have priorities numerically higher (lower urgency) than your driver ISRs.**
- **Calling blocking RTOS functions from ISR**: You cannot call `vTaskDelay()` from an interrupt. Use `FromISR` variants: `xQueueSendFromISR()`, `xSemaphoreGiveFromISR()`.
- **Enabling interrupts before initialization**: If your UART ISR fires before the UART data structure is initialized, you'll dereference a null pointer. Enable IRQ last.
- **Stack overflow from deep nesting**: Each interrupt entry pushes 8 registers (32 bytes). Deep nesting with small stacks = overflow = hard fault with confusing symptoms.

---

## 🔗 Related Notes

- [[Exception_Model_Overview|Exception Model Overview]]
- [[../Register_Programming/Register_Level_Programming|Register-Level Programming]]
- [[../../07_ARM_C_Integration/Interrupt_Vector_Table/Vector_Table_Structure|Interrupt Vector Table Structure]]
- [[../../08_RTOS_on_ARM/Context_Switching/Context_Switch_Mechanism|Context Switch Mechanism]]
- [[../../11_Debugging_Tooling/Hard_Fault_Debug/Debugging_Hard_Faults|Debugging Hard Faults]]

---

## 🎯 Interview Questions

1. **What does "nested" mean in NVIC?**
   *A higher-priority interrupt can preempt (interrupt) a currently-executing lower-priority interrupt handler. After the high-priority handler completes, execution returns to the lower-priority handler.*

2. **What is tail-chaining?**
   *An NVIC optimization where, if another interrupt is pending at the end of a handler, the hardware skips the unstack/stack cycle (saves ~12 cycles) and directly jumps to the next handler.*

3. **What is EXC_RETURN?**
   *A special value placed in LR when entering an exception. When this value is written to PC (via BX LR or POP {PC}), the hardware knows to perform an exception return — unstacking the saved registers and returning to the preempted context.*

4. **How does the NVIC decide priority on STM32 with 4 priority bits?**
   *It compares the preemption priority portion of each pending interrupt's IPR register. Lower value = higher priority. Same preemption priority = no nesting, subpriority used for tie-breaking.*

5. **Why is the vector table entry for handlers stored as "address | 1"?**
   *Bit 0 = 1 signals Thumb mode. All Cortex-M handlers execute in Thumb mode. The linker/compiler handles this automatically for function pointers.*

---

*← [[../MOC_CortexM|Stage 5: Cortex-M Deep Dive]]*
