# ARM Architecture Interview Questions

#interview #arm #beginner #intermediate #advanced

> These are real questions asked at embedded engineering interviews. Organized by difficulty. Each question has a model answer — but understand it, don't memorize it.

---

## 🟢 Beginner Level

**Q1: What does ARM stand for and who makes ARM chips?**

> ARM originally stood for "Acorn RISC Machine," later rebranded to "Advanced RISC Machines." ARM Ltd (now ARM Holdings) designs the architecture and licenses it. They don't make chips — companies like ST, NXP, Qualcomm, Apple, and TI manufacture chips using ARM licenses.

---

**Q2: What is RISC and how does ARM implement it?**

> RISC (Reduced Instruction Set Computer) is a CPU design philosophy emphasizing simple, fixed-size instructions that execute in one clock cycle (usually). ARM implements RISC with: fixed 32-bit instructions (Thumb: 16-bit), a load-store architecture (only LOAD/STORE access memory — everything else operates on registers), and a large register file (16 registers) to minimize memory accesses.

---

**Q3: What are the three Cortex profiles and their use cases?**

> - **Cortex-M**: Microcontrollers. Low-power, real-time, deterministic. Used in MCUs (STM32, Nordic, NXP Kinetis). This is what embedded engineers work with most.
> - **Cortex-A**: Application processors. High performance, runs full OS (Linux, Android). Used in phones, tablets, Raspberry Pi.
> - **Cortex-R**: Real-time processors. Used in automotive ABS, hard drive controllers — where hard real-time with ECC memory is critical.

---

**Q4: How many registers does Cortex-M have? Name the special ones.**

> 16 general-purpose 32-bit registers (R0–R15), plus xPSR.
> - R13 = SP (Stack Pointer)
> - R14 = LR (Link Register — return address)
> - R15 = PC (Program Counter)
> - xPSR = Program Status Register (flags: N, Z, C, V)

---

**Q5: What is the difference between Thumb and Thumb-2?**

> **Thumb**: 16-bit compressed instruction encoding for ARM7/9 era. Smaller code size (~30% smaller than ARM 32-bit), but limited — not all operations available.
> **Thumb-2**: Mixed 16/32-bit instruction set introduced with ARMv7. Best of both worlds — small code size + full instruction capability. Cortex-M uses ONLY Thumb-2 (no 32-bit ARM mode).

---

## 🟡 Intermediate Level

**Q6: Explain the ARM Cortex-M exception model.**

> When an interrupt fires:
> 1. CPU finishes current instruction
> 2. Hardware automatically pushes R0-R3, R12, LR, PC, xPSR onto the stack (32 bytes)
> 3. LR gets a special EXC_RETURN value
> 4. CPU fetches the handler address from the vector table
> 5. Handler executes
> 6. Handler returns via `BX LR` with EXC_RETURN value
> 7. Hardware automatically pops the saved registers
> 8. Execution resumes where it was interrupted
>
> This is "hardware-assisted context save" — no software prologue needed in handlers.

---

**Q7: What is the NVIC and how does interrupt priority work?**

> The NVIC (Nested Vectored Interrupt Controller) manages interrupt reception, prioritization, and preemption. Priority is a number where **lower = higher priority**. STM32 typically uses 4 priority bits = 16 levels (0–15). Priority 0 = highest, can preempt anything. Higher-priority interrupts can preempt lower-priority handlers (nesting). The BASEPRI register can mask all interrupts below a threshold without fully disabling interrupts.

---

**Q8: What is the difference between MSP and PSP?**

> ARM Cortex-M has two stack pointers:
> - **MSP (Main Stack Pointer)**: Used by exception handlers and the privileged thread mode (or bare-metal main). Reset default.
> - **PSP (Process Stack Pointer)**: Intended for RTOS tasks (unprivileged thread mode). FreeRTOS switches to PSP for tasks, keeping MSP for the OS kernel and interrupt handlers.
>
> The active SP (R13) is selected by the CONTROL register. This separation means a misbehaving task can't easily corrupt the kernel stack.

---

**Q9: Explain the Cortex-M memory map. Where is Flash, SRAM, and the NVIC?**

> The Cortex-M defines a standardized 4GB memory map:
> - `0x00000000 – 0x1FFFFFFF`: Code region (Flash typically at 0x08000000 on STM32)
> - `0x20000000 – 0x3FFFFFFF`: SRAM region (SRAM at 0x20000000)
> - `0x40000000 – 0x5FFFFFFF`: Peripheral region (APB/AHB peripherals)
> - `0xE000E000 – 0xE000EFFF`: Private Peripheral Bus (NVIC at 0xE000E100, SCB at 0xE000ED00)
>
> This layout is defined by ARM — every Cortex-M vendor follows it, which enables portable drivers.

---

**Q10: What does `volatile` do in embedded C and when must you use it?**

> `volatile` tells the compiler: "this variable may change outside of your view — do not optimize away reads or writes." Required when:
> 1. Variable is written by an interrupt handler and read in main loop
> 2. Variable maps to a hardware register (peripheral register)
> 3. Variable is written by DMA
> 4. Variable is used in a spin-wait loop that another thread/ISR terminates
>
> Without `volatile`, the compiler may cache the value in a register and never re-read it, causing your code to miss hardware-driven changes.

---

## 🔴 Advanced Level

**Q11: Walk me through what happens when a HardFault fires on Cortex-M.**

> 1. An exception fires (bus fault, mem fault, usage fault, or one of these escalates to HardFault)
> 2. CPU completes current instruction (or abandons a load/store mid-transfer)
> 3. Hardware pushes a frame: R0-R3, R12, LR, PC, xPSR onto the stack in use (MSP or PSP depending on which was active)
> 4. LR = EXC_RETURN (encodes which stack was in use)
> 5. CPU reads HardFault handler address from vector table (0x0000000C)
> 6. HardFault_Handler executes
> 7. To diagnose: read CFSR (Configurable Fault Status Register) at 0xE000ED28 for reason, BFAR/MMFAR for the fault address, and sp[6] (stacked PC) for the crashing instruction address

---

**Q12: How does a context switch work in FreeRTOS on Cortex-M?**

> 1. SysTick (1ms tick) fires → scheduler determines if switch needed → pends PendSV
> 2. PendSV fires at lowest priority (when all other ISRs done)
> 3. PendSV handler (assembly):
>    - Read PSP (current task's stack pointer)
>    - Push R4-R11 and EXC_RETURN to task's stack (hardware already saved R0-R3, R12, LR, PC, xPSR)
>    - Save PSP to current TCB (pxTopOfStack)
>    - Call vTaskSwitchContext() → selects next task → updates pxCurrentTCB
>    - Load new task's PSP from its TCB
>    - Pop R4-R11 from new task's stack
>    - Return via BX LR (EXC_RETURN) → hardware restores R0-R3, R12, LR, PC, xPSR
> 4. New task runs as if it was never interrupted

---

**Q13: What is the MPU and how would you use it to protect a critical buffer?**

> The MPU (Memory Protection Unit) on Cortex-M3/M4/M7 defines up to 8 memory regions with access permissions and attributes. Use cases:
> - Prevent tasks from writing to Flash
> - Catch null pointer dereferences (make region 0x00000000 no-access)
> - Isolate RTOS task stacks — if a task overflows its stack, MPU triggers MemManage fault instead of silently corrupting other tasks
>
> To protect a buffer: define an MPU region at the buffer address, set size, and limit access to privileged code only. Unprivileged task write → MemManage fault → caught deterministically.

---

**Q14: What is TrustZone and how does it differ from privilege levels?**

> **Privilege levels** (privileged/unprivileged in ARMv7-M) only affect access to system registers and MPU enforcement — both run in the same security domain.
>
> **TrustZone** (ARMv8-M) creates two completely isolated worlds:
> - **Secure World**: Has access to all memory and peripherals. Contains crypto keys, secure boot code, trusted services.
> - **Non-Secure World**: Normal application code. Cannot access Secure World memory — hardware-enforced.
>
> Transition via `SG` (Secure Gateway) instruction at defined entry points. Used in IoT devices for storing certificates, keys, and secure boot without exposing them to the application.

---

**Q15: Describe the ARM boot sequence from power-on to main().**

> 1. Reset released: CPU fetches initial SP from address 0x00000000 → loads into MSP
> 2. CPU fetches Reset_Handler address from 0x00000004 → jumps to it
> 3. Reset_Handler (startup code) runs:
>    a. Copies .data section from Flash LMA to SRAM VMA
>    b. Zeros .bss section in SRAM
>    c. (Optionally) initializes FPU, caches, clock
>    d. Calls main()
> 4. main() runs
>
> The "0x00000000" is aliased to Flash (0x08000000) via BOOT0 pin configuration. The vector table in Flash provides the SP and Reset_Handler address.

---

## ⚫ Professional/Expert Level

**Q16: Explain cache coherency issues in a multi-core ARM system with DMA.**

> When the CPU writes to a cache line, the DMA may read stale data from main memory (or vice versa). Solutions:
> - **Cache clean**: Write dirty cache lines back to memory before DMA reads
> - **Cache invalidate**: Discard cache lines before CPU reads data written by DMA
> - **Non-cacheable regions**: Map DMA buffers to non-cacheable memory (NORMAL NC or DEVICE type in MPU/MMU)
> - **Coherency hardware**: Full ARM Cortex-A clusters with ACE/CHI bus have hardware coherency (CCI/CCN)
>
> On Cortex-M7 (which has cache), DMA buffers MUST be in non-cacheable SRAM or explicitly cleaned/invalidated using SCB_CleanDCache_by_Addr() and SCB_InvalidateDCache_by_Addr().

---

**Q17: What is the ARM AAPCS? What happens if you violate it?**

> AAPCS (ARM Architecture Procedure Call Standard) defines:
> - R0-R3: Arguments and return values (caller-saved — caller must save before function call if needed)
> - R4-R11: General purpose (callee-saved — function must preserve these)
> - R12 (IP): Intra-procedure scratch register
> - R13 (SP): Must be 8-byte aligned on entry
> - R14 (LR): Return address
>
> Violating it (e.g., modifying R5 in a function without saving it) causes silent data corruption in the caller — the value in R5 after the call is garbage. Incredibly hard to debug because the corruption may be far from the root cause.

---

*← [[../MOC_Interview|Interview Preparation]] | [[../../_Dashboard/🏠 ARM Mastery Vault Home|Home]]*
