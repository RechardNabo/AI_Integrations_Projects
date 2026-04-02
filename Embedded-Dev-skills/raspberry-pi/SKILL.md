---
name: raspberry-pi
description: >
  Raspberry Pi (full Linux) development skill for Pi 1/2/3/4/5 and Compute
  Module variants. Use this skill for: Raspberry Pi OS setup, GPIO via
  libgpiod or /dev/gpiomem, SPI/I2C/UART from Linux userspace, C programs
  with pigpio or WiringPi, systemd service creation, cross-compilation from
  x86 host, device tree overlays, camera (libcamera), hardware PWM, real-time
  scheduling (PREEMPT_RT), interfacing with external MCUs over UART/SPI/I2C,
  and Node.js on Pi. Triggers on: Raspberry Pi, RPi, bcm2835, pigpio, libgpiod,
  /dev/spidev, /dev/i2c, raspi-config, devicetree, systemd, Pi 4, Pi 5.
---

# Raspberry Pi Skill

## Hardware GPIO Access Options (Choose One)

| Library | Language | Speed | Complexity | Recommended for |
|---------|----------|-------|------------|-----------------|
| `libgpiod` | C/Python | Medium | Low | New projects — kernel-supported |
| `pigpio` | C/Python | High | Medium | Bit-bang protocols, PWM, servo |
| `bcm2835` | C | Very high | Medium | Direct BCM register access |
| `/sys/class/gpio` | Shell/C | Slow | Low | Quick scripts (deprecated) |
| `RPi.GPIO` | Python | Medium | Low | Python prototyping only |
| `gpiozero` | Python | Low | Very low | Education, rapid prototyping |

---

## libgpiod (Recommended — Kernel ≥ 5.10)

```c
#include <gpiod.h>

int main(void) {
    struct gpiod_chip *chip = gpiod_chip_open("/dev/gpiochip0");
    struct gpiod_line *line = gpiod_chip_get_line(chip, 17); // GPIO17

    gpiod_line_request_output(line, "my-app", 0);
    gpiod_line_set_value(line, 1);  // HIGH
    gpiod_line_set_value(line, 0);  // LOW

    // Input with edge detection
    struct gpiod_line *btn = gpiod_chip_get_line(chip, 27);
    gpiod_line_request_falling_edge_events(btn, "my-app");
    gpiod_line_event_wait(btn, NULL);  // Block until event

    gpiod_chip_close(chip);
}
// Build: gcc -o app app.c -lgpiod
```

---

## SPI from C (Linux spidev)

```c
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int spi_fd;

void spi_init(const char *dev, uint32_t speed_hz) {
    spi_fd = open(dev, O_RDWR);  // "/dev/spidev0.0"
    uint8_t mode  = SPI_MODE_0;
    uint8_t bits  = 8;
    ioctl(spi_fd, SPI_IOC_WR_MODE, &mode);
    ioctl(spi_fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(spi_fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed_hz);
}

uint8_t spi_transfer(uint8_t tx) {
    uint8_t rx = 0;
    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)&tx,
        .rx_buf = (unsigned long)&rx,
        .len    = 1,
    };
    ioctl(spi_fd, SPI_IOC_MESSAGE(1), &tr);
    return rx;
}
```

---

## I2C from C

```c
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>
#include <fcntl.h>

int i2c_fd;

void i2c_init(const char *dev, uint8_t addr) {
    i2c_fd = open(dev, O_RDWR);  // "/dev/i2c-1"
    ioctl(i2c_fd, I2C_SLAVE, addr);
}

void i2c_write_reg(uint8_t reg, uint8_t val) {
    uint8_t buf[2] = {reg, val};
    write(i2c_fd, buf, 2);
}

uint8_t i2c_read_reg(uint8_t reg) {
    write(i2c_fd, &reg, 1);
    uint8_t val;
    read(i2c_fd, &val, 1);
    return val;
}
```

---

## UART from C

```c
#include <termios.h>
#include <fcntl.h>

int uart_fd;

void uart_init(const char *dev, speed_t baud) {
    // baud: B9600, B115200, B460800, etc.
    uart_fd = open(dev, O_RDWR | O_NOCTTY | O_NONBLOCK);
    struct termios tty = {};
    tcgetattr(uart_fd, &tty);
    cfsetispeed(&tty, baud);
    cfsetospeed(&tty, baud);
    tty.c_cflag = (tty.c_cflag & ~CSIZE) | CS8;
    tty.c_cflag |= (CLOCAL | CREAD);
    tty.c_cflag &= ~(PARENB | PARODD | CSTOPB | CRTSCTS);
    tty.c_lflag = 0;   // Raw mode
    tty.c_oflag = 0;
    tty.c_cc[VMIN]  = 1;
    tty.c_cc[VTIME] = 5;  // 0.5s read timeout
    tcsetattr(uart_fd, TCSANOW, &tty);
}
```

---

## systemd Service (Auto-start on Boot)

```ini
# /etc/systemd/system/my-sensor.service
[Unit]
Description=My Sensor Daemon
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/my-sensor
ExecStart=/home/pi/my-sensor/sensor_app
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable my-sensor
sudo systemctl start my-sensor
journalctl -u my-sensor -f   # Follow logs
```

---

## Real-Time Scheduling (for Tight Timing)

```c
#include <sched.h>

void set_realtime_priority(int prio) {
    struct sched_param sp = { .sched_priority = prio };
    // prio: 1 (low) to 99 (high)
    sched_setscheduler(0, SCHED_FIFO, &sp);
    // Lock memory to prevent page faults
    mlockall(MCL_CURRENT | MCL_FUTURE);
}
// For tighter RT: use PREEMPT_RT patched kernel
// Check: uname -r | grep -i rt
```

---

## Device Tree Overlay — Enable SPI/I2C/UART

```bash
# /boot/config.txt (Pi OS) or /boot/firmware/config.txt (Pi OS Bookworm)

dtparam=spi=on           # Enable SPI0
dtparam=i2c_arm=on       # Enable I2C1
dtparam=i2c_arm_baudrate=400000  # 400kHz I2C
enable_uart=1            # Enable UART0 (disables BT on Pi 3/4 unless configured)
dtoverlay=disable-bt     # Disable BT, restore UART0 to GPIO14/15
dtoverlay=uart1          # Enable UART1

# Check enabled overlays
dtoverlay -l
# Check I2C devices
i2cdetect -y 1
```

---

## Cross-Compilation from Ubuntu/Debian Host

```bash
# Install cross toolchain
sudo apt install gcc-aarch64-linux-gnu  # Pi 4/5 (64-bit)
sudo apt install gcc-arm-linux-gnueabihf # Pi 2/3 (32-bit)

# Compile
aarch64-linux-gnu-gcc -o app app.c -lgpiod

# rsync and run
rsync -avz app pi@raspberrypi.local:~/
ssh pi@raspberrypi.local './app'
```

---

## Checklist

- [ ] SPI/I2C/UART enabled in `config.txt` and confirmed with `ls /dev/spi* /dev/i2c*`
- [ ] User in `gpio`, `spi`, `i2c`, `dialout` groups: `sudo usermod -aG gpio,spi,i2c,dialout $USER`
- [ ] SD card is class 10 / A1 rated — slow cards cause filesystem corruption
- [ ] Proper shutdown before power cut: `sudo shutdown -h now` — never pull power
- [ ] `pigpiod` daemon running if using pigpio: `sudo systemctl enable pigpiod`
- [ ] Temperature monitored: `vcgencmd measure_temp` — throttling at 80°C on Pi 4
