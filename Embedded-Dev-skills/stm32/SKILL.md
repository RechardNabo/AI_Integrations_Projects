---
name: stm32
description: >
  STM32 microcontroller development skill for all STM32 families (F0–F7, G0/G4,
  H7, L0–L5, U5, WB/WL). Use this skill for: STM32CubeIDE setup, HAL vs LL
  driver selection, CubeMX code generation, clock configuration (RCC), GPIO,
  UART/USART, SPI, I2C, ADC, DMA, timers (TIM), PWM, CAN/FDCAN, USB, FreeRTOS
  integration, linker script customization, FLASH/OTP programming, low-power
  modes (STOP/STANDBY), and debugging with ST-Link/SWD. Triggers on: STM32,
  HAL_, LL_, CubeMX, ST-Link, NUCLEO, Discovery, stm32xxx.h.
---

# STM32 Skill

## Family Quick Reference

| Family | Core | Speed | Key Feature |
|--------|------|-------|-------------|
| STM32F0 | M0 | 48 MHz | Entry level, cheap |
| STM32F1 | M3 | 72 MHz | Blue Pill, most cloned |
| STM32F4 | M4F | 180 MHz | FPU, DSP, USB OTG |
| STM32F7 | M7 | 216 MHz | Cache, SDRAM, Ethernet |
| STM32H7 | M7 | 480 MHz | Dual-core option, high perf |
| STM32G4 | M4F | 170 MHz | HRTIM, FDCAN, Math accelerator |
| STM32L4 | M4F | 80 MHz | Ultra-low power |
| STM32U5 | M33 | 160 MHz | TrustZone, ultra-low power |
| STM32WB | M4+M0+ | 64 MHz | BLE 5.2 + 802.15.4 |

---

## HAL vs LL — When to Use Each

| Aspect | HAL | LL |
|--------|-----|----|
| Ease of use | High | Low |
| Code size | Large | Small |
| Speed | Slower (abstraction overhead) | Fast, direct register |
| Portability | High (same API across families) | Low |
| **Use for** | Prototyping, complex peripherals (USB, SDIO) | Tight loops, ISR code, UART, SPI, GPIO |

**Best practice**: Use CubeMX to generate HAL init, then switch critical paths to LL.

---

## GPIO

```c
/* HAL */
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
GPIO_PinState s = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

/* LL (faster, use in ISR or tight loops) */
LL_GPIO_SetOutputPin(GPIOA, LL_GPIO_PIN_5);
LL_GPIO_ResetOutputPin(GPIOA, LL_GPIO_PIN_5);
LL_GPIO_TogglePin(GPIOA, LL_GPIO_PIN_5);

/* Direct register (fastest) */
GPIOA->BSRR = (1U << 5);          // Set PA5
GPIOA->BSRR = (1U << (5 + 16));   // Reset PA5
GPIOA->ODR ^= (1U << 5);          // Toggle PA5 (not atomic on all families)
```

---

## UART

```c
/* HAL non-blocking with DMA */
HAL_UART_Transmit_DMA(&huart2, tx_buf, len);
HAL_UART_Receive_DMA(&huart2, rx_buf, len);

/* Callback override */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART2) {
        process_rx(rx_buf);
        HAL_UART_Receive_DMA(&huart2, rx_buf, len); // Re-arm
    }
}

/* LL polling TX */
void uart_putc(USART_TypeDef *uart, uint8_t c) {
    while (!LL_USART_IsActiveFlag_TXE(uart));
    LL_USART_TransmitData8(uart, c);
}

/* UART IDLE line detection — best for variable-length frames */
// Enable: __HAL_UART_ENABLE_IT(&huart2, UART_IT_IDLE);
// In IRQ: if (__HAL_UART_GET_FLAG(&huart2, UART_FLAG_IDLE)) { ... }
```

---

## SPI

```c
/* HAL full-duplex */
HAL_SPI_TransmitReceive(&hspi1, tx_buf, rx_buf, len, HAL_MAX_DELAY);

/* Chip select is manual — HAL does NOT drive CS */
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_RESET);
HAL_SPI_Transmit(&hspi1, tx_buf, len, 1000);
HAL_GPIO_WritePin(CS_GPIO_Port, CS_Pin, GPIO_PIN_SET);

/* LL single byte */
static inline uint8_t spi1_byte(uint8_t tx) {
    while (!LL_SPI_IsActiveFlag_TXE(SPI1));
    LL_SPI_TransmitData8(SPI1, tx);
    while (!LL_SPI_IsActiveFlag_RXNE(SPI1));
    return LL_SPI_ReceiveData8(SPI1);
}
```

---

## DMA Configuration Checklist

1. Enable DMA clock: `__HAL_RCC_DMA1_CLK_ENABLE()`
2. Configure channel/stream in CubeMX or manually set `DMA_InitTypeDef`
3. Link DMA to peripheral in peripheral init (HAL does this automatically)
4. Enable DMA interrupts for half/full transfer callbacks
5. For Cortex-M7 (F7/H7): call `SCB_CleanInvalidateDCache_by_Addr()` before/after DMA
6. For H7: DMA buffers MUST be in `SRAM1/SRAM2/DTCM` — not QSPI flash

```c
/* Cache maintenance for M7 */
SCB_CleanDCache_by_Addr((uint32_t*)tx_buf, len); // Before DMA TX
SCB_InvalidateDCache_by_Addr((uint32_t*)rx_buf, len); // After DMA RX
```

---

## Timers — PWM Output

```c
/* CubeMX: TIM3, CH1, PWM Generation Mode 1 */
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);

/* Set duty cycle: ARR = period ticks, CCR = duty ticks */
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty); // 0 to ARR

/* Frequency formula:
   f_PWM = TIM_CLK / ((PSC + 1) * (ARR + 1))
   Example: 72MHz / (72 * 1000) = 1 kHz PWM, ARR=999, PSC=71 */
```

---

## Low-Power Modes (L4 / G4 / U5)

| Mode | CPU | Peripherals | Wakeup latency | Current |
|------|-----|-------------|----------------|---------|
| Sleep | Off | On | µs | ~2 mA |
| Stop 1 | Off | RTC, LPUART, comparators | ~5 µs | ~3 µA |
| Stop 2 | Off | RTC only | ~5 µs | ~1 µA |
| Standby | Off | RTC, WKUP pins | ~50 µs | ~300 nA |
| Shutdown | Off | WKUP pins only | Reset | ~30 nA |

```c
/* Enter STOP2 mode */
HAL_PWREx_EnterSTOP2Mode(PWR_STOPENTRY_WFI);
/* After wakeup: re-init system clock (HSI active after STOP) */
SystemClock_Config();
```

---

## STM32CubeIDE / CubeMX Tips

- Generate code to **separate user/generated sections** — wrap custom code in `/* USER CODE BEGIN */ ... /* USER CODE END */` blocks
- Enable **SWV/ITM** for `printf` over SWD without UART: `ITM_SendChar()`
- Use **Live Expressions** in debug view for real-time variable watch
- Set `Heap_Size` and `Stack_Size` in `startup_stm32xxx.s`, not just in IDE
- Enable **Fault Handlers** in CubeMX system → `USE_FULL_ASSERT`

---

## Fault Handler Debug Snippet

```c
void HardFault_Handler(void) {
    __asm volatile (
        "tst lr, #4 \n"
        "ite eq \n"
        "mrseq r0, msp \n"
        "mrsne r0, psp \n"
        "ldr r1, [r0, #24] \n"  /* PC at fault */
        "bkpt #0 \n"
    );
}
/* In debugger: r0 points to stacked frame; r1 = faulting PC */
```

---

## Checklist

- [ ] Clock config matches target frequency (verify with `HAL_RCC_GetSysClockFreq()`)
- [ ] DMA IRQ priorities set lower than peripheral IRQ priorities
- [ ] `__HAL_LINKDMA()` called for all DMA-linked peripherals
- [ ] Cache maintenance for M7/H7 DMA transfers
- [ ] UART IDLE interrupt armed for variable-length packet reception
- [ ] Low-power wakeup: re-configure clocks after STOP modes
- [ ] JTAG/SWD pins not re-used as GPIO (or explicitly reconfigured)
