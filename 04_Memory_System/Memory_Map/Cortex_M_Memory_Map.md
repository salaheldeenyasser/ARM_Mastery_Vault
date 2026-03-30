# Cortex-M Memory Map

#intermediate #arm #cortex-m #memory

> The Cortex-M memory map is a 4GB address space divided into fixed regions. Knowing this map cold is essential — it determines where your code lives, where your variables live, and how peripherals are controlled.

---

## 📖 Concept Explanation

ARM Cortex-M defines a **standardized 4GB memory map** (32-bit address space: 0x00000000 to 0xFFFFFFFF). Every Cortex-M chip — whether from ST, NXP, TI, or Nordic — follows this same map. The regions are defined by ARM; chip vendors populate them with their specific Flash, SRAM, and peripherals.

This standardization is what makes CMSIS and portable drivers possible.

---

## 🔬 The Complete Cortex-M Memory Map

```
Address Range      Region              Typical Contents
─────────────────────────────────────────────────────────────
0xFFFFFFFF ┐
           │  Vendor-Specific     Reserved / chip-specific
0xE0100000 ┘  (512 MB)

0xE00FFFFF ┐
           │  Private Peripheral  System Control Space (SCS)
           │  Bus (PPB)           - NVIC:   0xE000E100
           │  (1 MB)              - SCB:    0xE000ED00
           │                      - SysTick: 0xE000E010
           │                      - MPU:    0xE000ED90
           │                      - CoreDebug: 0xE000EDF0
0xE0000000 ┘

0xDFFFFFFF ┐
           │  External Device     External peripherals (off-chip)
           │  (1 GB)              Non-cacheable by default
0xA0000000 ┘

0x9FFFFFFF ┐
           │  External RAM        External SDRAM, SRAM
           │  (1 GB)              Cacheable (if cache present)
0x60000000 ┘

0x5FFFFFFF ┐
           │  Peripheral          APB Peripherals (slow bus)
           │  (512 MB)            - UART, SPI, I2C, DAC...
           │                      APB1: 0x40000000 - 0x40007FFF
0x40000000 ┘                      APB2: 0x40010000 - 0x40017FFF

0x3FFFFFFF ┐
           │  SRAM                AHB Peripherals (fast bus)
           │  (512 MB)            - GPIO: 0x40020000 (STM32F4)
0x20000000 ┘                      SRAM1: 0x20000000

0x1FFFFFFF ┐
           │  Code                Flash Memory:  0x08000000
           │  (512 MB)            System Memory: 0x1FFF0000 (bootloader)
           │                      Option Bytes:  0x1FFFC000
0x00000000 ┘                      Aliased at:    0x00000000 (boot config)
```

### STM32F407 Concrete Example

```
0xFFFFFFFF │                              │
           │      Vendor Specific         │
0xE0100000 │──────────────────────────────│
           │  PPB: NVIC, SCS, CoreDebug   │ ARM-defined
0xE0000000 │──────────────────────────────│
           │                              │
0x60000000 │──────────────────────────────│
           │  APB1: TIM2-7, USART2-3...   │ STM32 peripherals
0x40000000 │──────────────────────────────│
           │  AHB1: GPIO, DMA, RCC        │
0x40020000 │──────────────────────────────│
           │  SRAM1 (112KB): 0x20000000   │ Variables, Stack, Heap
           │  SRAM2 (16KB):  0x2001C000   │
0x20000000 │──────────────────────────────│
           │  Flash (1MB):   0x08000000   │ Your firmware lives here
           │  System Memory: 0x1FFF0000   │ ST's bootloader (ROM)
           │  Aliased:       0x00000000   │ Boot destination
0x00000000 │──────────────────────────────│
```

### Why 0x00000000 is Special — Boot Configuration

At reset, the CPU jumps to address 0x00000000. What it finds there depends on BOOT pins:

```
BOOT0=0, BOOT1=x  → Flash aliased to 0x00000000 (normal operation)
BOOT0=1, BOOT1=0  → System Memory (ST Bootloader) aliased
BOOT0=1, BOOT1=1  → SRAM aliased (execute from RAM — rare)
```

The vector table at 0x00000000 (or the aliased Flash start) provides:
- `[0x00]`: Initial Stack Pointer value
- `[0x04]`: Reset Handler address (first code to run)

→ See [[../../07_ARM_C_Integration/Boot_Process/ARM_Boot_Process|ARM Boot Process]]

---

## 💻 Code Example

### Accessing Peripheral Registers via Memory Map

All peripheral control is done by reading/writing memory-mapped registers:

```c
// STM32F407 - RCC (Reset and Clock Control) at 0x40023800
// Enable GPIOA clock
#define RCC_BASE      0x40023800UL
#define RCC_AHB1ENR   (*(volatile uint32_t *)(RCC_BASE + 0x30))

#define RCC_AHB1ENR_GPIOAEN  (1U << 0)  // Bit 0

RCC_AHB1ENR |= RCC_AHB1ENR_GPIOAEN;  // Write 1 to bit 0 = enable GPIOA clock

// GPIOA at 0x40020000
#define GPIOA_BASE    0x40020000UL
#define GPIOA_MODER   (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_ODR     (*(volatile uint32_t *)(GPIOA_BASE + 0x14))

// Set PA5 (bit 5) as output
GPIOA_MODER &= ~(3U << (5*2));  // Clear bits 11:10
GPIOA_MODER |=  (1U << (5*2));  // Set to "01" = output mode

// Toggle PA5
GPIOA_ODR ^= (1U << 5);
```

### Locating Sections in Your Binary

```bash
# After compiling your firmware
arm-none-eabi-objdump -h firmware.elf

# Output:
# Sections:
# Idx Name          Size      VMA       LMA       File off  Algn
#   0 .isr_vector   000001a8  08000000  08000000  00010000  2**2
#   1 .text         00004f24  080001a8  080001a8  000101a8  2**3   ← Code in Flash
#   2 .rodata       00000234  08004ecc  08004ecc  00014ecc  2**2   ← Constants in Flash
#   3 .data         00000048  20000000  08005100  00015100  2**3   ← Init'd vars in SRAM
#   4 .bss          000002a0  20000048  08005148  000151a8  2**3   ← Zero'd vars in SRAM
```

VMA = Virtual Memory Address (where it runs)
LMA = Load Memory Address (where it's stored in Flash)
For `.data`: LMA = Flash, VMA = SRAM → startup code copies it!

→ See [[../../07_ARM_C_Integration/Memory_Sections/Text_Data_BSS_Sections|.text, .data, .bss Sections]]

---

## 🌍 Real-World Use Case

### Debugging a Peripheral That Won't Work

When a peripheral isn't responding, verify the memory map:

```c
// Step 1: Is the peripheral clock enabled?
// Check RCC register at 0x40023830 (AHB1ENR)
uint32_t ahb1enr = *(volatile uint32_t *)0x40023830;
// If bit N is 0 → clock disabled → peripheral registers read as 0

// Step 2: Is the base address correct?
// STM32F4 Reference Manual → Memory Map section
// Always verify: GPIOA = 0x40020000, NOT 0x40021000 (that's GPIOB)

// Step 3: Is the register offset correct?
// GPIOA ODR offset = 0x14, not 0x10 (which is IDR)
// Reference manual table → cross-check always
```

---

## ⚠️ Common Pitfalls

- **Non-volatile registers**: ALWAYS use `volatile` when accessing hardware registers. The compiler may optimize away multiple reads if it doesn't know the value can change.
- **Endianness**: ARM is usually little-endian. A 32-bit register value `0x12345678` is stored as `78 56 34 12` in memory. Matters when you cast pointers.
- **Byte vs word access**: Some peripheral registers only support 32-bit (word) access. Accessing them with `uint8_t *` will cause a fault or give wrong results.
- **Uninitialized peripheral clock**: On STM32, ALL peripheral clocks are disabled at reset. Forgetting `RCC_AHB1ENR |= (1 << GPIOAEN)` is the #1 beginner mistake.
- **The bit-band region**: 0x20000000-0x200FFFFF and 0x40000000-0x400FFFFF have bit-band aliases at 0x22000000 and 0x42000000 — atomic single-bit access (Cortex-M3/M4 only).

---

## 🔗 Related Notes

- [[../../07_ARM_C_Integration/Memory_Sections/Text_Data_BSS_Sections|.text, .data, .bss Sections]]
- [[../../07_ARM_C_Integration/Boot_Process/ARM_Boot_Process|ARM Boot Process]]
- [[../Stack_vs_Heap/Stack_Memory_Model|Stack Memory Model]]
- [[../../05_Cortex_M_Deep_Dive/NVIC/NVIC_Deep_Dive|NVIC (0xE000E100)]]
- [[../../06_Peripherals_BareMetal/GPIO/GPIO_Register_Programming|GPIO Register Programming]]

---

## 🎯 Interview Questions

1. **What address does the Cortex-M CPU jump to after reset?**
   *It reads the initial Stack Pointer from 0x00000000 and the Reset Handler address from 0x00000004, then jumps there. What's at 0x00000000 depends on BOOT pin configuration.*

2. **Where is the NVIC in the Cortex-M memory map?**
   *In the Private Peripheral Bus (PPB) region: 0xE000E000-0xE000EFFF. Specifically NVIC starts at 0xE000E100.*

3. **Why do peripheral registers need to be declared volatile?**
   *Because their values can change independently of program flow (hardware changes them). Without volatile, the compiler may cache the value in a register and never re-read it.*

4. **What's the difference between VMA and LMA?**
   *VMA (Virtual Memory Address) = where code/data runs. LMA (Load Memory Address) = where it's stored in Flash. The startup code copies .data from LMA (Flash) to VMA (SRAM) before main().*

5. **On STM32, GPIOA doesn't respond. First thing to check?**
   *Is the GPIOA clock enabled in RCC_AHB1ENR? All peripheral clocks are disabled at reset.*

---

*← [[../MOC_Memory|Stage 4: Memory System]]*
