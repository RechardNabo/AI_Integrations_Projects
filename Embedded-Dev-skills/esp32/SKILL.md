---
name: esp32
description: >
  ESP32 and ESP32-S2/S3/C3/C6/H2 development skill using ESP-IDF (preferred)
  and Arduino framework. Use this skill for: ESP-IDF project setup, FreeRTOS
  tasks on dual cores, Wi-Fi (station/AP/provisioning), BLE/BT Classic,
  MQTT, HTTP/HTTPS client, NVS (non-volatile storage), OTA firmware update,
  deep sleep + wakeup sources, ADC/DAC, I2S audio, RMT peripheral (WS2812),
  LEDC PWM, TWAI (CAN), UART/SPI/I2C, partition table, menuconfig, and
  ESP-NOW. Triggers on: ESP32, ESP-IDF, esp_*, wifi, nvs, ble, mqtt, idf.py.
---

# ESP32 Skill

## Project Setup (ESP-IDF v5.x)

```bash
# Install ESP-IDF
git clone --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf
cd ~/esp/esp-idf && ./install.sh esp32,esp32s3

# Source environment (add to ~/.bashrc)
. ~/esp/esp-idf/export.sh

# Create project
idf.py create-project my_project
cd my_project
idf.py set-target esp32s3   # or esp32, esp32c3, etc.
idf.py menuconfig           # Configure options
idf.py build flash monitor  # Build, flash, open serial monitor
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```

---

## Dual-Core Task Pinning

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// PRO_CPU = Core 0 (also runs Wi-Fi/BT stack — avoid heavy tasks here)
// APP_CPU = Core 1 (application tasks)
#define PRO_CPU 0
#define APP_CPU 1

void wifi_manager_task(void *arg) {
    for (;;) {
        /* ... */
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void sensor_task(void *arg) {
    for (;;) {
        /* ... */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void app_main(void) {
    xTaskCreatePinnedToCore(wifi_manager_task, "wifi_mgr", 4096, NULL, 5, NULL, PRO_CPU);
    xTaskCreatePinnedToCore(sensor_task,       "sensors",  2048, NULL, 4, NULL, APP_CPU);
}
```

---

## Wi-Fi Station Mode

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"

static EventGroupHandle_t wifi_events;
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static void wifi_event_handler(void *arg, esp_event_base_t base,
                                int32_t id, void *data) {
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        xEventGroupSetBits(wifi_events, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(const char *ssid, const char *password) {
    nvs_flash_init();
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);

    esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, wifi_event_handler, NULL);
    esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, wifi_event_handler, NULL);

    wifi_config_t wifi_cfg = {};
    strlcpy((char*)wifi_cfg.sta.ssid,     ssid,     sizeof(wifi_cfg.sta.ssid));
    strlcpy((char*)wifi_cfg.sta.password, password, sizeof(wifi_cfg.sta.password));

    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(WIFI_IF_STA, &wifi_cfg);
    esp_wifi_start();
    esp_wifi_connect();

    /* Wait for connection */
    xEventGroupWaitBits(wifi_events, WIFI_CONNECTED_BIT, false, true, portMAX_DELAY);
}
```

---

## NVS (Non-Volatile Storage)

```c
#include "nvs_flash.h"
#include "nvs.h"

void nvs_write_u32(const char *key, uint32_t val) {
    nvs_handle_t h;
    nvs_open("storage", NVS_READWRITE, &h);
    nvs_set_u32(h, key, val);
    nvs_commit(h);
    nvs_close(h);
}

uint32_t nvs_read_u32(const char *key, uint32_t default_val) {
    nvs_handle_t h;
    uint32_t val = default_val;
    if (nvs_open("storage", NVS_READONLY, &h) == ESP_OK) {
        nvs_get_u32(h, key, &val);
        nvs_close(h);
    }
    return val;
}
```

---

## Deep Sleep + Wakeup

```c
#include "esp_sleep.h"

#define uS_TO_S 1000000ULL

void enter_deep_sleep(uint32_t sleep_sec) {
    // Timer wakeup
    esp_sleep_enable_timer_wakeup(sleep_sec * uS_TO_S);

    // GPIO wakeup (ESP32 only — use ext0 or ext1)
    esp_sleep_enable_ext0_wakeup(GPIO_NUM_33, 0); // LOW = wakeup

    // Save state to RTC memory before sleep
    // RTC_DATA_ATTR uint32_t boot_count = 0;  (persists through deep sleep)

    esp_deep_sleep_start();  // Does not return
}

void app_main(void) {
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    switch (cause) {
        case ESP_SLEEP_WAKEUP_TIMER: /* ... */ break;
        case ESP_SLEEP_WAKEUP_EXT0:  /* ... */ break;
        default: /* Cold boot */ break;
    }
}
```

---

## OTA Firmware Update

```c
#include "esp_ota_ops.h"
#include "esp_https_ota.h"

esp_err_t do_ota(const char *url) {
    esp_http_client_config_t cfg = {
        .url             = url,
        .cert_pem        = server_cert_pem_start,  // Embed server cert
        .skip_cert_common_name_check = false,
    };
    esp_https_ota_config_t ota_cfg = { .http_config = &cfg };
    esp_err_t ret = esp_https_ota(&ota_cfg);
    if (ret == ESP_OK) {
        esp_restart();
    }
    return ret;
}
// Partition table must have two OTA slots: ota_0 and ota_1
// See: partitions_two_ota.csv
```

---

## RMT — WS2812 LED Strip

```c
#include "driver/rmt_tx.h"
#include "led_strip_encoder.h"  // Component from idf-component-manager

rmt_channel_handle_t rmt_chan;
rmt_encoder_handle_t led_encoder;

void ws2812_init(int gpio_pin) {
    rmt_tx_channel_config_t cfg = {
        .gpio_num = gpio_pin,
        .clk_src  = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = 10 * 1000 * 1000,  // 10 MHz → 100ns resolution
        .mem_block_symbols = 64,
        .trans_queue_depth = 4,
    };
    rmt_new_tx_channel(&cfg, &rmt_chan);
    led_strip_encoder_config_t enc_cfg = { .resolution = 10000000 };
    rmt_new_led_strip_encoder(&enc_cfg, &led_encoder);
    rmt_enable(rmt_chan);
}
```

---

## Partition Table Example

```csv
# partitions.csv
# Name,   Type, SubType,  Offset,   Size,   Flags
nvs,      data, nvs,      0x9000,   0x6000,
otadata,  data, ota,      0xf000,   0x2000,
ota_0,    app,  ota_0,    0x10000,  1792K,
ota_1,    app,  ota_1,    ,         1792K,
storage,  data, spiffs,   ,         256K,
```

---

## Checklist

- [ ] `nvs_flash_init()` called before any NVS or Wi-Fi init
- [ ] Task stack sizes verified — ESP-IDF default is 2–4 KB, increase for HTTPS/JSON
- [ ] Wi-Fi and BT tasks pinned to PRO_CPU (Core 0)
- [ ] Deep sleep: RTC variables marked `RTC_DATA_ATTR`
- [ ] OTA: rollback enabled (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`)
- [ ] ADC2 not used while Wi-Fi is active (hardware conflict on ESP32)
- [ ] Logging level set to `WARN` or `ERROR` in production builds
