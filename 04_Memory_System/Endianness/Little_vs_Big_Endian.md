# Little-Endian vs Big-Endian

#intermediate #arm #memory #endianness

> Endianness determines the byte order of multi-byte values in memory. ARM is typically little-endian. Get this wrong in a network protocol or binary file parser and you'll spend hours chasing phantom bugs.

---

## 📖 Concept Explanation

When a 32-bit value like `0x12345678` is stored in memory, which byte goes where?

- **Little-Endian**: Least significant byte at the lowest address (ARM default)
- **Big-Endian**: Most significant byte at the lowest address (network byte order)

Think of it as: which **end** of the number is at the low address?

---

## 🔬 Visual Comparison

```
Value: 0x12345678

Memory address:  0x2000  0x2001  0x2002  0x2003
                 ┌──────┬──────┬──────┬──────┐
Little-Endian:   │  78  │  56  │  34  │  12  │ ← LSB first (ARM default)
                 └──────┴──────┴──────┴──────┘
                   ↑ Lowest address = Least significant byte

                 ┌──────┬──────┬──────┬──────┐
Big-Endian:      │  12  │  34  │  56  │  78  │ ← MSB first (network order)
                 └──────┴──────┴──────┴──────┘
                   ↑ Lowest address = Most significant byte
```

---

## 💻 Code Examples

### Detecting Endianness at Runtime

```c
#include <stdint.h>
#include <stdbool.h>

bool is_little_endian(void) {
    uint32_t x = 0x00000001;
    /* If little-endian, byte at lowest address = 0x01 */
    return *((uint8_t *)&x) == 0x01;
}

/* At compile time for ARM (always little-endian in practice): */
#if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
    /* ARM Cortex-M in default configuration */
#endif
```

### The Type-Punning Endianness Trap

```c
/* WRONG: Assuming byte order when casting */
uint32_t sensor_value = 0x12345678;
uint8_t *bytes = (uint8_t *)&sensor_value;

/* On little-endian ARM: */
/* bytes[0] = 0x78 (LSB) */
/* bytes[1] = 0x56 */
/* bytes[2] = 0x34 */
/* bytes[3] = 0x12 (MSB) */

/* Code that worked on big-endian platform fails here! */
uint32_t bad_value = (bytes[0] << 24) | (bytes[1] << 16) |
                     (bytes[2] << 8)  |  bytes[3];
/* bad_value = 0x78563412 — WRONG on little-endian! */

/* CORRECT: Explicit endian conversion */
uint32_t correct = __builtin_bswap32(sensor_value);  /* = 0x78563412 */
/* Or use the portable macro: */
#define SWAP32(x) ((((x) & 0xFF000000) >> 24) | \
                   (((x) & 0x00FF0000) >>  8) | \
                   (((x) & 0x0000FF00) <<  8) | \
                   (((x) & 0x000000FF) << 24))
```

### Network Protocol Parsing (Big-Endian ↔ Little-Endian)

```c
/* IP/TCP/UDP headers use big-endian (network byte order) */
/* Received packet header: */
uint8_t raw_header[4] = {0x00, 0x50, 0x00, 0x01};  /* From network */
/* Port 0x0050 = 80 (HTTP), Source port 0x0001 */

/* WRONG on ARM (little-endian): */
uint16_t *dst_port = (uint16_t *)&raw_header[0];
/* *dst_port = 0x5000 on ARM — incorrect! */

/* CORRECT: */
uint16_t dst_port = (raw_header[0] << 8) | raw_header[1];  /* = 0x0050 = 80 */
uint16_t src_port = (raw_header[2] << 8) | raw_header[3];  /* = 0x0001 */

/* Or use CMSIS-style byte-swap macros: */
uint16_t dst_port_correct = __REV16(*(uint16_t *)&raw_header[0]);
/* __REV16: ARM instruction REV16 — reverses bytes within each 16-bit halfword */
```

### ARM Byte-Swap Instructions

```asm
; ARM Thumb-2 byte swap instructions:
REV   R0, R1    ; Reverse byte order in 32-bit word:   0x12345678 → 0x78563412
REV16 R0, R1    ; Reverse bytes in each 16-bit half:   0x12345678 → 0x34127856
REVSH R0, R1    ; Reverse bytes in low 16, sign-extend: 0x00001234 → 0x00003412
```

```c
/* C intrinsics for ARM byte swap (GCC): */
uint32_t swapped32 = __builtin_bswap32(value);
uint16_t swapped16 = __builtin_bswap16(value);

/* CMSIS equivalents: */
uint32_t s32 = __REV(value);    /* Maps to REV instruction */
uint16_t s16 = __REV16(value);  /* Maps to REV16 */
```

---

## 🌍 Real-World Use Cases

### Parsing a Binary File or Protocol

```c
/* Binary file has big-endian 32-bit magic number and 16-bit version */
/* File bytes: 0xDE 0xAD 0xBE 0xEF 0x00 0x01 */

typedef struct {
    uint32_t magic;    /* 0xDEADBEEF in big-endian file */
    uint16_t version;  /* 0x0001 */
} __attribute__((packed)) FileHeader;

void parse_header(uint8_t *buf) {
    FileHeader *h = (FileHeader *)buf;
    
    /* On little-endian ARM, multi-byte values from big-endian source are swapped */
    uint32_t magic   = __builtin_bswap32(h->magic);   /* = 0xDEADBEEF ✓ */
    uint16_t version = __builtin_bswap16(h->version);  /* = 0x0001 ✓ */
    
    if (magic != 0xDEADBEEF) {
        /* Invalid file */
    }
}
```

### SPI/I2C Multi-Byte Sensor Registers

```c
/* MPU-6050 IMU: 16-bit accelerometer in big-endian */
/* Registers: ACCEL_XOUT_H (high byte), ACCEL_XOUT_L (low byte) */

uint8_t raw[2];
I2C_ReadRegisters(MPU6050_ADDR, ACCEL_XOUT_H, raw, 2);

/* Combine bytes in the correct order (sensor is big-endian): */
int16_t accel_x = (int16_t)((raw[0] << 8) | raw[1]);

/* This is CORRECT regardless of host endianness — explicit byte assembly */
/* DO NOT do: int16_t accel_x = *(int16_t*)raw; — endianness-dependent! */
```

---

## ⚠️ Common Pitfalls

- **Casting byte array to multi-byte type**: `*(uint32_t*)byte_array` — reads 4 bytes in host byte order. On little-endian ARM, bytes are read LSB-first. If the array is big-endian data, you get the wrong value.
- **Struct overlaying a packed protocol**: `struct { uint16_t port; } header; port = *(uint16_t*)received_bytes;` — byte order bug if protocol is big-endian and target is little-endian.
- **Forgetting `__attribute__((packed))`**: Struct fields can have padding between them for alignment. Without `packed`, byte-by-byte struct overlay of a protocol buffer gives wrong field values.
- **Bit fields in structs are endianness-dependent**: The layout of bit fields in memory is implementation-defined and endianness-sensitive. Never use bit fields for hardware register overlays across platforms.

---

## 🔗 Related Notes

- [[Memory_Alignment|Memory Alignment]]
- [[Cortex_M_Memory_Map|Cortex-M Memory Map]]
- [[../../06_Peripherals_BareMetal/SPI/SPI_Protocol_and_Driver|SPI Protocol & Driver]]
- [[../../06_Peripherals_BareMetal/I2C/I2C_Protocol_and_Driver|I2C Protocol & Driver]]

---

## 🎯 Interview Questions

1. **Is ARM little-endian or big-endian?**
   *ARM is configurable via the ENDIAN bit in SCR, but almost all Cortex-M implementations are fixed little-endian. The ARMv7-M PPB (Private Peripheral Bus, 0xE0000000) is always little-endian.*

2. **You receive a 32-bit value `0x12345678` over a big-endian network protocol. How do you correctly store it on an ARM system?**
   *Use `__builtin_bswap32()` or `__REV()` to swap byte order before storing. Alternatively, assemble manually: `(buf[0]<<24) | (buf[1]<<16) | (buf[2]<<8) | buf[3]`.*

3. **Why should you avoid `*(uint32_t*)byte_ptr` for network parsing?**
   *It reads bytes in host byte order. On little-endian ARM, this interprets the first byte as the LSB. If the protocol is big-endian, the result is byte-swapped.*

4. **What does the ARM `REV` instruction do?**
   *Reverses the byte order of a 32-bit register: `REV R0, R1` makes R0's bytes = R1 reversed. Used to convert between little-endian and big-endian representations in a single cycle.*

---

*← [[../MOC_Memory|Stage 4: Memory System]]*
