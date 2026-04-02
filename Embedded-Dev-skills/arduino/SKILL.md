---
name: arduino
description: >
  Arduino development skill covering AVR (Uno/Nano/Mega), ARM (Due, Zero),
  and Arduino-framework-on-other-chips (ESP32, STM32, RP2040 via Arduino IDE).
  Use this skill for: Arduino project structure, library usage, Serial/Wire/SPI,
  pin manipulation, millis()-based timing, interrupt attachment, EEPROM, watchdog,
  low-power libraries, PlatformIO setup, custom board definitions, and moving
  from Arduino to bare-metal C. Triggers on: Arduino, .ino, setup(), loop(),
  analogRead, digitalRead, Wire, Serial, PlatformIO, AVR, ATmega.
---

# Arduino Skill

## Project Structure (PlatformIO — Recommended over Arduino IDE)

```
my_project/
├── platformio.ini          # Build config
├── src/
│   └── main.cpp            # Entry point
├── include/
│   └── config.h
├── lib/
│   └── MyDriver/           # Local libraries
└── test/
    └── test_main.cpp       # Unity unit tests
```

```ini
; platformio.ini
[env:uno]
platform  = atmelavr
board     = uno
framework = arduino
monitor_speed = 115200
lib_deps  =
    adafruit/Adafruit BusIO
    bblanchon/ArduinoJson

[env:esp32]
platform  = espressif32
board     = esp32dev
framework = arduino
monitor_speed = 115200
```

---

## Timing — Never Use delay()

```cpp
// BAD: delay() blocks everything
void loop() {
    blink();
    delay(500); // blocks UART, sensors, everything
}

// GOOD: millis()-based non-blocking timing
void loop() {
    static uint32_t lastBlink = 0;
    if (millis() - lastBlink >= 500) {
        lastBlink = millis();
        digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
    }
    // Other tasks run freely
    handle_serial();
    read_sensors();
}
```

---

## Fast GPIO (AVR)

```cpp
// Arduino: ~3 µs per digitalWrite call (AVR Uno)
// Direct port: ~63 ns

// Direct register access on AVR
#define LED_PIN 5  // PD5

// Equivalent to pinMode(5, OUTPUT)
DDRD  |=  (1 << LED_PIN);

// Equivalent to digitalWrite(5, HIGH)
PORTD |=  (1 << LED_PIN);

// Equivalent to digitalWrite(5, LOW)
PORTD &= ~(1 << LED_PIN);

// Read: equivalent to digitalRead(5)
bool state = (PIND >> LED_PIN) & 1;
```

---

## Interrupts

```cpp
volatile bool isr_flag = false;

void pin_isr() {
    isr_flag = true;  // Keep ISR minimal
}

void setup() {
    pinMode(2, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(2), pin_isr, FALLING);
}

void loop() {
    if (isr_flag) {
        isr_flag = false;
        handle_event();
    }
}

// Timer interrupt (AVR, 1ms tick)
ISR(TIMER1_COMPA_vect) {
    system_tick++;
}
```

---

## Serial / UART

```cpp
void setup() {
    Serial.begin(115200);
    while (!Serial);   // Wait for USB Serial on Leonardo/Due/Zero
}

// Non-blocking read
void handle_serial() {
    static char buf[64];
    static uint8_t idx = 0;
    while (Serial.available()) {
        char c = Serial.read();
        if (c == '\n') {
            buf[idx] = 0;
            process_command(buf);
            idx = 0;
        } else if (idx < sizeof(buf) - 1) {
            buf[idx++] = c;
        }
    }
}

// printf-style to Serial
// Add to setup(): Serial.printf("Value: %d\n", val); // ESP32/ARM only
// AVR: use F() macro to store strings in Flash
Serial.println(F("Hello from Flash")); // saves SRAM
```

---

## Wire (I2C)

```cpp
#include <Wire.h>

void i2c_write_reg(uint8_t addr, uint8_t reg, uint8_t val) {
    Wire.beginTransmission(addr);
    Wire.write(reg);
    Wire.write(val);
    Wire.endTransmission();
}

uint8_t i2c_read_reg(uint8_t addr, uint8_t reg) {
    Wire.beginTransmission(addr);
    Wire.write(reg);
    Wire.endTransmission(false);  // Repeated START
    Wire.requestFrom(addr, (uint8_t)1);
    return Wire.read();
}

// Scan for devices
void i2c_scan() {
    for (uint8_t a = 1; a < 127; a++) {
        Wire.beginTransmission(a);
        if (Wire.endTransmission() == 0) {
            Serial.printf("Found: 0x%02X\n", a);
        }
    }
}
```

---

## EEPROM

```cpp
#include <EEPROM.h>

struct Config { uint32_t magic; float setpoint; uint8_t mode; };
#define MAGIC 0xDEADBEEF

void config_save(Config *c) {
    EEPROM.put(0, *c);   // put() uses update() internally — no excess writes
}

bool config_load(Config *c) {
    EEPROM.get(0, *c);
    return c->magic == MAGIC;
}
```

---

## Watchdog

```cpp
#include <avr/wdt.h>   // AVR
// ESP32: use esp_task_wdt.h

void setup() {
    wdt_enable(WDTO_2S);  // Reset if not fed within 2s
}

void loop() {
    wdt_reset();           // Feed watchdog
    // ... main work ...
}
```

---

## Moving from Arduino to Bare Metal

When performance or size becomes a constraint:

| Arduino | Bare Metal Equivalent |
|---------|-----------------------|
| `digitalWrite()` | `PORTx |= (1<<n)` |
| `analogRead()` | Direct ADC registers |
| `delay()`| `_delay_ms()` or timer ISR |
| `Serial.begin()` | UART registers (UBRR, UCSR) |
| `Wire.begin()` | TWI registers (TWBR, TWCR) |
| `millis()` | Timer1 overflow ISR counter |
| `attachInterrupt()` | `ISR(INTx_vect)` |

---

## Checklist

- [ ] Using `millis()` instead of `delay()` for non-blocking timing
- [ ] ISR variables declared `volatile`
- [ ] String literals in Flash with `F()` macro (AVR SRAM is tiny: 2 KB on Uno)
- [ ] `Serial.flush()` before `sleep_mode()` to drain TX buffer
- [ ] Watchdog enabled in production firmware
- [ ] PlatformIO used instead of Arduino IDE for version control and CI
