# 🔧 Stage 5: Cortex-M Deep Dive — Map of Content

#intermediate #advanced #cortex-m #arm #roadmap

> The Cortex-M is where your embedded career lives. Every STM32, Nordic nRF, NXP LPC, and Microchip SAM you touch uses one. Master this stage and you're genuinely dangerous.

---

## NVIC — Interrupt Controller

- [[NVIC/NVIC_Deep_Dive|NVIC Architecture & Operation]]
- [[NVIC/Interrupt_Priority|Interrupt Priority & Grouping]]
- [[NVIC/Interrupt_Masking|PRIMASK, FAULTMASK, BASEPRI]]
- [[NVIC/Enabling_Disabling_IRQ|Enabling & Disabling Interrupts]]

## Exception Model

- [[Exception_Model/Exception_Model_Overview|Exception Model Overview]]
- [[Exception_Model/Exception_Types|Exception Types — IRQ, NMI, SysTick, Faults]]
- [[Exception_Model/Hardware_Stacking|Hardware Stacking on Exception Entry]]
- [[Exception_Model/EXC_RETURN_Values|EXC_RETURN Values Explained]]
- [[Exception_Model/Fault_Escalation|Fault Escalation to HardFault]]

## SysTick Timer

- [[SysTick/SysTick_Timer|SysTick Timer — Setup & Use]]
- [[SysTick/SysTick_as_RTOS_Tick|SysTick as RTOS Tick Source]]

## Low Power Modes

- [[Low_Power/Low_Power_Modes|WFI, WFE, Sleep, Deep Sleep]]
- [[Low_Power/Clock_Gating|Clock Gating Strategies]]
- [[Low_Power/RUN_STOP_STANDBY|RUN / STOP / STANDBY Modes (STM32)]]

## Register-Level Programming

- [[Register_Programming/Register_Level_Programming|Direct Register Access Techniques]]
- [[Register_Programming/Bit_Banding|Bit-Banding (Cortex-M3/M4)]]
- [[Register_Programming/SCB_Registers|System Control Block Registers]]

## CMSIS

- [[CMSIS/CMSIS_Overview|CMSIS — ARM's Hardware Abstraction]]
- [[CMSIS/CMSIS_Core_Functions|CMSIS Core Functions]]
- [[CMSIS/CMSIS_vs_HAL|CMSIS vs Vendor HAL]]

---

**Previous Stage →** [[../04_Memory_System/MOC_Memory|Stage 4: Memory System]]

**Next Stage →** [[../06_Peripherals_BareMetal/MOC_Peripherals|Stage 6: Peripherals & Bare-Metal]]

*← [[../_Dashboard/🏠 ARM Mastery Vault Home|Home]]*
