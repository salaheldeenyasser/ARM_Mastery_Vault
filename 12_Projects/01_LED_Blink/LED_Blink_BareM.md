# Project 1: LED Blink — Pure Bare-Metal

#project #beginner #intermediate #cortex-m #arm

> The "Hello World" of embedded systems — but done the right way. No HAL, no CMSIS startup magic, no IDE hand-holding. Just you, the datasheet, and the registers.

---

## 🎯 Project Goal

Blink the onboard LED on an STM32F407 Discovery board **without using any vendor HAL library**. Every register write is understood and intentional.

**What you will learn:**
- How to read and navigate a chip datasheet
- The ARM boot process (Reset_Handler → main)
- Memory-mapped peripheral control
- Writing a startup file and linker script by hand
- Building with Makefile + arm-none-eabi-gcc

---

## 🔬 Hardware: STM32F407 Discovery

```
Board: STM32F407G-DISC1
CPU: ARM Cortex-M4 @ 168 MHz (running at default 16 MHz HSI)
LED: PD12 (Green), PD13 (Orange), PD14 (Red), PD15 (Blue)

Peripheral path to control PD13 (Orange LED):
  1. Enable GPIOD clock in RCC
  2. Configure PD13 as output in GPIOD_MODER  
  3. Toggle PD13 in GPIOD_ODR
```

---

## 📁 Project Structure

```
01_LED_Blink/
├── Makefile
├── linker.ld
├── startup.c
├── main.c
└── README.md
```

---

## 💻 Implementation

### `linker.ld` — Memory Layout

```ld
/* STM32F407: 1MB Flash @ 0x08000000, 192KB SRAM @ 0x20000000 */
ENTRY(Reset_Handler)

_estack = 0x20020000;  /* End of SRAM (128KB SRAM1) */

MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    SRAM  (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
    .isr_vector : {
        KEEP(*(.isr_vector))
    } >FLASH

    .text : {
        *(.text*)
        *(.rodata*)
    } >FLASH

    _sidata = LOADADDR(.data);

    .data : {
        _sdata = .;
        *(.data*)
        _edata = .;
    } >SRAM AT>FLASH

    .bss : {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        _ebss = .;
    } >SRAM
}
```

### `startup.c` — Reset Handler & Vector Table

```c
#include <stdint.h>

/* Linker script symbols */
extern uint32_t _estack;
extern uint32_t _sidata;
extern uint32_t _sdata, _edata;
extern uint32_t _sbss,  _ebss;

/* Forward declarations */
void Reset_Handler(void);
void Default_Handler(void);
int  main(void);

/* Fault handlers — weak aliases to Default_Handler */
void NMI_Handler(void)       __attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void) __attribute__((weak, alias("Default_Handler")));
void MemManage_Handler(void) __attribute__((weak, alias("Default_Handler")));
void BusFault_Handler(void)  __attribute__((weak, alias("Default_Handler")));
void UsageFault_Handler(void)__attribute__((weak, alias("Default_Handler")));
void SVC_Handler(void)       __attribute__((weak, alias("Default_Handler")));
void PendSV_Handler(void)    __attribute__((weak, alias("Default_Handler")));
void SysTick_Handler(void)   __attribute__((weak, alias("Default_Handler")));

/* Vector Table — MUST be first in Flash */
__attribute__((section(".isr_vector")))
uint32_t vector_table[] = {
    (uint32_t)&_estack,           /* Initial Stack Pointer */
    (uint32_t)Reset_Handler,      /* Reset Handler */
    (uint32_t)NMI_Handler,        /* NMI */
    (uint32_t)HardFault_Handler,  /* HardFault */
    (uint32_t)MemManage_Handler,  /* MemManage */
    (uint32_t)BusFault_Handler,   /* BusFault */
    (uint32_t)UsageFault_Handler, /* UsageFault */
    0, 0, 0, 0,                   /* Reserved */
    (uint32_t)SVC_Handler,        /* SVCall */
    0, 0,                         /* Reserved */
    (uint32_t)PendSV_Handler,     /* PendSV */
    (uint32_t)SysTick_Handler,    /* SysTick */
    /* External interrupts would follow... */
};

/* Reset Handler — first code to execute after power-on */
void Reset_Handler(void) {
    /* Copy .data from Flash to SRAM */
    uint32_t *src = &_sidata;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) {
        *dst++ = *src++;
    }

    /* Zero-fill .bss */
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0;
    }

    /* Call main */
    main();

    /* Should never return — spin forever */
    while (1);
}

/* Default fault handler — spin to allow GDB attach */
void Default_Handler(void) {
    while (1);
}
```

### `main.c` — LED Blink Logic

```c
#include <stdint.h>

/*
 * STM32F407 Register Addresses
 * Source: STM32F407 Reference Manual (RM0090), Chapter 7 (RCC) & Chapter 8 (GPIO)
 */

/* RCC: Reset and Clock Control */
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t *)(RCC_BASE + 0x30UL))

/* Bit definitions for RCC_AHB1ENR */
#define RCC_AHB1ENR_GPIOAEN  (1U << 0)
#define RCC_AHB1ENR_GPIODEN  (1U << 3)   /* GPIOD clock enable */

/* GPIOD: General Purpose I/O Port D */
#define GPIOD_BASE      0x40020C00UL      /* From STM32F4 memory map */
#define GPIOD_MODER     (*(volatile uint32_t *)(GPIOD_BASE + 0x00UL))
#define GPIOD_OTYPER    (*(volatile uint32_t *)(GPIOD_BASE + 0x04UL))
#define GPIOD_OSPEEDR   (*(volatile uint32_t *)(GPIOD_BASE + 0x08UL))
#define GPIOD_PUPDR     (*(volatile uint32_t *)(GPIOD_BASE + 0x0CUL))
#define GPIOD_ODR       (*(volatile uint32_t *)(GPIOD_BASE + 0x14UL))
#define GPIOD_BSRR      (*(volatile uint32_t *)(GPIOD_BASE + 0x18UL))

/* GPIO MODER values */
#define GPIO_MODER_INPUT    0x0U
#define GPIO_MODER_OUTPUT   0x1U
#define GPIO_MODER_AF       0x2U
#define GPIO_MODER_ANALOG   0x3U

/* Pin definitions */
#define LED_GREEN   12  /* PD12 */
#define LED_ORANGE  13  /* PD13 */
#define LED_RED     14  /* PD14 */
#define LED_BLUE    15  /* PD15 */

/* Simple busy-wait delay */
static void delay_ms(uint32_t ms) {
    /* At 16 MHz HSI (default after reset), roughly calibrated */
    /* Better: use SysTick → see Stage 5 SysTick note */
    volatile uint32_t count = ms * 1600;
    while (count--);
}

static void gpio_init(void) {
    /* Step 1: Enable GPIOD clock */
    RCC_AHB1ENR |= RCC_AHB1ENR_GPIODEN;
    /* Small delay after enabling clock (read-back to flush) */
    (void)RCC_AHB1ENR;

    /* Step 2: Configure PD12-PD15 as outputs */
    /* MODER register: 2 bits per pin
     * PD12 → bits [25:24]
     * PD13 → bits [27:26]
     * PD14 → bits [29:28]
     * PD15 → bits [31:30]
     */

    /* Clear mode bits for PD12-PD15 */
    GPIOD_MODER &= ~(0xFFU << (LED_GREEN * 2));

    /* Set output mode (01) for PD12-PD15 */
    GPIOD_MODER |= (GPIO_MODER_OUTPUT << (LED_GREEN * 2))  |
                   (GPIO_MODER_OUTPUT << (LED_ORANGE * 2)) |
                   (GPIO_MODER_OUTPUT << (LED_RED * 2))    |
                   (GPIO_MODER_OUTPUT << (LED_BLUE * 2));

    /* Step 3: Push-pull output (OTYPER bit = 0 = default) */
    GPIOD_OTYPER &= ~((1U << LED_GREEN) | (1U << LED_ORANGE) |
                      (1U << LED_RED)   | (1U << LED_BLUE));

    /* Step 4: Low speed (OSPEEDR = 00 = default, ~2MHz for GPIO toggle) */

    /* Step 5: No pull-up/pull-down (PUPDR = 00 = default) */
}

int main(void) {
    gpio_init();

    while (1) {
        /* Turn on Orange LED using BSRR (atomic set — no read-modify-write needed) */
        GPIOD_BSRR = (1U << LED_ORANGE);         /* Set bit = pin HIGH */
        delay_ms(500);

        /* Turn off Orange LED using BSRR upper 16 bits */
        GPIOD_BSRR = (1U << (LED_ORANGE + 16));  /* Reset bit = pin LOW */
        delay_ms(500);
    }
}
```

### `Makefile` — Build System

```makefile
# Project: LED_Blink
# Target: STM32F407 (ARM Cortex-M4)

TARGET = led_blink
MCU    = cortex-m4
FPU    = -mfpu=fpv4-sp-d16 -mfloat-abi=hard

CC      = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy
SIZE    = arm-none-eabi-size

CFLAGS  = -mcpu=$(MCU) -mthumb $(FPU)
CFLAGS += -O0 -g3 -Wall -Wextra
CFLAGS += -ffunction-sections -fdata-sections
CFLAGS += -std=c11

LDFLAGS  = -mcpu=$(MCU) -mthumb $(FPU)
LDFLAGS += -T linker.ld
LDFLAGS += -Wl,--gc-sections          # Remove unused sections
LDFLAGS += -Wl,-Map=$(TARGET).map     # Generate map file
LDFLAGS += --specs=nosys.specs        # No OS syscalls
LDFLAGS += --specs=nano.specs         # Nano libc

SRCS = startup.c main.c
OBJS = $(SRCS:.c=.o)

all: $(TARGET).elf $(TARGET).bin

$(TARGET).elf: $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $^
	$(SIZE) $@

$(TARGET).bin: $(TARGET).elf
	$(OBJCOPY) -O binary $< $@

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

flash: $(TARGET).bin
	st-flash write $(TARGET).bin 0x08000000

clean:
	rm -f *.o *.elf *.bin *.map

.PHONY: all flash clean
```

---

## 🔨 Build & Flash

```bash
# Prerequisites: arm-none-eabi-gcc, st-flash (stlink tools)
# sudo apt install gcc-arm-none-eabi stlink-tools

# Build
make

# Output:
# arm-none-eabi-size led_blink.elf
#   text   data    bss    dec    hex filename
#    284      0      0    284    11c led_blink.elf
# Only 284 bytes! No OS, no HAL overhead.

# Flash to board (ST-Link V2 on Discovery)
make flash

# Debug with GDB
arm-none-eabi-gdb led_blink.elf
(gdb) target remote localhost:3333   # OpenOCD running in background
(gdb) monitor reset halt
(gdb) load                           # Flash the firmware
(gdb) break main
(gdb) continue
```

---

## 🧪 Exercises to Extend This Project

1. **Use BSRR for atomic toggle**: Rewrite toggle using `ODR ^= (1 << LED_ORANGE)` and observe non-atomicity issue. Then fix with BSRR.

2. **Add SysTick timer**: Replace the busy-wait `delay_ms()` with a proper SysTick-based delay. See → [[../../05_Cortex_M_Deep_Dive/SysTick/SysTick_Timer|SysTick Timer]]

3. **Button input**: Read the user button (PA0 on Discovery) and toggle the LED on press. This introduces GPIO input mode and debouncing.

4. **Interrupt-driven button**: Replace polling with EXTI interrupt on PA0. See → [[../03_Interrupt_System/Interrupt_System_Project|Project 3: Interrupt System]]

5. **Blink pattern in Flash**: Store a blink pattern array in Flash (const array) and play it back. Verifies your linker script is correct.

---

## ✅ Success Criteria

- [ ] LED blinks at ~1 Hz without any HAL or vendor library
- [ ] Binary size < 1KB
- [ ] Can pause execution with GDB and inspect GPIOD_ODR register live
- [ ] Understand every line in startup.c and main.c
- [ ] Can find the register addresses in the STM32F407 datasheet independently

---

## 🔗 Related Notes

- [[../../04_Memory_System/Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../07_ARM_C_Integration/Linker_Scripts/Linker_Script_Explained|Linker Script Explained]]
- [[../../07_ARM_C_Integration/Startup_Files/Startup_File_Anatomy|Startup File Anatomy]]
- [[../../07_ARM_C_Integration/Boot_Process/ARM_Boot_Process|ARM Boot Process]]
- [[../../06_Peripherals_BareMetal/GPIO/GPIO_Register_Programming|GPIO Register Programming]]

---

**Next Project →** [[../02_UART_Driver/UART_Driver_Project|Project 2: UART Driver]]

*← [[../MOC_Projects|Projects Overview]]*
