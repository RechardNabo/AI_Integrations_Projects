---
name: msp430
description: >
  MSP430 ultra-low-power microcontroller development skill for the MSP430
  family (MSP430F2xx, G2xx, F5xx, FR2xx, FR5xx/6xx FRAM devices). Use this
  skill for: Code Composer Studio (CCS) and Energia setup, driverlib vs direct
  register access, clock system (DCO, ACLK, MCLK, SMCLK), GPIO, UART
  (eUSCI_A/USCI), SPI, I2C, ADC10/ADC12, Timer_A/B, low-power modes (LPM0–4),
  watchdog, and MSP430FR FRAM write. Triggers on: MSP430, CCS, LaunchPad,
  ENERGIA, eUSCI, LPM, driverlib, msp430.h, FR2355, G2553.
---

# MSP430 Skill

## Family Overview

| Series | Flash/FRAM | RAM | Speed | Highlights |
|--------|-----------|-----|-------|------------|
| G2xx (Value Line) | 0.5–16 KB Flash | 128–512 B | 16 MHz | Cheapest, G2553 most used |
| F2xx | 1–120 KB Flash | 128 B–8 KB | 16 MHz | General purpose |
| F5xx/6xx | 4–512 KB Flash | 1–66 KB | 25 MHz | USB, high performance |
| FR2xx/4xx | FRAM | 0.5–16 KB | 16 MHz | Ultra-low power FRAM |
| FR5xx/6xx | FRAM | 8–128 KB | 24 MHz | FRAM, capacitive touch, LE |

**FRAM advantage**: Write anywhere (no flash erase cycles), near-zero write current, 10¹⁵ write endurance.

---

## Clock System (FR5xx/6xx example)

```c
#include <msp430.h>

void clock_init_16MHz(void) {
    // Unlock CS registers
    CSCTL0_H = CSKEY_H;

    // Set DCO to 16 MHz
    CSCTL1 = DCOFSEL_4 | DCORSEL;  // DCO = 16 MHz

    // MCLK = SMCLK = DCO; ACLK = VLOCLK (~10 kHz, low power)
    CSCTL2 = SELA__VLOCLK | SELS__DCOCLK | SELM__DCOCLK;

    // All dividers = 1
    CSCTL3 = DIVA__1 | DIVS__1 | DIVM__1;

    // Lock CS registers
    CSCTL0_H = 0;
}
```

---

## GPIO

```c
// Direction: BIT = output, 0 = input
P1DIR |=  BIT0;         // P1.0 output (red LED on LaunchPad)
P1DIR &= ~BIT3;         // P1.3 input  (button on G2 LaunchPad)

// Output
P1OUT |=  BIT0;         // Set high
P1OUT &= ~BIT0;         // Set low
P1OUT ^=  BIT0;         // Toggle

// Pull-up/down resistor
P1REN |=  BIT3;         // Enable resistor on P1.3
P1OUT |=  BIT3;         // Pull-UP (0 = pull-down)

// Read
if (P1IN & BIT3) { /* high */ }

// Select function (peripheral vs GPIO)
P1SEL0 |= BIT4 | BIT5;  // eUSCI_B0 SCL/SDA (check device datasheet)
P1SEL1 &= ~(BIT4 | BIT5);
```

---

## Low-Power Modes

```c
/* LPM levels:
   LPM0 : MCLK off, SMCLK/ACLK on  — fastest wakeup (~6 µs)
   LPM1 : MCLK/DCO off
   LPM2 : MCLK/SMCLK/DCO off, ACLK on
   LPM3 : All clocks off except ACLK — deepest with RTC (~1 µA)
   LPM4 : All clocks off (~0.1 µA)  — GPIO/NMI wakeup only
*/

// Enter LPM3 with interrupts enabled
__bis_SR_register(LPM3_bits | GIE);
__no_operation();  // Breakpoint-friendly NOP after wakeup

// Wake from ISR: clear LPM bits in saved SR
__bic_SR_register_on_exit(LPM3_bits);

// Typical patterns:
// main() → init peripherals → __bis_SR(LPM3 | GIE) → ISR wakes → process → re-enter LPM
```

---

## UART (eUSCI_A — FR/F5 series)

```c
void uart_init_9600(void) {
    // Assumes SMCLK = 1 MHz (use 16 MHz / 1 MHz depending on your config)
    UCA0CTLW0  = UCSWRST;           // Hold in reset
    UCA0CTLW0 |= UCSSEL__SMCLK;     // SMCLK source

    // 9600 from 1 MHz: UCBRx=6, UCBRSx=0x20, UCBRFx=8, UCOS16=1
    // Use online MSP430 baud calculator for other rates
    UCA0BRW    = 6;
    UCA0MCTLW  = UCBRS0 | UCOS16 | (8 << 4);

    UCA0CTLW0 &= ~UCSWRST;          // Release reset
    UCA0IE    |= UCRXIE;            // Enable RX interrupt
}

#pragma vector=USCI_A0_VECTOR
__interrupt void USCI_A0_ISR(void) {
    switch (__even_in_range(UCA0IV, USCI_UART_UCTXCPTIFG)) {
        case USCI_UART_UCRXIFG:
            rx_buf[rx_idx++] = UCA0RXBUF;
            break;
    }
}
```

---

## Timer_A — PWM Output

```c
void timer_pwm_init(void) {
    // PWM on P1.6 (TA0.1) — G2553
    P1DIR  |= BIT6;
    P1SEL  |= BIT6;     // Timer function

    TA0CCR0  = 1000 - 1;    // Period: 1000 ticks
    TA0CCR1  = 500;          // 50% duty cycle
    TA0CCTL1 = OUTMOD_7;     // Reset/Set mode
    TA0CTL   = TASSEL_2 |    // SMCLK
               MC_1     |    // Up mode
               TACLR;        // Clear timer
}

void pwm_set_duty(uint16_t duty) {
    TA0CCR1 = duty;  // 0 to TA0CCR0
}
```

---

## ADC12 (Single Conversion)

```c
uint16_t adc_read_channel(uint8_t ch) {
    ADC12CTL0  = ADC12SHT0_2 | ADC12ON;
    ADC12CTL1  = ADC12SHP;
    ADC12CTL2 |= ADC12RES_2;            // 12-bit
    ADC12MCTL0 = ch;                     // e.g. ADC12INCH_0 for A0
    ADC12CTL0 |= ADC12ENC | ADC12SC;    // Start conversion
    while (!(ADC12IFGR0 & BIT0));        // Wait
    return ADC12MEM0;
}
```

---

## FRAM Write (FR series)

```c
// FRAM can be written like RAM — no erase needed
// For large sequential writes, disable MPU write protection temporarily

#pragma PERSISTENT
uint32_t event_counter = 0;  // Persists through power cycles

// Protect FRAM from accidental writes in production:
// Enable MPU in CCS → Project → Properties → MSP430 → Memory Options
```

---

## Watchdog

```c
// Stop watchdog immediately (first line of main() in most projects)
WDTCTL = WDTPW | WDTHOLD;

// Configure and start watchdog (interval timer mode, ~1s @ ACLK 10kHz)
WDTCTL = WDTPW | WDTTMSEL | WDTCNTCL | WDTIS_5 | WDTSSEL_1;
SFRIE1 |= WDTIE;  // Enable WDT interval interrupt

// Feed watchdog
WDTCTL = WDTPW | WDTCNTCL | (WDTCTL & 0x00FF);
```

---

## CCS Tips

- Use **Register View** during debug — MSP430 SFRs are very readable
- Enable **ULP Advisor** (Code Composer → Tools → ULP Advisor) to find LPM issues
- Set **DCO calibration**: always use `DCOCTL = CALDCO_16MHZ; BCSCTL1 = CALBC1_16MHZ;` (G2 series) — do not rely on default DCO accuracy for UART

---

## Checklist

- [ ] Watchdog stopped or configured immediately in `main()`
- [ ] Clock source matches UART baud calculation
- [ ] `P1SEL`/`P1SEL2` set correctly for peripheral pins
- [ ] LPM entered with `GIE` bit set — otherwise device sleeps forever
- [ ] ISR exits LPM with `__bic_SR_register_on_exit()` when needed
- [ ] ADC reference settled before reading (add small delay after enabling)
- [ ] FRAM persistent variables use `#pragma PERSISTENT` or placed in `.noinit`
