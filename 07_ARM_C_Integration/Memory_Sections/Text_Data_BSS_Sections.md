# .text, .data, and .bss — Memory Sections Deep Dive

#advanced #arm #memory #linker #cortex-m

> Every byte of your firmware belongs to a section. Understanding these sections tells you where your code and data live, why, and what the startup code must do before main() runs.

---

## 📖 The Three Core Sections

| Section | Contains | Storage | Access |
|---------|----------|---------|--------|
| `.text` | Executable code + read-only data | Flash (permanent) | Read/Execute |
| `.data` | Initialized global/static variables | Flash (LMA) → copied to SRAM (VMA) | Read/Write |
| `.bss` | Uninitialized global/static variables | SRAM only (no Flash storage) | Read/Write |

---

## 🔬 Deep Dive

### .text — The Code Section

Everything that goes in Flash and never changes at runtime:

```c
// All of these go in .text:
int add(int a, int b) { return a + b; }    // ← Function code
const uint32_t crc_table[256] = { ... };   // ← const arrays (.rodata subsection)
const char version[] = "v1.2.3";           // ← String literals (.rodata)
```

```
Flash Memory Layout:
┌──────────────────────────┐ 0x08000000
│  .isr_vector             │  Interrupt vector table (must be first!)
│  256 bytes (64 entries)  │
├──────────────────────────┤ 0x08000100
│  .text                   │  All compiled functions
│  ~10KB                   │  (main, Init, UART_Send, etc.)
├──────────────────────────┤ 0x08002800
│  .rodata                 │  const arrays, string literals
│  ~2KB                    │
├──────────────────────────┤ 0x08003000
│  .data (LMA)             │  Initial values for initialized globals
│  ~200 bytes              │  ← Copied to SRAM at startup
└──────────────────────────┘ 0x08003100
```

### .data — Initialized Variables

These have a split personality: stored in Flash (to survive power cycles), but must live in SRAM to be writable.

```c
// All of these go in .data:
int counter = 100;                  // Initialized to non-zero
uint8_t tx_buffer[8] = {1,2,3,4};  // Initialized array
static int call_count = 1;          // Static initialized variable
```

```
Flash                           SRAM
LMA (stored here)               VMA (runs here)
┌──────────────┐                ┌──────────────┐
│ .data (LMA)  │ ──startup──►   │ .data (VMA)  │
│              │    copies      │              │
│ 0x08003000   │                │ 0x20000000   │
│              │                │              │
│ [100]        │                │ [100]        │ counter
│ [1][2][3][4] │                │ [1][2][3][4] │ tx_buffer
└──────────────┘                └──────────────┘

_sidata = 0x08003000  ← LMA (linker script symbol)
_sdata  = 0x20000000  ← VMA start
_edata  = 0x20000208  ← VMA end
```

The startup code copies from `_sidata` to `[_sdata, _edata)`:

```c
// In Reset_Handler:
uint32_t *src = &_sidata;
uint32_t *dst = &_sdata;
while (dst < &_edata) *dst++ = *src++;
```

**If this copy is missing**: All initialized globals read as zero or garbage.

### .bss — Zero-Initialized Variables

BSS stands for "Block Started by Symbol" (historical name). No storage in Flash — startup code just zeroes the SRAM region.

```c
// All of these go in .bss:
int flag;                    // Uninitialized → guaranteed 0 by C standard
static uint8_t buffer[1024]; // Large uninitialized buffer
uint32_t error_count;        // Uninitialized global
```

```
Flash                           SRAM
No storage in Flash!            ┌──────────────┐
(just linker symbols)           │ .bss         │
                                │ 0x20000208   │
                                │              │
_sbss = 0x20000208              │ [0x00000000] │ flag (zeroed)
_ebss = 0x20000610              │ [0x00...000] │ buffer (zeroed)
                                │ [0x00000000] │ error_count (zeroed)
                                └──────────────┘
```

**Why BSS saves Flash space**: A 1KB buffer (`uint8_t buf[1024]`) in `.bss` costs:
- Flash: 0 bytes (just two symbols in the linker script)
- SRAM: 1024 bytes

The same buffer as `.data` (`uint8_t buf[1024] = {0}`) costs 1024 bytes of Flash AND 1024 bytes of SRAM.

**Always prefer `.bss` (leave uninitialized or initialize to zero) over `.data` (explicit non-zero init) unless you need a non-zero initial value.**

---

## 💻 Code Example

### Verifying Section Placement

```bash
# Check where variables end up
arm-none-eabi-nm firmware.elf | sort

# Output:
# 08000000 T _start_vector_table    ← .isr_vector in Flash (T = text)
# 08000140 T main                   ← main() in Flash
# 08003000 r crc_table              ← .rodata in Flash (r = read-only)
# 08004000 d _sidata                ← Start of .data LMA in Flash (d = data)
# 20000000 D counter                ← .data VMA in SRAM (D = initialized data)
# 20000004 D tx_buffer
# 20000008 B flag                   ← .bss in SRAM (B = BSS)
# 20000010 B error_count

# Check section sizes:
arm-none-eabi-size firmware.elf
#    text    data     bss     dec     hex filename
#   16384     256    4096   20736    5100 firmware.elf
# text=16KB Flash, data=256B Flash+SRAM, bss=4KB SRAM only
```

### Placing Variables in Specific Sections

```c
// Force a variable into a custom section
__attribute__((section(".my_section")))
uint32_t special_variable = 0xDEADBEEF;

// Force a function to SRAM (for Flash programming routines)
__attribute__((section(".ramfunc"), noinline))
void flash_write_byte(uint32_t addr, uint8_t data) {
    // This function runs from SRAM (required during Flash erase/write)
    *(volatile uint8_t*)addr = data;
}

// Place DMA buffer with alignment requirement
__attribute__((section(".dma_buffer"), aligned(32)))
uint8_t dma_tx_buffer[1024];

// Read-only lookup table (explicit .rodata)
__attribute__((section(".rodata")))
const uint8_t sin_table[256] = { /* ... */ };
```

### Checking Stack and Heap Usage at Runtime

```c
// Get current stack usage (distance from SP to bottom)
extern uint32_t _estack;      // Top of stack (from linker script)
extern uint32_t _Min_Stack_Size;

uint32_t stack_used(void) {
    uint32_t sp;
    __asm volatile("MOV %0, SP" : "=r"(sp));
    uint32_t stack_top = (uint32_t)&_estack;
    uint32_t stack_bottom = stack_top - (uint32_t)&_Min_Stack_Size;
    return stack_top - sp;       // Bytes currently used
}

// Paint stack with pattern, check watermark later
void stack_paint(void) {
    extern uint32_t _sbss, _ebss;
    uint32_t *p = &_ebss;  // Start just after BSS
    uint32_t sp;
    __asm volatile("MOV %0, SP" : "=r"(sp));
    while ((uint32_t)p < sp) {
        *p++ = 0xDEADBEEF;  // Fill unused stack with marker
    }
}

uint32_t stack_highwater_mark(void) {
    extern uint32_t _ebss;
    uint32_t *p = &_ebss;
    while (*p == 0xDEADBEEF) p++;
    return (uint32_t)p - (uint32_t)&_ebss;  // Bytes consumed from bottom
}
```

---

## 🌍 Real-World Use Case

### Memory Budget for a Real Product

```
STM32L432KC — 256KB Flash, 64KB SRAM

Target: IoT sensor node with FreeRTOS + BLE stack

Section Budget:
├── Flash (256KB total)
│   ├── .isr_vector:   0.4KB
│   ├── .text:        80KB  (FreeRTOS + BLE + App code)
│   ├── .rodata:       8KB  (Tables, strings)
│   ├── .data (LMA):   2KB  (Initialized vars stored in Flash)
│   └── Free:        165KB  (For future features / OTA double-buffer)
│
└── SRAM (64KB total)
    ├── .data (VMA):   2KB  (Initialized globals)
    ├── .bss:          4KB  (Uninitialized globals)
    ├── FreeRTOS heap: 16KB (configTOTAL_HEAP_SIZE)
    ├── Task stacks:   8KB  (4 tasks × 2KB each)
    ├── Main stack:    2KB  (MSP stack for ISRs)
    └── Free:         32KB  (Headroom)
```

This budget is defined in the linker script and verified with `arm-none-eabi-size` in the CI/CD pipeline — every build fails if Flash or SRAM exceeds the budget.

---

## ⚠️ Common Pitfalls

- **Explicit zero init wastes Flash**: `uint8_t buf[1024] = {0}` puts 1024 zero bytes in Flash (.data LMA). Just write `uint8_t buf[1024]` — it goes to .bss and startup zeros it for free.
- **`static` variables inside functions**: They go to .data (if initialized) or .bss (if not) — NOT on the stack. First call initializes them (for C++). Shared between calls. NOT thread-safe without mutex.
- **Missing `volatile` on hardware-mapped variables**: If you define `uint32_t *gpio = (uint32_t*)0x40020C14`, you must dereference it as volatile: `*(volatile uint32_t*)0x40020C14`. Or declare the pointer itself volatile.
- **Copy loop in words, not bytes**: The standard startup code copies .data 4 bytes at a time (word aligned). If .data is not 4-byte aligned, the copy is wrong. Linker scripts must enforce `. = ALIGN(4)` around .data.

---

## 🔗 Related Notes

- [[../Linker_Scripts/Linker_Script_Explained|Linker Script Explained]]
- [[../Startup_Files/Startup_File_Anatomy|Startup File Anatomy]]
- [[../Boot_Process/ARM_Boot_Process|ARM Boot Process]]
- [[../../04_Memory_System/Memory_Map/Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../04_Memory_System/Stack_vs_Heap/Stack_Memory_Model|Stack Memory Model]]

---

## 🎯 Interview Questions

1. **What's the difference between .data and .bss?**
   *.data holds initialized variables (has storage in Flash for initial values, copied to SRAM at startup). .bss holds uninitialized or zero-initialized variables (no Flash storage, startup just zeros the SRAM region).*

2. **Why does `int x = 0` go to .bss but `int x = 1` goes to .data?**
   *Compilers put explicitly zero-initialized or uninitialized variables in .bss as an optimization — they don't need Flash storage since startup zeroes .bss anyway. Non-zero initial values must be stored in Flash.*

3. **What is LMA vs VMA for the .data section?**
   *LMA (Load Memory Address) = where the initial values are stored in Flash. VMA (Virtual Memory Address) = where the variable actually lives in SRAM during execution. Startup code copies from LMA to VMA.*

4. **How would you reduce Flash usage of a firmware image?**
   *Use .bss instead of .data where possible (avoid non-zero initial values that aren't necessary). Use -Os compiler flag. Enable LTO (Link-Time Optimization). Remove unused code with -ffunction-sections + --gc-sections. Use smaller libc (nano). Replace printf with a custom minimal implementation.*

---

*← [[../MOC_C_Integration|Stage 7: ARM + C Integration]]*
