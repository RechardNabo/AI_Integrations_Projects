---
name: pic32
description: >
  PIC32 microcontroller development skill covering PIC32MX and PIC32MZ families.
  Use this skill for any PIC32 task: MPLAB X project setup, XC32 compiler options,
  peripheral library (PLIB) vs Harmony v3, SFR register access, interrupt config
  (multi-vector vs single-vector), DMA, USB, Ethernet, CAN, UART/SPI/I2C,
  config bits (DEVCFG), boot sequence, linker script customization, and debugging
  with PICkit or ICD. Triggers on: XC32, MPLAB, PIC32, DEVCFG, SFR, Harmony.
---

# PIC32 Skill

## Family Overview

| Family | Core | Max Speed | Flash | RAM | Highlights |
|--------|------|-----------|-------|-----|------------|
| PIC32MX1/2 | M4K | 50 MHz | 32–256 KB | 8–64 KB | Low-cost, USB |
| PIC32MX3/4 | M4K | 80 MHz | 256–512 KB | 32–128 KB | USB, Ethernet |
| PIC32MX5/6/7 | M4K | 80 MHz | 512 KB | 128 KB | USB, CAN, Ethernet |
| PIC32MZ EF | M-class | 252 MHz | 1–2 MB | 512 KB | FPU, Crypto, Hi-speed USB |
| PIC32MZ DA | M-class | 200 MHz | 1–2 MB | 32 MB DRAM | DRAM, Graphics |

---

## Config Bits (DEVCFG) — Critical Setup

```c
// PIC32MX example — always at top of main.c or in a dedicated config.c
#pragma config FNOSC    = FRCPLL    // FRC + PLL
#pragma config POSCMOD  = XT        // XT crystal mode (or HS, EC)
#pragma config FPLLIDIV = DIV_2     // PLL input divider: 8MHz/2 = 4MHz
#pragma config FPLLMUL  = MUL_20    // PLL multiplier: 4*20 = 80MHz
#pragma config FPLLODIV = DIV_1     // PLL output divider
#pragma config FPBDIV   = DIV_1     // Peripheral Bus = SYSCLK
#pragma config FWDTEN   = OFF       // Watchdog disabled (enable in production!)
#pragma config ICESEL   = ICS_PGx1  // Debug pins PGx1
#pragma config JTAGEN   = OFF       // Disable JTAG (free up pins)
#pragma config BWP      = OFF       // Boot Flash write protect OFF
#pragma config CP       = OFF       // Code protect OFF
```

---

## SFR Register Access

```c
#include <xc.h>

// Direct SFR access via xc.h macros
// SET/CLR/INV atomic registers avoid read-modify-write issues
LATBSET = (1 << 5);   // Set RB5 high (atomic)
LATBCLR = (1 << 5);   // Set RB5 low  (atomic)
LATBINV = (1 << 5);   // Toggle RB5   (atomic)

// TRIS: 0 = output, 1 = input
TRISBCLR = (1 << 5);  // RB5 as output

// Analog: disable analog on digital pins
ANSELB   = 0x0000;    // All PORTB pins digital

// Input change notification (PIC32MX)
CNPUBbits.CNPUB5 = 1; // Enable pull-up on RB5
```

---

## Interrupt Configuration (Multi-Vector Mode)

```c
// Enable multi-vector interrupts (recommended)
INTCONSET = _INTCON_MVEC_MASK;
__builtin_enable_interrupts();

// ISR syntax for XC32
void __ISR(_UART1_RX_VECTOR, IPL4SOFT) UART1_RX_ISR(void) {
    uint8_t data = U1RXREG;
    IFS0CLR = _IFS0_U1RXIF_MASK;  // Clear interrupt flag
}

// Configure UART1 interrupt
IPC6bits.U1IP = 4;   // Priority 4
IPC6bits.U1IS = 0;   // Sub-priority 0
IEC0bits.U1RXIE = 1; // Enable RX interrupt

// IPL levels: IPL1 (lowest) to IPL7 (highest, non-maskable at IPL7)
// Use IPL4SOFT for most interrupts — allows nesting
// Use IPL7SRS for critical, non-nestable ISRs
```

---

## UART Driver

```c
void uart1_init(uint32_t baud, uint32_t pb_clk) {
    U1MODEbits.ON   = 0;
    U1BRG           = (pb_clk / (16 * baud)) - 1;
    U1MODEbits.PDSEL = 0b00; // 8 bit, no parity
    U1MODEbits.STSEL = 0;    // 1 stop bit
    U1STAbits.UTXEN  = 1;
    U1STAbits.URXEN  = 1;
    U1MODEbits.ON    = 1;
}

void uart1_putc(uint8_t c) {
    while (U1STAbits.UTXBF);  // Wait if TX buffer full
    U1TXREG = c;
}

uint8_t uart1_getc(void) {
    while (!U1STAbits.URXDA); // Wait for data
    return (uint8_t)U1RXREG;
}
```

---

## SPI Driver

```c
void spi2_init(void) {
    SPI2CON = 0;                   // Reset
    uint32_t dummy = SPI2BUF;     // Clear RX buffer
    SPI2BRG = (PB_CLK / (2 * SPI_FREQ)) - 1;
    SPI2CONbits.MSTEN = 1;        // Master mode
    SPI2CONbits.CKP   = 0;        // Clock idle low (Mode 0)
    SPI2CONbits.CKE   = 1;        // Data on falling edge
    SPI2CONbits.MODE16 = 0;       // 8-bit
    SPI2CONbits.ON    = 1;
}

uint8_t spi2_transfer(uint8_t tx) {
    SPI2BUF = tx;
    while (!SPI2STATbits.SPIRBF);
    return (uint8_t)SPI2BUF;
}
```

---

## DMA (PIC32MZ / MX with DMA module)

```c
// DMA channel 0: UART2 TX
void dma_uart_tx_init(void) {
    DCH0CON  = 0;
    DCH0ECON = (_UART2_TX_VECTOR << 8) | (1 << 4); // Cell transfer event
    DCH0SSA  = KVA_TO_PA(tx_buffer);   // Source: RAM buffer
    DCH0DSA  = KVA_TO_PA(&U2TXREG);   // Dest: UART TX reg
    DCH0SSIZ = sizeof(tx_buffer);
    DCH0DSIZ = 1;
    DCH0CSIZ = 1;                       // 1 byte per cell
    DCH0CONbits.CHAEN = 0;             // No auto-enable
    DCH0CONbits.CHPRI = 3;             // Highest DMA priority
    DCH0CONbits.ON    = 1;
}
// KVA_TO_PA macro converts virtual to physical address for DMA
#define KVA_TO_PA(v)  ((uint32_t)(v) & 0x1FFFFFFF)
```

---

## MPLAB Harmony v3 Quick Notes

- Use **MCC (MPLAB Code Configurator)** or **Harmony Configurator** to generate init code
- Generated files live in `src/config/<config_name>/`
- Never edit generated files directly — re-generate after config changes
- Main entry: `SYS_Initialize()` → `SYS_Tasks()` (superloop or RTOS)
- Harmony uses a **driver model**: `DRV_USART_Open()`, `DRV_USART_WriteBuffer()`, etc.
- Enable `SYS_DEBUG` and `SYS_CONSOLE` for UART debug output during development

---

## Debugging Checklist

- [ ] Config bits verified (wrong PLL = wrong UART baud, no boot)
- [ ] `JTAGEN = OFF` if using PGx pins for other functions
- [ ] `KVA_TO_PA()` applied to all DMA source/destination addresses
- [ ] Peripheral bus clock (`PBCLK`) matches `U1BRG` calculation
- [ ] Cache coherency handled for DMA (PIC32MZ): `__builtin_dcache_writeback_inv(buf, len)`
- [ ] Interrupt priority matches ISR IPL declaration exactly
