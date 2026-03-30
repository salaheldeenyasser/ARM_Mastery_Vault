# ARM Assembly Quick Reference

#reference #assembly #arm #cheatsheet

---

## Data Movement

| Instruction | Example | Effect |
|---|---|---|
| MOV | `MOV R0, #5` | R0 = 5 |
| MOV | `MOV R0, R1` | R0 = R1 |
| MOVW | `MOVW R0, #0x1234` | R0 = 0x1234 (16-bit imm) |
| MOVT | `MOVT R0, #0x5678` | R0[31:16] = 0x5678 |
| MVN | `MVN R0, R1` | R0 = ~R1 |
| LDR | `LDR R0, [R1]` | R0 = Mem[R1] |
| LDR | `LDR R0, [R1, #4]` | R0 = Mem[R1+4] |
| LDRB | `LDRB R0, [R1]` | R0 = (uint8_t)Mem[R1] |
| LDRH | `LDRH R0, [R1]` | R0 = (uint16_t)Mem[R1] |
| STR | `STR R0, [R1]` | Mem[R1] = R0 |
| STRB | `STRB R0, [R1]` | Mem[R1] = R0[7:0] |
| PUSH | `PUSH {R4-R7, LR}` | Push to stack |
| POP | `POP {R4-R7, PC}` | Pop from stack + return |

---

## Arithmetic

| Instruction | Example | Effect |
|---|---|---|
| ADD | `ADD R0, R1, R2` | R0 = R1 + R2 |
| ADDS | `ADDS R0, R1, R2` | R0 = R1 + R2, set flags |
| ADC | `ADC R0, R1, R2` | R0 = R1 + R2 + C |
| SUB | `SUB R0, R1, R2` | R0 = R1 - R2 |
| SUBS | `SUBS R0, R1, R2` | R0 = R1 - R2, set flags |
| MUL | `MUL R0, R1, R2` | R0 = R1 × R2 (32-bit) |
| UDIV | `UDIV R0, R1, R2` | R0 = R1 / R2 (unsigned) |
| SDIV | `SDIV R0, R1, R2` | R0 = R1 / R2 (signed) |
| RSB | `RSB R0, R1, #0` | R0 = 0 - R1 (negate) |

---

## Logic & Bitwise

| Instruction | Example | Effect |
|---|---|---|
| AND | `AND R0, R1, R2` | R0 = R1 & R2 |
| ORR | `ORR R0, R1, R2` | R0 = R1 \| R2 |
| EOR | `EOR R0, R1, R2` | R0 = R1 ^ R2 |
| BIC | `BIC R0, R1, R2` | R0 = R1 & ~R2 (bit clear) |
| LSL | `LSL R0, R1, #3` | R0 = R1 << 3 |
| LSR | `LSR R0, R1, #3` | R0 = R1 >> 3 (logical) |
| ASR | `ASR R0, R1, #3` | R0 = R1 >> 3 (arithmetic) |
| ROR | `ROR R0, R1, #3` | R0 = rotate right by 3 |

---

## Comparison (Sets Flags Only)

| Instruction | Example | Effect |
|---|---|---|
| CMP | `CMP R0, R1` | Sets flags for R0 - R1 |
| CMN | `CMN R0, R1` | Sets flags for R0 + R1 |
| TST | `TST R0, R1` | Sets flags for R0 & R1 |
| TEQ | `TEQ R0, R1` | Sets flags for R0 ^ R1 |

---

## Branches

| Instruction | Example | Effect |
|---|---|---|
| B | `B label` | Unconditional branch |
| BL | `BL func` | Branch + link (call) |
| BX | `BX LR` | Branch to register (return) |
| BLX | `BLX R0` | Branch + link to register |
| BEQ | `BEQ label` | Branch if Z=1 |
| BNE | `BNE label` | Branch if Z=0 |
| BLT | `BLT label` | Branch if signed < |
| BGT | `BGT label` | Branch if signed > |
| BLE | `BLE label` | Branch if signed <= |
| BGE | `BGE label` | Branch if signed >= |
| BLO | `BLO label` | Branch if unsigned < (C=0) |
| BHI | `BHI label` | Branch if unsigned > |

---

## Condition Codes

| Suffix | Flags | Meaning |
|---|---|---|
| EQ | Z=1 | Equal |
| NE | Z=0 | Not equal |
| CS/HS | C=1 | Carry set / unsigned higher or same |
| CC/LO | C=0 | Carry clear / unsigned lower |
| MI | N=1 | Minus / negative |
| PL | N=0 | Plus / positive |
| VS | V=1 | Overflow |
| VC | V=0 | No overflow |
| HI | C=1, Z=0 | Unsigned higher |
| LS | C=0 or Z=1 | Unsigned lower or same |
| GE | N=V | Signed >= |
| LT | N≠V | Signed < |
| GT | Z=0, N=V | Signed > |
| LE | Z=1 or N≠V | Signed <= |
| AL | (any) | Always (default) |

---

## Special Instructions

| Instruction | Effect |
|---|---|
| NOP | No operation (pipeline filler) |
| WFI | Wait For Interrupt (low power) |
| WFE | Wait For Event (low power) |
| SEV | Send Event (wakes WFE) |
| DSB | Data Synchronization Barrier |
| DMB | Data Memory Barrier |
| ISB | Instruction Synchronization Barrier |
| CPSID i | Disable IRQ (set PRIMASK) |
| CPSIE i | Enable IRQ (clear PRIMASK) |
| MRS | `MRS R0, PSP` Read special register |
| MSR | `MSR PSP, R0` Write special register |
| SVC | `SVC #0` Supervisor call |
| BKPT | `BKPT #0` Breakpoint (halts debugger) |

---

## AAPCS Register Summary

```
R0-R3  : Arguments / Return value (caller-saved)
R4-R11 : General purpose (callee-saved — you MUST preserve)
R12    : Scratch (caller-saved, used by linker veneers)
R13    : Stack Pointer (SP)
R14    : Link Register (LR) — return address
R15    : Program Counter (PC)
xPSR   : N Z C V Q T ISR_NUMBER
```

---

## Common Patterns

```asm
; Function prologue (non-leaf)
PUSH {R4-R7, LR}

; Function epilogue (non-leaf)
POP  {R4-R7, PC}

; Set a bit (bit 3 of R0)
ORR R0, R0, #(1 << 3)

; Clear a bit (bit 5 of R0)
BIC R0, R0, #(1 << 5)

; Toggle a bit (bit 2 of R0)
EOR R0, R0, #(1 << 2)

; Test if bit is set
TST R0, #(1 << 4)
BNE bit_was_set

; Atomic bit-set via LDREX/STREX
retry:
    LDREX R1, [R0]
    ORR   R1, R1, #(1 << 3)
    STREX R2, R1, [R0]
    CMP   R2, #0
    BNE   retry
```

---

*← [[../MOC_Reference|Quick Reference]]*
