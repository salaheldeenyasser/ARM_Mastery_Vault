# ⏱️ Stage 8: RTOS on ARM — Map of Content

#advanced #arm #rtos #cortex-m #roadmap

> An RTOS isn't magic. It's carefully orchestrated hardware tricks — PendSV, SysTick, and stack manipulation. Understand the mechanism, not just the API.

---

## RTOS Concepts

- [[RTOS_Concepts/What_Is_RTOS|What is an RTOS?]]
- [[RTOS_Concepts/RTOS_vs_Bare_Metal|RTOS vs Bare-Metal — When to Use Which]]
- [[RTOS_Concepts/Determinism_and_Latency|Determinism and Interrupt Latency]]

## Task Scheduling

- [[Task_Scheduling/Scheduling_Algorithms|Scheduling Algorithms (Round-Robin, Priority, EDF)]]
- [[Task_Scheduling/Task_States|Task States — Ready, Running, Blocked, Suspended]]
- [[Task_Scheduling/Tick_and_Preemption|Tick & Preemption]]

## Context Switching

- [[Context_Switching/Context_Switch_Mechanism|Context Switch Mechanism (PendSV + Assembly)]]
- [[Context_Switching/Task_Stack_Layout|Task Stack Layout]]
- [[Context_Switching/FPU_Context|FPU Context in RTOS]]

## Synchronization

- [[Synchronization/Mutex_and_Semaphores|Mutex and Semaphores]]
- [[Synchronization/Priority_Inversion|Priority Inversion & Priority Inheritance]]
- [[Synchronization/Message_Queues|Message Queues]]
- [[Synchronization/Event_Groups|Event Groups / Event Flags]]

## FreeRTOS

- [[FreeRTOS/FreeRTOS_on_Cortex_M|FreeRTOS on Cortex-M — Setup & Configuration]]
- [[FreeRTOS/FreeRTOS_Task_API|FreeRTOS Task API]]
- [[FreeRTOS/FreeRTOS_Memory_Management|FreeRTOS Memory Management (heap_1 to heap_5)]]
- [[FreeRTOS/FreeRTOS_Debugging|Debugging FreeRTOS Applications]]

---

**Previous →** [[../07_ARM_C_Integration/MOC_C_Integration|Stage 7: ARM + C Integration]]

**Next →** [[../09_Performance_Optimization/MOC_Performance|Stage 9: Performance Optimization]]

*← [[../_Dashboard/🏠 ARM Mastery Vault Home|Home]]*
