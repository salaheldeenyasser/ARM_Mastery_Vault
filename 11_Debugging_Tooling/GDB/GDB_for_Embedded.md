# GDB for Embedded Systems

#intermediate #advanced #debugging #arm #tools

> GDB is your window into a running ARM system. With OpenOCD as the bridge, you can halt any Cortex-M, inspect every register, modify memory, and step through assembly in real time.

---

## 📖 The GDB + OpenOCD Stack

```
Your PC
  │
  │  arm-none-eabi-gdb (GDB client)
  │      communicates via TCP port 3333
  │
  ▼
  OpenOCD (debug server — translates GDB protocol to debug hardware protocol)
  │
  │  JTAG or SWD protocol
  │
  ▼
  Debug Probe (ST-Link, J-Link, CMSIS-DAP)
  │
  │  Physical wires (SWDIO, SWDCLK, GND, optionally VCC)
  │
  ▼
  Target MCU (STM32F407 running your firmware)
```

---

## 🔬 Starting a Debug Session

### OpenOCD (run first, in a separate terminal)

```bash
# For STM32F407 Discovery with ST-Link V2:
openocd \
  -f interface/stlink.cfg \
  -f target/stm32f4x.cfg

# Output shows:
# Info : STLINK V2J39S7 (API v2) VID:PID 0483:3748
# Info : clock speed 2000 kHz
# Info : SWD DPIDR 0x2ba01477
# Info : stm32f4x.cpu: hardware has 6 breakpoints, 4 watchpoints
# Info : Listening on port 3333 for gdb connections  ← Ready!
```

### GDB (connect to OpenOCD)

```bash
arm-none-eabi-gdb firmware.elf

# Inside GDB:
(gdb) target remote localhost:3333    # Connect to OpenOCD
(gdb) monitor reset halt              # Reset target and halt at reset vector
(gdb) load                            # Flash the firmware
(gdb) break main                      # Set breakpoint at main()
(gdb) continue                        # Run until breakpoint
```

---

## 💻 Essential GDB Commands

### Execution Control

```bash
(gdb) continue          # c — run until breakpoint or halt
(gdb) next              # n — next line (step OVER function calls)
(gdb) step              # s — step INTO function calls
(gdb) nexti             # ni — next assembly instruction (over)
(gdb) stepi             # si — next assembly instruction (into)
(gdb) finish            # run until current function returns
(gdb) until 42          # run until line 42
(gdb) return 0          # force function to return with value 0

# Halt a running target:
(gdb) Ctrl+C            # Send SIGINT → target halts
```

### Breakpoints & Watchpoints

```bash
(gdb) break main              # Breakpoint at function
(gdb) break main.c:42         # Breakpoint at line 42 of main.c
(gdb) break *0x08001234       # Breakpoint at memory address
(gdb) info breakpoints        # List all breakpoints
(gdb) delete 2                # Delete breakpoint #2
(gdb) disable 1               # Disable without deleting
(gdb) enable 1                # Re-enable

# Watchpoints — break when a variable changes:
(gdb) watch my_var            # Break on write
(gdb) rwatch my_var           # Break on read
(gdb) awatch my_var           # Break on read or write

# Conditional breakpoint:
(gdb) break loop.c:15 if counter > 100
```

### Inspecting State

```bash
# Registers
(gdb) info registers          # All registers
(gdb) info registers r0 r1    # Specific registers
(gdb) print $pc               # Print register by name
(gdb) set $r0 = 0x42          # Write to register

# Memory — x/[count][format][size] address
# Format: x=hex, d=decimal, u=unsigned, o=octal, t=binary, s=string, i=instruction
# Size:   b=byte, h=halfword, w=word(32-bit), g=giant(64-bit)
(gdb) x/4xw 0x20000000       # 4 words in hex at SRAM start
(gdb) x/1xw 0x40020014       # GPIOA_ODR register
(gdb) x/16xb 0x08000000      # 16 bytes (vector table start)
(gdb) x/10i $pc              # Disassemble 10 instructions from PC
(gdb) x/s 0x08004100        # Print null-terminated string at address

# Variables
(gdb) print my_variable       # Print a C variable
(gdb) print/x counter         # Print in hex
(gdb) print *ptr              # Dereference pointer
(gdb) print arr[5]            # Array element
(gdb) display my_var          # Auto-print on every step

# Stack
(gdb) backtrace               # bt — call stack
(gdb) frame 2                 # Switch to stack frame 2
(gdb) info locals             # Local variables in current frame
```

### Peripheral Register Inspection

```bash
# Read the GPIOD Output Data Register (ODR):
(gdb) x/1xw 0x40020C14       # GPIOD_ODR at base+0x14

# Read NVIC enable register (which IRQs are enabled?):
(gdb) x/1xw 0xE000E100       # NVIC_ISER0

# Read the CFSR (fault status after HardFault):
(gdb) x/1xw 0xE000ED28       # SCB->CFSR

# Read current task's stack (in FreeRTOS):
(gdb) print pxCurrentTCB->pxTopOfStack
(gdb) x/20xw pxCurrentTCB->pxTopOfStack
```

### Flashing & Memory Modification

```bash
(gdb) load                    # Load ELF file into Flash
(gdb) monitor flash write_image firmware.bin 0x08000000 bin
(gdb) monitor flash erase_sector 0 0 0   # Erase sector

# Write memory at runtime:
(gdb) set {uint32_t}0x20000004 = 0xDEADBEEF
(gdb) set my_global_var = 42
```

---

## 🔬 Useful GDB Workflows

### Post-Mortem HardFault Analysis

```bash
# Target has crashed in HardFault_Handler (breakpoint set there)
(gdb) info registers
# Note: pc is inside HardFault_Handler — not the crash site!

# Determine which stack was active (check LR = EXC_RETURN):
(gdb) print/x $lr
# 0xFFFFFFF9 → MSP was active (crashed in thread/handler with MSP)
# 0xFFFFFFFD → PSP was active (RTOS task crashed)

# Read the stacked PC (8 registers pushed, PC is 7th = SP+24):
(gdb) x/8xw $msp    # or $psp depending on EXC_RETURN
# [0]=R0  [1]=R1  [2]=R2  [3]=R3
# [4]=R12 [5]=LR  [6]=PC  [7]=xPSR
#                     ↑ This is where the crash happened!

(gdb) info line *0x08001234   # What source line is the crash PC?
(gdb) list *0x08001234        # Show source around crash PC
```

### Analyzing RTOS Tasks

```bash
# FreeRTOS task inspection (with debug symbols from freertos.elf):
(gdb) print pxCurrentTCB->pcTaskName   # Current task name
(gdb) print uxCurrentNumberOfTasks      # How many tasks exist?

# Walk the task list:
(gdb) print ((TCB_t*)pxReadyTasksLists[4].pxIndex)->pcTaskName
```

### Setting a GDB Init Script

Create `.gdbinit` in your project directory:
```
# .gdbinit — auto-loaded by GDB
target remote localhost:3333
monitor reset halt
load
break HardFault_Handler
break main
```

Then just run `arm-none-eabi-gdb firmware.elf` and everything connects automatically.

---

## 🌍 Real-World Use Case

### Debugging a Sporadic Peripheral Failure

```bash
# Problem: SPI transfer randomly fails after ~100 successful transfers
# Hypothesis: DMA counter or flag not properly reset

# Set a watchpoint on the failure counter:
(gdb) watch spi_error_count

# Run:
(gdb) continue

# When watchpoint triggers:
# Hardware watchpoint 1: spi_error_count
# Old value = 0
# New value = 1
# 0x08003456 in spi_dma_complete_callback (...)

# Now inspect the SPI status register at the moment of failure:
(gdb) x/1xw 0x40013000     # SPI1 base address
(gdb) x/1xw 0x40013008     # SPI1_SR (status register)
# 0x40013008: 0x00000041   → bit 0 = RXNE (data available), bit 6 = BSY!
# BSY flag still set when we tried to start a new transfer → root cause found!
```

---

## ⚠️ Common Pitfalls

- **GDB and OpenOCD version mismatch**: Use the GDB and OpenOCD that come with the same toolchain. Mixing versions causes protocol errors.
- **Breakpoints limited**: Cortex-M has only 4–6 hardware breakpoints. If you set more, GDB silently converts them to software breakpoints (replaces opcode with BKPT) — won't work in Flash-execute-only or ROM.
- **Optimizer removes variables**: At `-O2`, variables may be in registers and not visible to GDB. Use `-O0` or `-Og` for debugging builds.
- **`load` vs `monitor reset halt`**: `load` flashes the firmware but doesn't reset. Always `monitor reset halt` after `load` to start fresh.
- **Semihosting must be enabled in OpenOCD**: If you use printf via semihosting, add `monitor arm semihosting enable` in GDB before `continue`.

---

## 🔗 Related Notes

- [[OpenOCD_Setup|OpenOCD Setup & Configuration]]
- [[JTAG_vs_SWD|JTAG vs SWD]]
- [[../Hard_Fault_Debug/Debugging_Hard_Faults|Debugging Hard Faults]]
- [[../Reading_Registers/Live_Register_Inspection|Live Register Inspection]]

---

## 🎯 Interview Questions

1. **How do you find the source line that caused a HardFault without a debugger connected at the time of crash?**
   *Save the stacked PC (sp+24 bytes when MSP was active) in your HardFault handler to flash or RAM, reset the system, then use GDB `info line *0x<saved_pc>` to map it to source.*

2. **What's the difference between a hardware and software breakpoint in embedded GDB?**
   *Hardware breakpoints use the core's debug comparators (4-6 on Cortex-M) and work anywhere including Flash. Software breakpoints replace the instruction opcode with BKPT — only work in RAM (writable memory).*

3. **How do you inspect a specific peripheral register in GDB?**
   *Use `x/1xw <register_address>`. Find register addresses from the reference manual or CMSIS header. Example: `x/1xw 0xE000E100` reads NVIC_ISER0.*

---

*← [[../MOC_Debugging|Stage 11: Debugging & Tooling]]*
