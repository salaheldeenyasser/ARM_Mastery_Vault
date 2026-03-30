# What is Embedded Systems?

#beginner #foundations #embedded

> An embedded system is a computer system built into a larger device, designed to do a **specific, dedicated function** — usually with tight constraints on power, cost, and real-time behavior.

---

## 📖 Concept Explanation

Unlike a general-purpose PC (which runs any program you want), an embedded system is purpose-built. The software is typically fixed, the hardware is optimized for the task, and the system must respond to the real world — often in real time.

**You interact with embedded systems dozens of times per day:**
- Your microwave (keypad scanning, timer control)
- Car ABS system (must react in <10ms — real-time critical)
- Medical pacemaker (ultra-low power, safety-critical)
- WiFi router (networking stack on a small chip)
- Your smartwatch (ARM Cortex-M + sensors + BLE)

---

## 🔬 Deep Dive

### The Embedded Constraints Triangle

```
            Performance
                 ▲
                /|\
               / | \
              /  |  \
             /   |   \
            ▼         ▼
         Power ────── Cost

Every embedded design lives inside this triangle.
Move closer to one corner, you sacrifice the others.
```

### What Makes Embedded Different from PC Programming?

| Aspect | PC Software | Embedded Software |
|--------|------------|------------------|
| OS | Windows/Linux | Bare-metal or RTOS |
| Memory | GB of RAM | KB to MB of RAM |
| Debugging | printf(), IDE | GDB + JTAG, logic analyzer |
| I/O | Files, network | GPIO, UART, SPI, I2C |
| Timing | "Fast enough" | Deterministic/real-time |
| Power | Unlimited (plugged in) | uA to mA (battery) |
| Updates | Click "Update" | JTAG/UART or OTA |
| Crash | Restart process | System hangs — real world consequences |

### Levels of Embedded Software

```
Level 4: Application Layer
         (Your sensor reading loop, state machine)
              ↓
Level 3: RTOS / OS Abstraction
         (FreeRTOS tasks, semaphores, queues)
              ↓
Level 2: HAL / Driver Layer  
         (GPIO driver, UART driver, SPI driver)
              ↓
Level 1: Peripheral Register Access
         (Direct memory-mapped register writes)
              ↓
Level 0: Hardware
         (Transistors, flip-flops, GPIO pins)
```

This vault teaches from **Level 0 upward** — you will understand each layer because you've built the one below it.

### Categories of Embedded Systems

**By Constraint:**

| Category | Example | Key Requirement |
|----------|---------|----------------|
| Hard Real-Time | Airbag controller | Miss deadline = system failure |
| Soft Real-Time | Video streaming | Miss deadline = degraded quality |
| Safety-Critical | Medical device | Failure = risk to life (DO-178C, IEC 62443) |
| Ultra-Low Power | IoT sensor node | Run for years on a coin cell |
| High Performance | Baseband processor | DSP + ARM combo |

---

## 💻 Code Example

### Bare-Metal vs RTOS vs Linux

**Bare-metal (no OS):**
```c
// Everything in one big loop — you control everything
int main(void) {
    GPIO_Init();       // Setup pin PA5 as output
    UART_Init(115200); // Setup UART at 115200 baud
    
    while(1) {
        GPIO_Toggle(PA5);       // Toggle LED
        UART_Send("Tick\r\n");  // Send over UART
        Delay_ms(500);          // Busy-wait delay
    }
}
```

**With RTOS (FreeRTOS):**
```c
// Multiple tasks running "simultaneously" via scheduler
void LED_Task(void *pvParam) {
    while(1) {
        GPIO_Toggle(PA5);
        vTaskDelay(pdMS_TO_TICKS(500)); // Non-blocking delay
    }
}

void UART_Task(void *pvParam) {
    while(1) {
        UART_Send("Tick\r\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main(void) {
    xTaskCreate(LED_Task, "LED", 128, NULL, 1, NULL);
    xTaskCreate(UART_Task, "UART", 256, NULL, 1, NULL);
    vTaskStartScheduler(); // Never returns
}
```

---

## 🌍 Real-World Use Case

**The STM32F4 Discovery Board (your learning platform):**
- ARM Cortex-M4 @ 168 MHz
- 1 MB Flash (your program lives here)
- 192 KB SRAM (your variables live here)
- USB, I2C, SPI, UART, Timers, ADC — all built in
- Cost: ~$15 USD

This single chip can run:
- A digital oscilloscope
- A motor controller
- A USB audio interface  
- A GPS data logger
- A LoRa IoT node

All with zero external ICs. That's the power of modern embedded systems.

---

## ⚠️ Common Pitfalls

- **Treating embedded like PC programming**: `printf()` doesn't "just work". You need a UART, and it costs time. In interrupt context? Dangerous.
- **Forgetting volatility**: Variables changed by hardware (interrupts, DMA) MUST be declared `volatile` — the compiler won't assume they change on its own.
- **Stack overflow**: On a PC, the OS expands your stack. On a bare-metal MCU with 4KB of stack — you overflow, you corrupt data silently.
- **Global state in interrupts**: Interrupt handlers can fire at any time. Shared state between main loop and ISR needs proper synchronization.

---

## 🔗 Related Notes

- [[What_Is_A_CPU|What is a CPU?]]
- [[MCU_vs_MPU_vs_SoC|MCU vs MPU vs SoC]]
- [[Bare_Metal_vs_RTOS_vs_Linux|Bare-Metal vs RTOS vs Linux]]
- [[../../02_ARM_Architecture/ARM_Profiles/Cortex_M_Profile|Cortex-M Profile]]
- [[../../08_RTOS_on_ARM/RTOS_Concepts/What_Is_RTOS|What is an RTOS?]]

---

## 🎯 Interview Questions

1. **What makes a system "embedded"?**
   *Purpose-built computer integrated into a larger system, doing a specific function with constraints on power, cost, size, or real-time response.*

2. **What is a hard real-time system?**
   *A system where missing a timing deadline causes a system failure (not just degraded performance). Example: automotive airbag deployment.*

3. **Why is `volatile` important in embedded C?**
   *It tells the compiler "this variable may change outside of normal program flow" — such as being modified by an interrupt or hardware register. Without it, the compiler may optimize away reads/writes.*

4. **What's the biggest difference between embedded and PC software development?**
   *Resources, determinism, and consequences. Embedded has KB of RAM, must respond in microseconds, and a crash may have physical consequences (not just a pop-up).*

---

**Next →** [[MCU_vs_MPU_vs_SoC|MCU vs MPU vs SoC]]

*← [[../MOC_Foundations|Stage 1 Foundations]]*
