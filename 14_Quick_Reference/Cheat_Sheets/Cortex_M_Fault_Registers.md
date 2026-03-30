# Cortex-M Fault Registers — Quick Reference Card

#reference #debugging #cortex-m #cheatsheet

> Keep this open during every HardFault debug session.

---

## Fault Register Addresses

| Register | Address | Description |
|---|---|---|
| **CFSR** | `0xE000ED28` | Configurable Fault Status Register |
| **HFSR** | `0xE000ED2C` | HardFault Status Register |
| **DFSR** | `0xE000ED30` | Debug Fault Status Register |
| **MMFAR** | `0xE000ED34` | MemManage Fault Address Register |
| **BFAR** | `0xE000ED38` | BusFault Address Register |
| **AFSR** | `0xE000ED3C` | Auxiliary Fault Status Register |

### GDB Quick-Read

```bash
(gdb) x/xw 0xE000ED28   # CFSR
(gdb) x/xw 0xE000ED2C   # HFSR
(gdb) x/xw 0xE000ED34   # MMFAR (check MMARVALID first)
(gdb) x/xw 0xE000ED38   # BFAR  (check BFARVALID first)
```

---

## CFSR — Configurable Fault Status Register (0xE000ED28)

### MMFSR — Bits [7:0] — MemManage Fault

| Bit | Name | Meaning | Common Cause |
|-----|------|---------|-------------|
| 0 | IACCVIOL | Instruction access violation | Execute from MPU-protected region |
| 1 | DACCVIOL | Data access violation | Read/write to MPU-protected region |
| 3 | MUNSTKERR | Fault on exception return unstack | Stack pointer corrupted on return |
| 4 | MSTKERR | Fault on exception entry stack | Stack overflow at exception entry |
| 5 | MLSPERR | FPU lazy state fault | FPU issue on exception entry |
| **7** | **MMARVALID** | **MMFAR is valid** | **Read MMFAR for the address** |

### BFSR — Bits [15:8] — Bus Fault

| Bit | Name | Meaning | Common Cause |
|-----|------|---------|-------------|
| 8 | IBUSERR | Instruction fetch bus error | Fetch from non-existent address |
| 9 | PRECISERR | Precise data bus error | Access to invalid address (BFAR valid) |
| 10 | IMPRECISERR | Imprecise data bus error | Write-buffer delayed fault (BFAR not valid) |
| 11 | UNSTKERR | Bus fault on unstack | Stack corrupted before exception return |
| 12 | STKERR | Bus fault on stack | SP points to invalid memory |
| 13 | LSPERR | FPU lazy stacking bus fault | FPU + bus issue |
| **15** | **BFARVALID** | **BFAR is valid** | **Read BFAR for the address** |

### UFSR — Bits [31:16] — Usage Fault

| Bit | Name | Meaning | Common Cause |
|-----|------|---------|-------------|
| 16 | UNDEFINSTR | Undefined instruction | Corrupted PC, wrong ISA mode |
| 17 | INVSTATE | Invalid EPSR state | T-bit cleared in xPSR |
| 18 | INVPC | Invalid PC on exception return | EXC_RETURN value corrupted |
| 19 | NOCP | No coprocessor | FPU not enabled (CPACR register) |
| 24 | UNALIGNED | Unaligned access | Unaligned access with UNALIGN_TRP set |
| 25 | DIVBYZERO | Division by zero | DIV_0_TRP enabled and UDIV/SDIV by 0 |

---

## HFSR — HardFault Status Register (0xE000ED2C)

| Bit | Name | Meaning |
|-----|------|---------|
| 1 | VECTTBL | Vector table read error on exception entry |
| 30 | FORCED | HardFault caused by escalated configurable fault |
| 31 | DEBUGEVT | Breakpoint/watchpoint while debug disabled |

> If **FORCED = 1**: A configurable fault (MemManage/Bus/Usage) escalated because it fired inside a fault handler or was disabled. Check CFSR for the actual reason.

---

## Exception Return Values (EXC_RETURN in LR)

| Value | Meaning |
|-------|---------|
| `0xFFFFFFF1` | Return to Handler mode, use MSP, no FPU |
| `0xFFFFFFF9` | Return to Thread mode, use MSP, no FPU |
| `0xFFFFFFFD` | Return to Thread mode, use PSP, no FPU |
| `0xFFFFFFE1` | Return to Handler mode, use MSP, with FPU |
| `0xFFFFFFE9` | Return to Thread mode, use MSP, with FPU |
| `0xFFFFFFED` | Return to Thread mode, use PSP, with FPU |

---

## Stacked Frame on Exception Entry

```
If PSP was active (bit 2 of EXC_RETURN = 1):
   PSP → [R0] [R1] [R2] [R3] [R12] [LR] [PC] [xPSR]

Extract stacked PC:
  stacked_pc = ((uint32_t *)psp)[6]

GDB:
  (gdb) x/8xw $psp    ← or $msp depending on EXC_RETURN
  Index [6] = stacked PC = address of faulting instruction
```

---

## System Control Block Key Registers

| Register | Address | Purpose |
|---|---|---|
| SCB->CPUID | 0xE000ED00 | CPU ID (read MCU type) |
| SCB->ICSR | 0xE000ED04 | Interrupt Control State |
| SCB->VTOR | 0xE000ED08 | Vector Table Offset |
| SCB->AIRCR | 0xE000ED0C | Reset + Priority Grouping |
| SCB->SCR | 0xE000ED10 | Sleep Control |
| SCB->CCR | 0xE000ED14 | Config (UNALIGN_TRP, DIV_0_TRP) |
| SCB->SHPR[0] | 0xE000ED18 | System Handler Priority (MemManage, etc.) |
| SCB->SHCSR | 0xE000ED24 | System Handler Control |
| SCB->CFSR | 0xE000ED28 | ← Fault status |
| SCB->HFSR | 0xE000ED2C | ← HardFault status |

---

## HardFault Debug Flowchart

```
HardFault fires
      │
      ▼
Read LR (EXC_RETURN)
      │
      ├─ Bit 2 = 0 → Use MSP to read stacked frame
      └─ Bit 2 = 1 → Use PSP to read stacked frame
              │
              ▼
      stacked_pc = frame[6] → SOURCE LINE OF CRASH
              │
              ▼
      Read HFSR (0xE000ED2C)
              │
              ├─ FORCED=1 → Read CFSR (0xE000ED28)
              │      │
              │      ├─ MMFSR bits set → Memory protection violation
              │      │      MMARVALID=1 → read MMFAR for address
              │      │
              │      ├─ BFSR bits set → Bus error
              │      │      BFARVALID=1 → read BFAR for address
              │      │
              │      └─ UFSR bits set → Usage fault
              │             UNDEFINSTR=1 → Bad instruction / wrong ISA
              │             NOCP=1 → FPU not enabled
              │             DIVBYZERO=1 → Division by zero
              │
              └─ VECTTBL=1 → Vector table read failed (wrong VTOR)
```

---

*← [[../MOC_Reference|Quick Reference]] | [[../../11_Debugging_Tooling/Hard_Fault_Debug/Debugging_Hard_Faults|Hard Fault Debug Guide]]*
