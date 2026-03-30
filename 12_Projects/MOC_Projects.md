# 🏗️ Projects — Map of Content

#project #all-levels #arm #cortex-m

> Theory without practice is hollow. Build these in order — each project depends on knowledge from the previous one. By Project 6, you're employable.

---

## Project Progression

```
Project 1 → 2 → 3 → 4 → 5 → 6
            │   │   │   │   │
           GPIO │EXTI│RTOS│Boot│System
           UART │    │    │    │Integration
```

## Projects

- [[01_LED_Blink/LED_Blink_BareM|Project 1: LED Blink — Pure Bare-Metal]]
  *Skills: startup file, linker script, GPIO registers, Makefile*

- [[02_UART_Driver/UART_Driver_Project|Project 2: UART Driver From Scratch]]
  *Skills: UART protocol, TX/RX registers, interrupt-driven I/O, ring buffer*

- [[03_Interrupt_System/Interrupt_System_Project|Project 3: Interrupt-Based System]]
  *Skills: NVIC, EXTI, ISR design, shared state, volatile*

- [[04_RTOS_Scheduler/RTOS_Scheduler_Project|Project 4: Mini RTOS Scheduler]]
  *Skills: PendSV, SysTick, context switch, task stacks*

- [[05_Bootloader/Bootloader_Project|Project 5: Bootloader]]
  *Skills: boot process, VTOR, Flash programming, UART protocol*

- [[06_Mini_System/Mini_Embedded_System|Project 6: Mini Embedded System]]
  *Skills: everything combined — sensors, RTOS, UART, power management*

---

## Hardware Required

- STM32F407 Discovery board (~$15) — all projects
- USB-UART adapter (~$3) — Projects 2, 5, 6
- Logic analyzer (~$10) — Project 3 onward (highly recommended)
- Breadboard + jumper wires — Projects 3, 6

---

*← [[../_Dashboard/🏠 ARM Mastery Vault Home|Home]]*
