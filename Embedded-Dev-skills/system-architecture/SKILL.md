---
name: system-architecture
description: >
  Embedded system architecture design: memory maps, boot sequences, RTOS selection
  and configuration, peripheral bus topology, power domains, and inter-task
  communication. Use this skill whenever the user is designing or reviewing a
  system-level architecture for any MCU/MPU target — including memory layout,
  clock trees, DMA channel planning, RTOS task design, IPC mechanisms (queues,
  semaphores, event flags), bootloader design, or firmware update strategies.
  Also triggers on: layered architecture, HAL design, driver model, startup code,
  linker script design, or memory-mapped peripheral strategy.
---

# System Architecture Skill

## Layered Firmware Architecture

```
┌─────────────────────────────────────────┐
│           Application Layer             │  Business logic, state machines
├─────────────────────────────────────────┤
│           Middleware Layer              │  Protocol stacks, RTOS services,
│   (FreeRTOS / Zephyr / bare-metal)      │  filesystems, command parsers
├─────────────────────────────────────────┤
│        Hardware Abstraction Layer       │  Board-specific pin/clock/bus config
├─────────────────────────────────────────┤
│          Peripheral Driver Layer        │  UART, SPI, I2C, ADC, Timer drivers
├─────────────────────────────────────────┤
│         Chip Support Package (CSP)      │  Vendor HAL / CMSIS / datasheet regs
├─────────────────────────────────────────┤
│              Hardware                   │  MCU silicon, board
└─────────────────────────────────────────┘
```

**Rule**: Upper layers call downward only. Drivers use callbacks or event queues to signal upward — never direct function calls into application code.

---

## Memory Map Planning

### Typical Cortex-M Memory Layout

```
0x00000000 - 0x1FFFFFFF   Code region      (Flash, typically aliased)
0x20000000 - 0x3FFFFFFF   SRAM region
0x40000000 - 0x5FFFFFFF   Peripheral region (APB/AHB registers)
0x60000000 - 0x9FFFFFFF   External RAM
0xA0000000 - 0xBFFFFFFF   External device (FSMC/FMC)
0xE0000000 - 0xFFFFFFFF   System (SysTick, NVIC, ITM, DWT)
```

### Linker Script Sections (GCC)

```ld
MEMORY {
    FLASH  (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    RAM    (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    CCMRAM (rwx) : ORIGIN = 0x10000000, LENGTH = 64K   /* STM32 CCM */
}

SECTIONS {
    .vectors : { KEEP(*(.isr_vector)) } > FLASH
    .text    : { *(.text*) *(.rodata*) } > FLASH
    .data    : { *(.data*) } > RAM AT > FLASH  /* LMA in flash, VMA in RAM */
    .bss     : { *(.bss*) *(COMMON) } > RAM
    .noinit  : { *(.noinit*) } > RAM            /* never zero-initialized */
    .heap    : { . = ALIGN(8); _heap_start = .; . += HEAP_SIZE; } > RAM
    .stack   : { . = ALIGN(8); . += STACK_SIZE; _stack_top = .; } > RAM
}
```

---

## RTOS Task Design

### Task Sizing Guidelines

| Task Priority | Role | Stack | Period |
|--------------|------|-------|--------|
| Highest | Safety monitor / watchdog feed | 256 B | 1 ms |
| High | Comms RX/TX | 512–1K | Event-driven |
| Medium | Sensor sampling | 512 B | 10–100 ms |
| Low | Data processing | 1–4K | 100 ms+ |
| Idle | LED blink, diagnostics | 256 B | 500 ms+ |

### FreeRTOS Boilerplate

```c
/* Task handle */
static TaskHandle_t xSensorTask = NULL;

/* Task function */
static void vSensorTask(void *pvParams) {
    TickType_t xLastWake = xTaskGetTickCount();
    for (;;) {
        sensor_read();
        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(10));
    }
}

/* Queue for ISR -> Task communication */
static QueueHandle_t xUartQ = NULL;

void app_init(void) {
    xUartQ = xQueueCreate(32, sizeof(uint8_t));
    xTaskCreate(vSensorTask, "Sensor", 512, NULL, 3, &xSensorTask);
    vTaskStartScheduler();
}

/* Sending from ISR */
void UART_IRQHandler(void) {
    uint8_t byte = UART->DR;
    BaseType_t xHighPrio = pdFALSE;
    xQueueSendFromISR(xUartQ, &byte, &xHighPrio);
    portYIELD_FROM_ISR(xHighPrio);
}
```

### IPC Selection Guide

| Mechanism | Use Case | Notes |
|-----------|----------|-------|
| Queue | Producer → Consumer data stream | Copies data; safe across priorities |
| Binary Semaphore | ISR → task trigger | Signal only, no data |
| Mutex | Shared resource (SPI bus, UART) | Priority inheritance supported |
| Event Group | Multiple tasks wait on multiple events | Up to 24 bits |
| Stream Buffer | High-throughput byte stream | FreeRTOS 10+ |
| Direct Notification | Single task notification | Fastest, replaces semaphore in most cases |

---

## Clock Tree Design

```
External Crystal (HSE)
    │
    ├── PLL (multiply / divide)
    │       ├── SYSCLK → AHB → APB1 (slow peripherals)
    │       │                → APB2 (fast peripherals)
    │       ├── USB PLL (48 MHz exact)
    │       └── I2S PLL (audio)
    │
    └── RTC Clock (LSE 32.768 kHz)

Rules:
- APB1 ≤ SYSCLK/2 for most STM32
- Peripherals on APB2 can run at full SYSCLK
- Always configure clocks BEFORE enabling peripheral clocks
- Measure actual frequency with MCO output pin during bringup
```

---

## DMA Planning

```
DMA Request → Channel → Stream → Memory

Best practices:
1. Assign DMA channels from datasheet DMA request table — conflicts cause silent failures.
2. Use circular mode for continuous ADC / UART RX.
3. Use double-buffer mode to process one buffer while DMA fills the other.
4. Enable half-transfer + transfer-complete interrupts for double-buffer.
5. Cache maintenance required on Cortex-M7 (SCB_CleanInvalidateDCache).
6. Peripheral-to-memory: set peripheral as source, increment memory, fixed peripheral address.
```

---

## Bootloader Architecture

```
┌─────────────────────────────────────┐  Flash start (e.g. 0x08000000)
│  Bootloader (16–32 KB)              │
│  - Checks update flag in NOINIT RAM │
│  - Validates app CRC/signature      │
│  - Jumps to application             │
├─────────────────────────────────────┤  App start (e.g. 0x08008000)
│  Application                        │
│  - Sets update flag + resets        │
│  - Or just runs                     │
├─────────────────────────────────────┤
│  Firmware image (download slot)     │
└─────────────────────────────────────┘

Jump-to-app pattern (Cortex-M):
```

```c
typedef void (*app_entry_t)(void);

void bootloader_jump_to_app(uint32_t app_addr) {
    /* Stack pointer is at app_addr + 0, reset vector at app_addr + 4 */
    uint32_t sp     = *(volatile uint32_t *)(app_addr);
    uint32_t pc     = *(volatile uint32_t *)(app_addr + 4);
    app_entry_t app = (app_entry_t)pc;
    __set_MSP(sp);
    app();          /* never returns */
}
```

---

## Power Management Checklist

- [ ] Identify all power rails and their enable GPIO pins
- [ ] Use lowest sleep mode compatible with wakeup latency requirement
- [ ] Disable unused peripheral clocks (`RCC_APBxENR`)
- [ ] Configure unused GPIO as analog input (lowest leakage)
- [ ] Measure sleep current with a current probe before software optimization
- [ ] Wake sources: RTC alarm, GPIO EXTI, UART activity, WDT

---

## See Also

- `embedded-c/SKILL.md` — Low-level C patterns used in all layers  
- `stm32/SKILL.md` — STM32-specific clock/DMA/peripheral details  
- `esp32/SKILL.md` — ESP-IDF task and power management  
- `rpi-pico/SKILL.md` — RP2040 PIO and dual-core design  
