---
name: rpi-pico
description: >
  Raspberry Pi Pico / RP2040 and Pico 2 / RP2350 development skill using the
  Pico C/C++ SDK. Use this skill for: SDK setup and CMake config, dual-core
  programming (core0/core1), PIO (Programmable I/O) state machine design and
  assembly, DMA chaining, UART/SPI/I2C/PWM/ADC APIs, flash XIP and DMA to
  flash, multicore FIFOs, hardware interpolator, sleep/dormant modes,
  USB (TinyUSB via SDK), and Pico 2 (RP2350) differences. Triggers on:
  RP2040, RP2350, Pico, pico-sdk, PIO, .pio, pio_sm, gpio_put, pico/stdlib.h,
  multicore, CMakeLists pico_sdk_import.
---

# Raspberry Pi Pico / RP2040 Skill

## SDK Setup

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt install cmake gcc-arm-none-eabi libnewlib-arm-none-eabi build-essential

# Clone SDK
git clone https://github.com/raspberrypi/pico-sdk.git ~/pico-sdk
cd ~/pico-sdk && git submodule update --init

# Add to ~/.bashrc
export PICO_SDK_PATH=~/pico-sdk
```

### CMakeLists.txt Template

```cmake
cmake_minimum_required(VERSION 3.13)
include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)

project(my_project C CXX ASM)
set(CMAKE_C_STANDARD 11)

pico_sdk_init()

add_executable(my_project main.c)

target_link_libraries(my_project
    pico_stdlib
    hardware_gpio
    hardware_spi
    hardware_i2c
    hardware_uart
    hardware_pwm
    hardware_adc
    hardware_dma
    hardware_pio
    pico_multicore
)

# Enable USB serial (stdio over USB)
pico_enable_stdio_usb(my_project 1)
pico_enable_stdio_uart(my_project 0)

# Generate .uf2 file
pico_add_extra_outputs(my_project)
```

---

## GPIO

```c
#include "pico/stdlib.h"
#include "hardware/gpio.h"

gpio_init(25);                     // LED on Pico
gpio_set_dir(25, GPIO_OUT);
gpio_put(25, 1);                   // High
gpio_put(25, 0);                   // Low
gpio_xor_mask(1u << 25);           // Toggle (atomic)

// Input with pull-up
gpio_init(15);
gpio_set_dir(15, GPIO_IN);
gpio_pull_up(15);
bool val = gpio_get(15);

// IRQ
gpio_set_irq_enabled_with_callback(15,
    GPIO_IRQ_EDGE_FALL, true, &gpio_callback);

void gpio_callback(uint gpio, uint32_t events) {
    // Runs on core 0 only
}
```

---

## Dual-Core Programming

```c
#include "pico/multicore.h"

volatile bool shared_flag = false;

void core1_entry(void) {
    while (true) {
        // Core 1 work
        uint32_t msg = multicore_fifo_pop_blocking();  // Wait for message
        shared_flag = true;
        multicore_fifo_push_blocking(0xDEAD);           // Reply
    }
}

int main(void) {
    multicore_launch_core1(core1_entry);

    // Send message to core1
    multicore_fifo_push_blocking(0x1234);

    // Wait for reply
    uint32_t reply = multicore_fifo_pop_blocking();

    // Spinlock for shared data
    spin_lock_t *lock = spin_lock_init(spin_lock_claim_unused(true));
    uint32_t save = spin_lock_blocking(lock);
    // ... critical section ...
    spin_unlock(lock, save);
}
```

---

## PIO (Programmable I/O) — Core RP2040 Feature

PIO = 2 independent state machine blocks (PIO0, PIO1), each with 4 state machines. Run custom serial protocols at up to 133 MHz, independent of CPU.

### WS2812 PIO Example (.pio file)

```pio
; ws2812.pio — sends NZR data for WS2812 LEDs
.program ws2812
.side_set 1

.define public T1 2
.define public T2 5
.define public T3 3

.wrap_target
bitloop:
    out x, 1        side 0 [T3 - 1]  ; Get bit, drive low
    jmp !x do_zero  side 1 [T1 - 1]  ; Drive high
do_one:
    jmp bitloop     side 1 [T2 - 1]  ; Long high = 1
do_zero:
    nop             side 0 [T2 - 1]  ; Short high = 0
.wrap
```

```c
// Load and start PIO program (generated header from ws2812.pio)
#include "ws2812.pio.h"

PIO pio = pio0;
uint sm  = pio_claim_unused_sm(pio, true);
uint offset = pio_add_program(pio, &ws2812_program);
ws2812_program_init(pio, sm, offset, WS2812_PIN, 800000, false);

// Send 24-bit GRB pixel
static inline void put_pixel(uint32_t grb) {
    pio_sm_put_blocking(pio, sm, grb << 8u);
}
```

---

## DMA

```c
#include "hardware/dma.h"

int dma_ch = dma_claim_unused_channel(true);

dma_channel_config cfg = dma_channel_get_default_config(dma_ch);
channel_config_set_transfer_data_size(&cfg, DMA_SIZE_8);
channel_config_set_read_increment(&cfg, true);
channel_config_set_write_increment(&cfg, false);
channel_config_set_dreq(&cfg, DREQ_SPI0_TX);  // SPI0 TX pacing

dma_channel_configure(dma_ch, &cfg,
    &spi_get_hw(spi0)->dr,  // Write to SPI DR
    tx_buffer,              // Read from buffer
    len,                    // Transfer count
    true);                  // Start immediately

dma_channel_wait_for_finish_blocking(dma_ch);
```

---

## ADC

```c
#include "hardware/adc.h"

adc_init();
adc_gpio_init(26);      // ADC0 = GPIO26
adc_select_input(0);    // Select ADC0

uint16_t raw = adc_read();          // 12-bit: 0–4095
float volts  = raw * 3.3f / 4096.0f;

// Temperature sensor (internal, ADC4)
adc_set_temp_sensor_enabled(true);
adc_select_input(4);
float temp_c = 27.0f - (adc_read() * 3.3f / 4096.0f - 0.706f) / 0.001721f;
```

---

## Sleep / Dormant Modes

```c
#include "pico/sleep.h"
#include "hardware/rosc.h"
#include "hardware/rtc.h"

// Dormant: lowest power, only ROSC/XOSC edge wakeup or RTC alarm
void go_dormant_until_pin(uint gpio_pin) {
    rosc_dormant();  // Switch to ROSC, stop XOSC
    // Wakeup on GPIO edge — must re-init clocks after wakeup
    gpio_set_dormant_irq_enabled(gpio_pin, GPIO_IRQ_EDGE_FALL, true);
    sleep_goto_dormant_until_edge_high(gpio_pin);
    // Re-init after wakeup:
    clocks_init();
}
```

---

## RP2350 (Pico 2) Differences

| Feature | RP2040 | RP2350 |
|---------|--------|--------|
| Core | Dual Cortex-M0+ | Dual Cortex-M33 OR RISC-V Hazard3 |
| Speed | 133 MHz | 150 MHz |
| Flash | External only | External + 4 KB OTP |
| RAM | 264 KB | 520 KB |
| PIO | 2 × 4 SM | 3 × 4 SM |
| Security | None | TrustZone, secure boot |

```c
// RP2350 specific: choose RISC-V or ARM cores in CMake
# set(PICO_PLATFORM rp2350-riscv)  # RISC-V mode
# set(PICO_PLATFORM rp2350)        # ARM mode (default)
```

---

## Checklist

- [ ] `pico_enable_stdio_usb(target 1)` for USB serial output
- [ ] PIO programs loaded from `.pio.h` generated headers (run `pioasm` or let CMake)
- [ ] DMA buffers 4-byte aligned for 32-bit transfers
- [ ] `multicore_fifo_push_blocking` used only from one core per direction
- [ ] `sleep_ms()` / `busy_wait_us()` distinction: `sleep_ms` is interruptible
- [ ] Flash writes disable XIP cache: call `flash_range_program()` with IRQs disabled
