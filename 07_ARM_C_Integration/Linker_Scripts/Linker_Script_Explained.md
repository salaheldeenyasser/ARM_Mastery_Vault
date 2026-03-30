# Linker Script Explained

#advanced #arm #linker #memory #toolchain

> The linker script is the architectural blueprint that tells the linker: *"Here is the physical memory of this chip. Place the code here, the data there, and the stack at the top."* Without it, your firmware won't exist as a valid binary.

---

## 📖 Concept Explanation

When you compile C code, you get `.o` files (object files). The linker combines them into a final executable. But the linker needs to know:
- Where does Flash start and how big is it?
- Where does SRAM start and how big is it?
- Which section goes where?
- Where should the stack be?
- Where is the interrupt vector table?

A **linker script** (`.ld` file) answers all of these questions for your specific hardware.

---

## 🔬 Deep Dive

### Minimal Linker Script Structure

```
Memory Definition
      ↓
  MEMORY { }          ← "Here are the physical memories on this chip"
  
  SECTIONS { }        ← "Here is how to place code/data into those memories"
      ↓
Output Sections
```

### Full Annotated Linker Script (STM32F407)

```ld
/* ======================================================
   STM32F407VG Linker Script
   Flash: 1MB at 0x08000000
   SRAM1: 112KB at 0x20000000
   SRAM2: 16KB  at 0x2001C000
   ====================================================== */

/* Entry point - where execution begins */
ENTRY(Reset_Handler)

/* Highest address of the user mode stack */
/* SRAM1 ends at 0x20000000 + 112*1024 = 0x2001C000 */
_estack = 0x2001C000;   /* End of SRAM1 = initial stack pointer */

/* Minimum sizes for heap and stack */
_Min_Heap_Size  = 0x200;   /* 512 bytes */
_Min_Stack_Size = 0x400;   /* 1024 bytes */

/* ---- MEMORY REGIONS ---- */
MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K   /* rx = read/execute */
  SRAM1 (xrw) : ORIGIN = 0x20000000, LENGTH = 112K    /* xrw = execute/read/write */
  SRAM2 (xrw) : ORIGIN = 0x2001C000, LENGTH = 16K
}

/* ---- SECTIONS ---- */
SECTIONS
{
  /* ---- 1. INTERRUPT VECTOR TABLE (must be first in Flash) ---- */
  .isr_vector :
  {
    . = ALIGN(4);           /* Align to 4-byte boundary */
    KEEP(*(.isr_vector))    /* KEEP prevents dead-code elimination */
    . = ALIGN(4);
  } >FLASH                  /* Place in FLASH region */

  /* ---- 2. CODE (.text) ---- */
  .text :
  {
    . = ALIGN(4);
    *(.text)                /* All .text sections from all .o files */
    *(.text*)               /* Wildcard: .text.Reset_Handler, etc. */
    *(.glue_7)              /* ARM/Thumb interworking glue code */
    *(.glue_7t)
    *(.eh_frame)
    
    KEEP(*(.init))          /* C runtime initialization */
    KEEP(*(.fini))
    
    . = ALIGN(4);
    _etext = .;             /* Symbol: end of .text = start of .rodata */
  } >FLASH

  /* ---- 3. READ-ONLY DATA (.rodata) ---- */
  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)              /* const variables, string literals */
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH

  /* ---- 4. ARM Exception Tables (for C++ unwinding, optional) ---- */
  .ARM.extab : { *(.ARM.extab* .gnu.linkonce.armextab.*) } >FLASH
  .ARM        : { *(.ARM.exidx* .gnu.linkonce.armexidx.*) } >FLASH

  /* ---- 5. Symbols for startup code ---- */
  _sidata = LOADADDR(.data);  /* LMA of .data section (in Flash) */

  /* ---- 6. INITIALIZED DATA (.data) ---- */
  /* Stored in Flash (LMA), copied to SRAM at boot (VMA) */
  .data :
  {
    . = ALIGN(4);
    _sdata = .;             /* Symbol: start of .data in SRAM */
    *(.data)
    *(.data*)
    . = ALIGN(4);
    _edata = .;             /* Symbol: end of .data in SRAM */
  } >SRAM1 AT>FLASH         /* VMA=SRAM1, LMA=FLASH */
  /*   ^^^^^^^^^^^^ This is the key: runs in SRAM, stored in Flash */

  /* ---- 7. ZERO-INITIALIZED DATA (.bss) ---- */
  /* No storage in Flash — startup code zeroes this region */
  .bss :
  {
    . = ALIGN(4);
    _sbss = .;              /* Symbol: start of .bss */
    *(.bss)
    *(.bss*)
    *(COMMON)               /* Uninitialized globals (C "common" symbols) */
    . = ALIGN(4);
    _ebss = .;              /* Symbol: end of .bss */
  } >SRAM1

  /* ---- 8. HEAP ---- */
  ._user_heap_stack :
  {
    . = ALIGN(8);
    PROVIDE(end = .);       /* sbrk() uses 'end' as heap start */
    PROVIDE(_end = .);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >SRAM1

  /* ---- 9. REMOVE UNUSED SECTIONS ---- */
  /DISCARD/ :
  {
    libc.a(*)
    libm.a(*)
    libgcc.a(*)
  }
}
```

### How Startup Code Uses These Symbols

The symbols defined in the linker script (`_sdata`, `_edata`, `_sidata`, `_sbss`, `_ebss`) are used by the startup code to:

```c
// startup_stm32f407.c - Reset_Handler
void Reset_Handler(void) {
    /* 1. Copy .data from Flash (LMA) to SRAM (VMA) */
    uint32_t *src = &_sidata;   // LMA: where .data sits in Flash
    uint32_t *dst = &_sdata;    // VMA: where .data should be in SRAM
    while (dst < &_edata) {
        *dst++ = *src++;
    }

    /* 2. Zero-fill .bss in SRAM */
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0;
    }

    /* 3. Call main */
    main();

    /* 4. Should never reach here */
    while(1);
}
```

→ See [[../Startup_Files/Startup_File_Anatomy|Startup File Anatomy]]

---

## 💻 Code Example

### Placing a Variable in a Specific Memory Region

```c
// Place a DMA buffer in SRAM2 (for DMA2 which can only access SRAM2 on STM32F4)
__attribute__((section(".sram2"))) 
uint8_t dma_buffer[1024];

// In your linker script, add:
// .sram2 :
// {
//     *(.sram2)
// } >SRAM2

// Place a variable in CCM RAM (zero-wait-state on STM32F4)
__attribute__((section(".ccmram")))
uint32_t time_critical_buffer[256];
```

### Forcing a Function to SRAM (for faster execution or Flash programming)

```c
// Execute this function from SRAM (copied there at startup)
__attribute__((section(".ramfunc")))
void flash_program_sector(uint32_t sector, uint32_t *data, uint32_t len) {
    // Flash programming requires running from SRAM
    // because you can't read Flash while writing it
}

// In linker script:
// .ramfunc :
// {
//     *(.ramfunc)
// } >SRAM1 AT>FLASH  /* Same VMA/LMA trick as .data */
```

### Checking Symbol Addresses

```bash
# Print all symbols from ELF file
arm-none-eabi-nm firmware.elf | grep -E "_sdata|_edata|_sbss|_ebss|_sidata|_estack"

# Output:
# 2001bff8 B _ebss
# 20000000 D _sdata  
# 20000078 D _edata
# 08007890 R _sidata  ← Flash address (LMA of .data)
# 20000078 B _sbss
# 2001c000 N _estack  ← Top of SRAM1
```

---

## 🌍 Real-World Use Case

### Multi-Region Memory Layout (Real Product)

```ld
/* Production firmware with multiple memory strategies */
MEMORY {
    FLASH_BOOT   (rx)  : ORIGIN = 0x08000000, LENGTH = 32K   /* Bootloader */
    FLASH_APP    (rx)  : ORIGIN = 0x08008000, LENGTH = 992K  /* Application */
    CCM_RAM      (xrw) : ORIGIN = 0x10000000, LENGTH = 64K   /* Zero-wait RAM */
    SRAM1        (xrw) : ORIGIN = 0x20000000, LENGTH = 112K
    SRAM2        (xrw) : ORIGIN = 0x2001C000, LENGTH = 16K   /* DMA only */
}
```

This layout separates the bootloader from application, putting real-time tasks in CCM RAM and DMA buffers in SRAM2 where the DMA controller can reach them.

---

## ⚠️ Common Pitfalls

- **`_estack` wrong**: If `_estack` points past the end of SRAM, your stack will corrupt other memory or overflow. Always calculate: `SRAM_ORIGIN + SRAM_SIZE`.
- **Forgetting `KEEP`**: The linker does dead-code elimination. Your interrupt vector table will be stripped unless you wrap it with `KEEP(*(.isr_vector))`.
- **Missing `>SRAM1 AT>FLASH`**: If you use `>SRAM1` alone for `.data`, the initialized values aren't stored in Flash — they'll be zero at reset.
- **Alignment issues**: DMA controllers often require buffers aligned to 4 or 32 bytes. Use `. = ALIGN(32);` in the linker script or `__attribute__((aligned(32)))` in C.
- **Stack and heap overlap**: If your heap grows up and stack grows down and they meet — silent corruption. Add assertions in startup code to verify `_end + _Min_Stack_Size <= _estack`.

---

## 🔗 Related Notes

- [[../Startup_Files/Startup_File_Anatomy|Startup File Anatomy]]
- [[../Memory_Sections/Text_Data_BSS_Sections|.text, .data, .bss Sections]]
- [[../Boot_Process/ARM_Boot_Process|ARM Boot Process]]
- [[../../04_Memory_System/Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../04_Memory_System/Stack_vs_Heap/Stack_Memory_Model|Stack Memory Model]]

---

## 🎯 Interview Questions

1. **What is the difference between VMA and LMA?**
   *VMA (Virtual Memory Address) is where code/data executes. LMA (Load Memory Address) is where it's stored in Flash. For `.data`, LMA=Flash, VMA=SRAM — startup code copies it.*

2. **Why must the interrupt vector table be at the start of Flash?**
   *Because the Cortex-M CPU, after reset, reads the initial SP from 0x00000000 and the Reset_Handler address from 0x00000004. These addresses are aliased to the start of Flash.*

3. **What happens if you forget `KEEP(*(.isr_vector))`?**
   *The linker's garbage collector may remove the vector table as unreferenced code, producing a binary with no vector table — the CPU will jump to a garbage address on reset.*

4. **What does `>SRAM1 AT>FLASH` mean for the `.data` section?**
   *It creates a section that runs (VMA) in SRAM1 but is stored (LMA) in Flash. The startup code must copy it from Flash to SRAM at boot before any C code runs.*

---

*← [[../MOC_C_Integration|Stage 7: ARM + C Integration]]*
