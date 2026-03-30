# ARM Boot Process

#advanced #arm #cortex-m #startup #boot

> From the moment power hits the chip to the first line of your `main()`, a precise sequence unfolds. Understanding this sequence means you can debug the most fundamental system failures — and write a bootloader from scratch.

---

## 📖 Concept Explanation

When power is applied or reset is released on a Cortex-M microcontroller, the CPU doesn't magically start executing `main()`. It performs a well-defined hardware sequence, then your startup code handles software initialization, and finally C runtime is ready.

---

## 🔬 The Complete Boot Sequence

```
POWER ON / RESET
      │
      ▼
┌─────────────────────────────────────────────┐
│  STEP 1: Hardware Reset                     │
│  - All registers initialized to reset values│
│  - BOOT pins sampled to determine boot src  │
│  - Memory aliasing configured               │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STEP 2: CPU reads from 0x00000000          │
│  - Word[0] → loaded into MSP (Stack Pointer)│
│  - Word[1] → loaded into PC (Reset_Handler) │
│  [0x00000000 is aliased to Flash start]     │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STEP 3: Reset_Handler executes             │
│  (Your startup.c / startup_stm32f4xx.c)    │
│  a) Copy .data from Flash → SRAM            │
│  b) Zero-fill .bss in SRAM                 │
│  c) Init system clock (SystemInit)          │
│  d) Init C++ static constructors (if any)  │
│  e) Call main()                             │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STEP 4: main() runs                        │
│  Your application code                      │
│  - Initialize peripherals                   │
│  - Start RTOS scheduler (if applicable)     │
│  - Enter main loop                          │
└─────────────────────────────────────────────┘
```

### Step 1: BOOT Pin Decoding (STM32)

The BOOT0 pin (and BOOT1/nBOOT1 on some devices) selects what gets aliased to 0x00000000:

```
BOOT0=0 → Flash memory (0x08000000) aliased to 0x00000000
           Normal operation: your firmware boots

BOOT0=1, nBOOT1=1 → System memory (0x1FFF0000) aliased
                     ST's factory bootloader (DFU/UART download)

BOOT0=1, nBOOT1=0 → Embedded SRAM aliased
                     Execute from SRAM (used for RAM debugging)
```

### Step 2: Vector Table Fetch — The True First Instruction

```
Reset vector table layout in Flash (0x08000000):

Offset  │ Value                    │ Purpose
────────┼──────────────────────────┼─────────────────────────────
0x0000  │ 0x20020000               │ Initial MSP (top of SRAM)
0x0004  │ 0x08000141 (odd = Thumb) │ Reset_Handler address
0x0008  │ 0x08000143               │ NMI_Handler address
0x000C  │ 0x08000145               │ HardFault_Handler address
...     │ ...                      │ ...

CPU does:
  MSP ← Memory[0x00000000]   = 0x20020000   (initial stack pointer)
  PC  ← Memory[0x00000004]   = 0x08000141   (Reset_Handler, LSB=1 for Thumb)
```

### Step 3: Reset_Handler — Annotated

```c
/* startup_stm32f407.c — annotated for learning */

/* Linker-provided symbols (from linker script) */
extern uint32_t _sidata;   /* LMA of .data (in Flash) */
extern uint32_t _sdata;    /* VMA start of .data (in SRAM) */
extern uint32_t _edata;    /* VMA end of .data */
extern uint32_t _sbss;     /* Start of .bss (in SRAM) */
extern uint32_t _ebss;     /* End of .bss */
extern uint32_t _estack;   /* Initial stack pointer value */

void __attribute__((naked, noreturn)) Reset_Handler(void) {
    /*
     * WHY naked? Because we do NOT want the compiler to generate
     * a function prologue (PUSH {LR}). At reset, LR = 0xFFFFFFFF
     * and there's no valid stack yet (though MSP was set by hardware).
     */

    /* ── Phase 1: Copy .data from Flash (LMA) to SRAM (VMA) ──
     *
     * Initialized global variables like:
     *   int counter = 5;        → 5 is stored in Flash (.data LMA)
     *   const char *msg = "hi"; → pointer value stored in Flash
     *
     * After reset, SRAM has garbage. We copy the initial values.
     */
    uint32_t *src = &_sidata;   /* Flash address of .data contents */
    uint32_t *dst = &_sdata;    /* SRAM address where .data should live */

    while (dst < &_edata) {
        *dst++ = *src++;
    }

    /* ── Phase 2: Zero-fill .bss in SRAM ──
     *
     * Uninitialized globals like:
     *   static int flag;     → must start as 0 (C standard guarantee)
     *   uint8_t buffer[256]; → must start as all zeros
     *
     * These have NO storage in Flash (saves Flash space).
     * Startup code must zero them.
     */
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0U;
    }

    /* ── Phase 3: System Clock Initialization ──
     *
     * After reset, CPU runs from HSI (internal 16 MHz RC oscillator).
     * For STM32F407 at full 168 MHz, we configure:
     *   PLL source = HSE (8 MHz external crystal)
     *   PLL multiplier → 168 MHz
     *   Flash wait states → 5 (required at 168 MHz, 3.3V)
     *   AHB/APB prescalers → configure bus speeds
     */
    SystemInit();   /* ST-provided or your own clock configuration */

    /* ── Phase 4: C++ Static Constructors (if any) ──
     * Calls __libc_init_array() which invokes constructors
     * registered in .init_array section. Skip in pure C.
     */
    /* __libc_init_array(); */

    /* ── Phase 5: Jump to main() ──
     * main() should NEVER return. If it does, we spin forever.
     */
    main();

    /* If main() returns (it shouldn't), prevent undefined behavior */
    while (1) { __asm volatile("wfi"); }
}
```

---

## 💻 Code Example

### Tracing the Boot Sequence with GDB

```bash
# Connect GDB before reset
arm-none-eabi-gdb firmware.elf
(gdb) target remote localhost:3333

# Halt at the very beginning
(gdb) monitor reset halt

# Verify MSP was loaded from vector table
(gdb) info registers
sp    0x20020000  0x20020000   ← MSP loaded from [0x00000000]
pc    0x08000141  Reset_Handler + 1  ← PC at Reset_Handler (Thumb, odd)

# Step through .data copy
(gdb) break Reset_Handler
(gdb) continue
(gdb) next   # Step through the while loop

# Verify .data was copied correctly
# If "int x = 42;" is at 0x20000008, check it:
(gdb) x/xw 0x20000008
0x20000008: 0x0000002a   ← 42 in hex — correctly copied!

# Verify .bss was zeroed
(gdb) x/8xw 0x20000050   ← Somewhere in .bss region
0x20000050: 0x00000000 0x00000000 0x00000000 0x00000000  ← All zeros ✓
```

### Clock Configuration (SystemInit for STM32F407)

```c
void SystemInit(void) {
    /* 1. Enable HSE (8 MHz external oscillator) */
    RCC->CR |= RCC_CR_HSEON;
    while (!(RCC->CR & RCC_CR_HSERDY));  /* Wait for HSE stable */

    /* 2. Configure Flash wait states BEFORE increasing clock */
    FLASH->ACR = FLASH_ACR_PRFTEN |      /* Prefetch enable */
                 FLASH_ACR_ICEN   |      /* Instruction cache enable */
                 FLASH_ACR_DCEN   |      /* Data cache enable */
                 FLASH_ACR_LATENCY_5WS;  /* 5 wait states for 168 MHz */

    /* 3. Configure PLL: HSE(8MHz) × 21 / 2 = 168 MHz */
    RCC->PLLCFGR = (8  << RCC_PLLCFGR_PLLM_Pos) |  /* /8 → 1 MHz VCO input */
                   (336 << RCC_PLLCFGR_PLLN_Pos) |  /* ×336 → 336 MHz VCO */
                   (1  << RCC_PLLCFGR_PLLP_Pos) |   /* /2 → 168 MHz SYSCLK */
                   RCC_PLLCFGR_PLLSRC_HSE        |  /* Source = HSE */
                   (7  << RCC_PLLCFGR_PLLQ_Pos);    /* /7 → 48 MHz USB */

    /* 4. Enable PLL */
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY));

    /* 5. AHB prescaler = 1 (168 MHz), APB1 = /4 (42 MHz), APB2 = /2 (84 MHz) */
    RCC->CFGR |= RCC_CFGR_HPRE_DIV1 | RCC_CFGR_PPRE1_DIV4 | RCC_CFGR_PPRE2_DIV2;

    /* 6. Switch SYSCLK source to PLL */
    RCC->CFGR |= RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL);

    /* System is now running at 168 MHz */
}
```

---

## 🌍 Real-World Use Case

### Bootloader Boot Sequence

In products with a bootloader:

```
Power ON
  │
  ▼
Bootloader Reset_Handler → SystemInit → main()
  │
  ├─ Check BOOT trigger (button held? corrupted flag? UART command?)
  │     YES → Enter firmware download mode (DFU, UART, CAN)
  │     NO  ↓
  ▼
Validate Application (check CRC of Flash @ 0x08008000)
  │
  ├─ CRC FAIL → Enter emergency DFU mode
  │
  ▼
Remap vector table: SCB->VTOR = 0x08008000  ← Point to application's vectors
  │
  ▼
Load application's MSP: MSP = *(uint32_t*)0x08008000
  │
  ▼
Jump to application's Reset_Handler: PC = *(uint32_t*)0x08008004
  │
  ▼
Application runs from 0x08008000 onward
```

→ See [[../../12_Projects/05_Bootloader/Bootloader_Project|Project 5: Bootloader]]

---

## ⚠️ Common Pitfalls

- **Missing SystemInit call**: If you enable a peripheral before the clock is configured, it may run at 16 MHz instead of 168 MHz — timings are off, UART gets garbage characters.
- **Forgetting to copy .data**: If your Reset_Handler skips the .data copy loop, all initialized globals start as zero/garbage. `int x = 42;` will read as 0.
- **Startup running from wrong address**: If BOOT0 is high unexpectedly (floating pin), the bootloader runs instead of your application. Always tie BOOT0 to GND with a pull-down in your PCB.
- **Stack pointer not set before first use**: On Cortex-M, the hardware sets MSP from the vector table before Reset_Handler runs — so you CAN use the stack in Reset_Handler. On older ARM architectures (ARM7), you had to set SP manually in the first instruction.
- **SCB->VTOR not set after bootloader jumps**: If a bootloader jumps to the application without setting `SCB->VTOR` to the application's vector table address, the application's interrupt handlers will never be found — system crashes on first interrupt.

---

## 🔗 Related Notes

- [[../Startup_Files/Startup_File_Anatomy|Startup File Anatomy]]
- [[../Linker_Scripts/Linker_Script_Explained|Linker Script Explained]]
- [[../Memory_Sections/Text_Data_BSS_Sections|.text, .data, .bss Sections]]
- [[../Interrupt_Vector_Table/Vector_Table_Structure|Interrupt Vector Table Structure]]
- [[../../04_Memory_System/Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../12_Projects/05_Bootloader/Bootloader_Project|Project 5: Bootloader]]

---

## 🎯 Interview Questions

1. **What are the first two things a Cortex-M CPU does after reset?**
   *Load the initial MSP from address 0x00000000 (word 0 of the vector table), then load PC from 0x00000004 (Reset_Handler address) and jump to it.*

2. **Why must the .data section be copied at startup? Doesn't the linker put it in SRAM?**
   *The linker assigns .data a VMA in SRAM but stores its initial values in Flash (LMA). At power-on, SRAM has indeterminate values. The startup code copies the initial values from Flash to SRAM.*

3. **What does SCB->VTOR do and when do you use it?**
   *SCB->VTOR (Vector Table Offset Register) tells the CPU where the interrupt vector table is located. A bootloader uses it to redirect interrupt handling to the application's vector table after jumping to the application.*

4. **What happens if main() returns in a bare-metal system?**
   *Undefined behavior — the CPU will execute whatever garbage bytes follow main() in Flash. Best practice: spin forever (`while(1)`) after the main() call in Reset_Handler, or assert.*

---

*← [[../MOC_C_Integration|Stage 7: ARM + C Integration]]*
