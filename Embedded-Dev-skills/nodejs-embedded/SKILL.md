---
name: nodejs-embedded
description: >
  Node.js skill for embedded and IoT contexts: running Node.js on Raspberry Pi
  or BeagleBone, interfacing with hardware from Node (serialport, i2c-bus,
  spi-device, onoff/pigpio for GPIO), MQTT with mqtt.js, HTTP/WebSocket servers
  on-device, Johnny-Five robotics framework, communication bridge between MCU
  (Arduino/ESP32/PIC) and a Pi over serial/UART, building dashboards with
  Express + Socket.io, and deploying with PM2 or systemd. Also covers: async
  patterns for hardware I/O, Buffer for binary protocol parsing, and Node.js
  on constrained devices (memory limits, no swap). Triggers on: Node.js on Pi,
  serialport, i2c-bus, johnny-five, mqtt.js, PM2, embedded Node, MCU bridge,
  hardware I/O Node.
---

# Node.js for Embedded Systems Skill

## Hardware I/O Packages

| Package | Purpose | Install |
|---------|---------|---------|
| `serialport` | UART/USB-Serial to MCU | `npm i serialport` |
| `i2c-bus` | I2C sensor/device access | `npm i i2c-bus` |
| `spi-device` | SPI peripheral access | `npm i spi-device` |
| `onoff` | GPIO (uses sysfs/libgpiod) | `npm i onoff` |
| `pigpio` | GPIO + hardware PWM (Pi only) | `npm i pigpio` |
| `mqtt` | MQTT client (IoT messaging) | `npm i mqtt` |
| `johnny-five` | Robotics/Arduino abstraction | `npm i johnny-five` |
| `ws` | WebSocket server | `npm i ws` |
| `express` | HTTP REST API / dashboard | `npm i express` |

---

## Serial Port — MCU Bridge Pattern

```js
// bridge.js — Receive structured data from Arduino/STM32/ESP32 over UART
const { SerialPort } = require('serialport');
const { ReadlineParser } = require('@serialport/parser-readline');

const port = new SerialPort({
    path: '/dev/ttyUSB0',  // or /dev/ttyAMA0 for Pi UART
    baudRate: 115200,
    autoOpen: false,
});

const parser = port.pipe(new ReadlineParser({ delimiter: '\n' }));

port.open(err => {
    if (err) { console.error('Port error:', err.message); process.exit(1); }
    console.log('Serial open');
});

// Receive: "TEMP:23.4,HUM:58.1\r\n"
parser.on('data', line => {
    const fields = Object.fromEntries(
        line.trim().split(',').map(f => f.split(':'))
    );
    console.log('Parsed:', fields);  // { TEMP: '23.4', HUM: '58.1' }
    publishMqtt(fields);
});

// Send command to MCU
function send_command(cmd) {
    port.write(cmd + '\n', err => {
        if (err) console.error('Write error:', err);
    });
}
```

---

## Binary Protocol Parser (Buffer)

```js
// MCU sends: [0xAA][0xBB][uint16 LE value][uint8 checksum]
parser.on('data', chunk => {
    if (chunk.length < 5) return;
    if (chunk[0] !== 0xAA || chunk[1] !== 0xBB) return;
    const value    = chunk.readUInt16LE(2);
    const checksum = chunk[4];
    const computed = (0xAA + 0xBB + (value & 0xFF) + ((value >> 8) & 0xFF)) & 0xFF;
    if (computed !== checksum) { console.warn('Bad checksum'); return; }
    console.log('Value:', value);
});

// Build binary packet to send to MCU
function build_packet(value) {
    const buf = Buffer.alloc(5);
    buf[0] = 0xAA;
    buf[1] = 0xBB;
    buf.writeUInt16LE(value, 2);
    buf[4] = (0xAA + 0xBB + (value & 0xFF) + ((value >> 8) & 0xFF)) & 0xFF;
    return buf;
}
```

---

## I2C (i2c-bus)

```js
const i2c = require('i2c-bus');

async function readSensor() {
    const bus = await i2c.openPromisified(1);  // /dev/i2c-1

    const BME280_ADDR = 0x76;

    // Write register address, then read
    await bus.writeByte(BME280_ADDR, 0xF3, 0x00);  // addr, reg, value
    const status = await bus.readByte(BME280_ADDR, 0xF3);

    // Bulk read: read 6 bytes from register 0xF7
    const buf = Buffer.alloc(6);
    await bus.readI2cBlock(BME280_ADDR, 0xF7, 6, buf);

    await bus.close();
    return buf;
}
```

---

## MQTT Client

```js
const mqtt = require('mqtt');

const client = mqtt.connect('mqtt://broker.local:1883', {
    clientId:  'pi-sensor-01',
    username:  'user',
    password:  'pass',
    keepalive: 60,
    reconnectPeriod: 2000,
    will: {  // Last-will: published if client disconnects unexpectedly
        topic:   'sensors/pi01/status',
        payload: 'offline',
        qos:     1,
        retain:  true,
    },
});

client.on('connect', () => {
    console.log('MQTT connected');
    client.subscribe('sensors/pi01/cmd', { qos: 1 });
    client.publish('sensors/pi01/status', 'online', { retain: true });
});

client.on('message', (topic, payload) => {
    const cmd = payload.toString();
    if (cmd === 'reboot') { /* ... */ }
});

function publish_sensor(temp, hum) {
    const payload = JSON.stringify({ temp, hum, ts: Date.now() });
    client.publish('sensors/pi01/data', payload, { qos: 0 });
}
```

---

## Express + Socket.io Dashboard

```js
// dashboard.js — Live sensor dashboard served from Pi
const express   = require('express');
const http      = require('http');
const { Server } = require('socket.io');

const app    = express();
const server = http.createServer(app);
const io     = new Server(server);

app.use(express.static('public'));

io.on('connection', socket => {
    console.log('Browser connected:', socket.id);
    socket.on('command', data => handle_command(data));
});

// Push sensor data to all browsers
function broadcast_sensor(data) {
    io.emit('sensor_data', data);
}

// Called from serial/MQTT listener
setInterval(() => {
    broadcast_sensor({ temp: 23.4, hum: 58, ts: Date.now() });
}, 1000);

server.listen(3000, () => console.log('Dashboard at http://localhost:3000'));
```

---

## GPIO with onoff

```js
const { Gpio } = require('onoff');

const led = new Gpio(17, 'out');
const btn = new Gpio(27, 'in', 'falling', { debounceTimeout: 10 });

btn.watch((err, value) => {
    if (err) throw err;
    led.writeSync(led.readSync() ^ 1);  // Toggle LED
});

// Cleanup on exit
process.on('SIGINT', () => {
    led.unexport();
    btn.unexport();
    process.exit();
});
```

---

## PM2 Process Manager (Production)

```bash
npm install -g pm2

# Start app
pm2 start bridge.js --name sensor-bridge

# Auto-start on boot
pm2 startup systemd
pm2 save

# Useful commands
pm2 status
pm2 logs sensor-bridge
pm2 restart sensor-bridge
pm2 monit           # Resource usage dashboard
```

---

## Memory & Performance on Constrained Devices

```js
// Check heap usage
const used = process.memoryUsage();
console.log(`Heap: ${Math.round(used.heapUsed/1024/1024)} MB / ${Math.round(used.heapTotal/1024/1024)} MB`);

// Limit V8 heap on devices with <512MB RAM
// Start Node with: node --max-old-space-size=128 app.js

// Avoid memory leaks:
// - Remove all EventEmitter listeners when no longer needed
// - Use streaming APIs (fs.createReadStream) for large files
// - Clear intervals and timeouts on shutdown
// - Avoid global arrays that grow unbounded (use circular buffers)
```

---

## Async Patterns for Hardware I/O

```js
// PATTERN: Read sensor every N seconds, don't let errors crash loop
async function sensor_loop() {
    while (true) {
        try {
            const data = await readSensor();
            publish_sensor(data);
        } catch (err) {
            console.error('Sensor read failed:', err.message);
        }
        await sleep(5000);
    }
}

const sleep = ms => new Promise(r => setTimeout(r, ms));

// PATTERN: Serial + MQTT in same process — both are event-driven, no threads needed
// Node's event loop handles both concurrently without blocking
```

---

## Checklist

- [ ] Node.js LTS version installed: `nvm install --lts` (avoid distro's outdated package)
- [ ] User in `dialout` group for serial: `sudo usermod -aG dialout $USER`
- [ ] User in `i2c`, `gpio`, `spi` groups for hardware access
- [ ] `pigpio` requires root OR run `sudo pigpiod` daemon first
- [ ] PM2 / systemd configured for auto-restart and boot persistence
- [ ] `process.on('SIGINT', cleanup)` to unexport GPIO on Ctrl+C
- [ ] `--max-old-space-size` set if running on Pi Zero / 256 MB board
- [ ] MQTT client has Last Will and reconnect logic configured
