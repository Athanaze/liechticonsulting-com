matrix_keyboard/CMakeLists.txt:
# The following lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
# "Trim" the build. Include the minimal set of components, main, and anything it depends on.
idf_build_set_property(MINIMAL_BUILD ON)
project(matrix_keyboard)


matrix_keyboard/README.md:
| Supported Targets | ESP32-S2 |
| ----------------- | -------- |

# Matrix Keyboard Example (based on Dedicated GPIO)

(See the README.md file in the upper level 'examples' directory for more information about examples.)

## Overview

This example mainly illustrates how to drive a common matrix keyboard using the dedicated GPIO APIs, including:

* Manipulate the level on a group of GPIOs
* Trigger edge interrupt
* Read level on a group of GPIOs

Dedicated GPIO is designed to speed up CPU operations on one or a group of GPIOs by writing assembly codes with Espressif customized instructions (please refer TRM to get more information about these instructions).

This matrix keyboard driver is interrupt-driven, supports a configurable debounce time. GPIOs used by row and column lines are also configurable during driver installation stage.

## How to use example

### Hardware Required

This example can run on any target that has the dedicated feature (e.g. ESP32-S2). It's not necessary for your matrix board to have pull-up resisters on row/column lines. The driver has enabled internal pull-up resister by default. A typical matrix board should look as follows:

```text
row_0   +--------+-------------------+------------------------------+-----------------+
                 |                   |                              |
                 |       +           |       +                      |       +
                 |     +-+-+         |     +-+-+          ......    |     +-+-+
  .              +-----+   +-----+   +-----+   +-----+              +-----+   +-----+
  .                              |                   |                              |
  .                      .       |           .       |                      .       |
                         .       |           .       |    ......            .       |
                         .       |           .       |                      .       |
                         .       |           .       |                      .       |
                                 |                   |                              |
row_n   +--------+-------------------+------------------------------+-----------------+
                 |               |   |               |              |               |
                 |       +       |   |       +       |              |       +       |
                 |     +-+-+     |   |     +-+-+     |    ......    |     +-+-+     |
                 +-----+   +-----+   +-----+   +-----+              +-----+   +-----+
                                 |                   |                              |
                                 |                   |                              |
                                 |                   |                              |
                                 +                   +                              +
                                col_0               col_1          ......          col_m
```

### Build and Flash

Build the project and flash it to the board, then run monitor tool to view serial output:

```text
idf.py -p PORT flash monitor
```

(Replace PORT with the name of the serial port to use.)

(To exit the serial monitor, type ``Ctrl-]``.)

See the Getting Started Guide for full steps to configure and use ESP-IDF to build projects.

## Example Output

```text
I (2883) example: press event, key code = 0002
I (3003) example: release event, key code = 0002
I (5053) example: press event, key code = 0001
I (5203) example: release event, key code = 0001
I (6413) example: press event, key code = 0000
I (6583) example: release event, key code = 0000
I (7963) example: press event, key code = 0003
I (8113) example: release event, key code = 0003
I (8773) example: press event, key code = 0103
I (8923) example: release event, key code = 0103
I (9543) example: press event, key code = 0203
I (9683) example: release event, key code = 0203
```

## Troubleshooting

For any technical queries, please open an [issue](https://github.com/espressif/esp-idf/issues) on GitHub. We will get back to you soon.


matrix_keyboard/main/CMakeLists.txt:
idf_component_register(SRCS "matrix_keyboard_example_main.c"
                       PRIV_REQUIRES matrix_keyboard
                       INCLUDE_DIRS "")


matrix_keyboard/main/matrix_keyboard_example_main.c:
/*
 * SPDX-FileCopyrightText: 2020-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>
#include "esp_log.h"
#include "matrix_keyboard.h"

const static char *TAG = "example";

/**
 * @brief Matrix keyboard event handler
 * @note This function is run under OS timer task context
 */
esp_err_t example_matrix_kbd_event_handler(matrix_kbd_handle_t mkbd_handle, matrix_kbd_event_id_t event, void *event_data, void *handler_args)
{
    uint32_t key_code = (uint32_t)event_data;
    switch (event) {
    case MATRIX_KBD_EVENT_DOWN:
        ESP_LOGI(TAG, "press event, key code = %04"PRIx32, key_code);
        break;
    case MATRIX_KBD_EVENT_UP:
        ESP_LOGI(TAG, "release event, key code = %04"PRIx32, key_code);
        break;
    }
    return ESP_OK;
}

void app_main(void)
{
    matrix_kbd_handle_t kbd = NULL;
    // Apply default matrix keyboard configuration
    matrix_kbd_config_t config = MATRIX_KEYBOARD_DEFAULT_CONFIG();
    // Set GPIOs used by row and column line
    config.col_gpios = (int[]) {
        10, 11, 12, 13
    };
    config.nr_col_gpios = 4;
    config.row_gpios = (int[]) {
        14, 15, 16, 17
    };
    config.nr_row_gpios = 4;
    // Install matrix keyboard driver
    matrix_kbd_install(&config, &kbd);
    // Register keyboard input event handler
    matrix_kbd_register_event_handler(kbd, example_matrix_kbd_event_handler, NULL);
    // Keyboard start to work
    matrix_kbd_start(kbd);
}


matrix_keyboard/components/matrix_keyboard/CMakeLists.txt:
set(component_srcs "src/matrix_keyboard.c")

idf_component_register(SRCS "${component_srcs}"
                       INCLUDE_DIRS "include"
                       PRIV_INCLUDE_DIRS ""
                       PRIV_REQUIRES "esp_driver_gpio"
                       REQUIRES "")


matrix_keyboard/components/matrix_keyboard/include/matrix_keyboard.h:
/*
 * SPDX-FileCopyrightText: 2020-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#pragma once

#include <stdint.h>
#include "esp_err.h"

#ifdef __cplusplus
extern "C" {
#endif

#define MAKE_KEY_CODE(row, col) ((row << 8) | (col))
#define GET_KEY_CODE_ROW(code)  ((code >> 8) & 0xFF)
#define GET_KEY_CODE_COL(code)  (code & 0xFF)

/**
 * @brief Type defined for matrix keyboard handle
 *
 */
typedef struct matrix_kbd_t *matrix_kbd_handle_t;

/**
 * @brief Matrix keyboard event ID
 *
 */
typedef enum {
    MATRIX_KBD_EVENT_DOWN, /*!< Key is pressed down */
    MATRIX_KBD_EVENT_UP    /*!< Key is released */
} matrix_kbd_event_id_t;

/**
 * @brief Type defined for matrix keyboard event handler
 *
 * @note The event handler runs in a OS timer context
 *
 * @param[in] mkbd_handle Handle of matrix keyboard that return from `matrix_kbd_install`
 * @param[in] event Event ID, refer to `matrix_kbd_event_id_t` to see all supported events
 * @param[in] event_data Data for corresponding event
 * @param[in] handler_args Arguments that user passed in from `matrix_kbd_register_event_handler`
 * @return Currently always return ESP_OK
 */
typedef esp_err_t (*matrix_kbd_event_handler)(matrix_kbd_handle_t mkbd_handle, matrix_kbd_event_id_t event, void *event_data, void *handler_args);

/**
 * @brief Configuration structure defined for matrix keyboard
 *
 */
typedef struct {
    const int *row_gpios;  /*!< Array, contains GPIO numbers used by row line */
    const int *col_gpios;  /*!< Array, contains GPIO numbers used by column line */
    uint32_t nr_row_gpios; /*!< row_gpios array size */
    uint32_t nr_col_gpios; /*!< col_gpios array size */
    uint32_t debounce_ms;  /*!< Debounce time */
} matrix_kbd_config_t;

/**
 * @brief Default configuration for matrix keyboard driver
 *
 */
#define MATRIX_KEYBOARD_DEFAULT_CONFIG() \
{                                        \
    .row_gpios = NULL,                   \
    .col_gpios = NULL,                   \
    .nr_row_gpios = 0,                   \
    .nr_col_gpios = 0,                   \
    .debounce_ms = 20,                   \
}

/**
 * @brief Install matrix keyboard driver
 *
 * @param[in] config Configuration of matrix keyboard driver
 * @param[out] mkbd_handle Returned matrix keyboard handle if installation succeed
 * @return
 *      - ESP_OK: Install matrix keyboard driver successfully
 *      - ESP_ERR_INVALID_ARG: Install matrix keyboard driver failed because of some invalid argument
 *      - ESP_ERR_NO_MEM: Install matrix keyboard driver failed because there's no enough capable memory
 *      - ESP_FAIL: Install matrix keyboard driver failed because of other error
 */
esp_err_t matrix_kbd_install(const matrix_kbd_config_t *config, matrix_kbd_handle_t *mkbd_handle);

/**
 * @brief Uninstall matrix keyboard driver
 *
 * @param[in] mkbd_handle Handle of matrix keyboard that return from `matrix_kbd_install`
 * @return
 *      - ESP_OK: Uninstall matrix keyboard driver successfully
 *      - ESP_ERR_INVALID_ARG: Uninstall matrix keyboard driver failed because of some invalid argument
 *      - ESP_FAIL: Uninstall matrix keyboard driver failed because of other error
 */
esp_err_t matrix_kbd_uninstall(matrix_kbd_handle_t mkbd_handle);

/**
 * @brief Start matrix keyboard driver
 *
 * @param[in] mkbd_handle Handle of matrix keyboard that return from `matrix_kbd_install`
 * @return
 *      - ESP_OK: Start matrix keyboard driver successfully
 *      - ESP_ERR_INVALID_ARG: Start matrix keyboard driver failed because of some invalid argument
 *      - ESP_FAIL: Start matrix keyboard driver failed because of other error
 */
esp_err_t matrix_kbd_start(matrix_kbd_handle_t mkbd_handle);

/**
 * @brief Stop matrix kayboard driver
 *
 * @param[in] mkbd_handle Handle of matrix keyboard that return from `matrix_kbd_install`
 * @return
 *      - ESP_OK: Stop matrix keyboard driver successfully
 *      - ESP_ERR_INVALID_ARG: Stop matrix keyboard driver failed because of some invalid argument
 *      - ESP_FAIL: Stop matrix keyboard driver failed because of other error
 */
esp_err_t matrix_kbd_stop(matrix_kbd_handle_t mkbd_handle);

/**
 * @brief Register matrix keyboard event handler
 *
 * @param[in] mkbd_handle Handle of matrix keyboard that return from `matrix_kbd_install`
 * @param[in] handler Event handler
 * @param[in] args Arguments that will be passed to the handler
 * @return
 *      - ESP_OK: Register event handler successfully
 *      - ESP_ERR_INVALID_ARG: Register event handler failed because of some invalid argument
 *      - ESP_FAIL: Register event handler failed because of other error
 */
esp_err_t matrix_kbd_register_event_handler(matrix_kbd_handle_t mkbd_handle, matrix_kbd_event_handler handler, void *args);

#ifdef __cplusplus
}
#endif


matrix_keyboard/components/matrix_keyboard/src/matrix_keyboard.c:
/*
 * SPDX-FileCopyrightText: 2020-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/timers.h"
#include "esp_log.h"
#include "esp_check.h"
#include "driver/dedic_gpio.h"
#include "driver/gpio.h"
#include "esp_private/gpio.h"
#include "matrix_keyboard.h"

static const char *TAG = "mkbd";

typedef struct matrix_kbd_t matrix_kbd_t;

struct matrix_kbd_t {
    dedic_gpio_bundle_handle_t row_bundle;
    dedic_gpio_bundle_handle_t col_bundle;
    uint32_t nr_row_gpios;
    uint32_t nr_col_gpios;
    TimerHandle_t debounce_timer;
    matrix_kbd_event_handler event_handler;
    void *event_handler_args;
    uint32_t row_state[0];
};

static IRAM_ATTR bool matrix_kbd_row_isr_callback(dedic_gpio_bundle_handle_t row_bundle, uint32_t row_index, void *args)
{
    BaseType_t high_task_wakeup = pdFALSE;
    matrix_kbd_t *mkbd = (matrix_kbd_t *)args;

    // temporarily disable interrupt
    dedic_gpio_bundle_set_interrupt_and_callback(row_bundle, (1 << mkbd->nr_row_gpios) - 1, DEDIC_GPIO_INTR_NONE, NULL, NULL);
    // get row id, start to check the col id
    dedic_gpio_bundle_write(row_bundle, 1 << row_index, 0);
    dedic_gpio_bundle_write(mkbd->col_bundle, (1 << mkbd->nr_col_gpios) - 1, (1 << mkbd->nr_col_gpios) - 1);
    xTimerStartFromISR(mkbd->debounce_timer, &high_task_wakeup);
    return high_task_wakeup == pdTRUE;
}

static void matrix_kbd_debounce_timer_callback(TimerHandle_t xTimer)
{
    matrix_kbd_t *mkbd = (matrix_kbd_t *)pvTimerGetTimerID(xTimer);

    uint32_t row_out = dedic_gpio_bundle_read_out(mkbd->row_bundle);
    uint32_t col_in = dedic_gpio_bundle_read_in(mkbd->col_bundle);
    row_out = (~row_out) & ((1 << mkbd->nr_row_gpios) - 1);
    ESP_LOGD(TAG, "row_out=%"PRIx32", col_in=%"PRIx32, row_out, col_in);
    int row = -1;
    int col = -1;
    uint32_t key_code = 0;
    while (row_out) {
        row = __builtin_ffs(row_out) - 1;
        uint32_t changed_col_bits = mkbd->row_state[row] ^ col_in;
        while (changed_col_bits) {
            col = __builtin_ffs(changed_col_bits) - 1;
            ESP_LOGD(TAG, "row=%d, col=%d", row, col);
            key_code = MAKE_KEY_CODE(row, col);
            if (col_in & (1 << col)) {
                mkbd->event_handler(mkbd, MATRIX_KBD_EVENT_UP, (void *)key_code, mkbd->event_handler_args);
            } else {
                mkbd->event_handler(mkbd, MATRIX_KBD_EVENT_DOWN, (void *)key_code, mkbd->event_handler_args);
            }
            changed_col_bits = changed_col_bits & (changed_col_bits - 1);
        }
        mkbd->row_state[row] = col_in;
        row_out = row_out & (row_out - 1);
    }

    // row lines set to high level
    dedic_gpio_bundle_write(mkbd->row_bundle, (1 << mkbd->nr_row_gpios) - 1, (1 << mkbd->nr_row_gpios) - 1);
    // col lines set to low level
    dedic_gpio_bundle_write(mkbd->col_bundle, (1 << mkbd->nr_col_gpios) - 1, 0);
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd->row_bundle, (1 << mkbd->nr_row_gpios) - 1,
                                                 DEDIC_GPIO_INTR_BOTH_EDGE, matrix_kbd_row_isr_callback, mkbd);
}

esp_err_t matrix_kbd_install(const matrix_kbd_config_t *config, matrix_kbd_handle_t *mkbd_handle)
{
    esp_err_t ret = ESP_OK;
    matrix_kbd_t *mkbd = NULL;
    ESP_RETURN_ON_FALSE(config && mkbd_handle, ESP_ERR_INVALID_ARG, TAG, "invalid argument");

    mkbd = calloc(1, sizeof(matrix_kbd_t) + (config->nr_row_gpios) * sizeof(uint32_t));
    ESP_RETURN_ON_FALSE(mkbd, ESP_ERR_NO_MEM, TAG, "no mem for matrix keyboard context");

    // Create a ont-shot os timer, used for key debounce
    mkbd->debounce_timer = xTimerCreate("kb_debounce", pdMS_TO_TICKS(config->debounce_ms), pdFALSE, mkbd, matrix_kbd_debounce_timer_callback);
    ESP_GOTO_ON_FALSE(mkbd->debounce_timer, ESP_FAIL, err, TAG, "create debounce timer failed");

    mkbd->nr_col_gpios = config->nr_col_gpios;
    mkbd->nr_row_gpios = config->nr_row_gpios;

    dedic_gpio_bundle_config_t bundle_row_config = {
        .gpio_array = config->row_gpios,
        .array_size = config->nr_row_gpios,
        // Each GPIO used in matrix key board should be able to input and output
        .flags = {
            .in_en = 1,
            .out_en = 1,
        },
    };
    ESP_GOTO_ON_ERROR(dedic_gpio_new_bundle(&bundle_row_config, &mkbd->row_bundle), err, TAG, "create row bundle failed");

    // In case the keyboard doesn't design a resister to pull up row/col line
    // We enable the internal pull up resister, enable Open Drain as well
    for (int i = 0; i < config->nr_row_gpios; i++) {
        gpio_pullup_en(config->row_gpios[i]);
        gpio_od_enable(config->row_gpios[i]);
    }

    dedic_gpio_bundle_config_t bundle_col_config = {
        .gpio_array = config->col_gpios,
        .array_size = config->nr_col_gpios,
        .flags = {
            .in_en = 1,
            .out_en = 1,
        },
    };
    ESP_GOTO_ON_ERROR(dedic_gpio_new_bundle(&bundle_col_config, &mkbd->col_bundle), err, TAG, "create col bundle failed");

    for (int i = 0; i < config->nr_col_gpios; i++) {
        gpio_pullup_en(config->col_gpios[i]);
        gpio_od_enable(config->col_gpios[i]);
    }

    // Disable interrupt
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd->row_bundle, (1 << config->nr_row_gpios) - 1,
                                                 DEDIC_GPIO_INTR_NONE, NULL, NULL);
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd->col_bundle, (1 << config->nr_col_gpios) - 1,
                                                 DEDIC_GPIO_INTR_NONE, NULL, NULL);

    * mkbd_handle = mkbd;
    return ESP_OK;
err:
    if (mkbd->debounce_timer) {
        xTimerDelete(mkbd->debounce_timer, 0);
    }
    if (mkbd->col_bundle) {
        dedic_gpio_del_bundle(mkbd->col_bundle);
    }
    if (mkbd->row_bundle) {
        dedic_gpio_del_bundle(mkbd->row_bundle);
    }
    free(mkbd);
    return ret;
}

esp_err_t matrix_kbd_uninstall(matrix_kbd_handle_t mkbd_handle)
{
    ESP_RETURN_ON_FALSE(mkbd_handle, ESP_ERR_INVALID_ARG, TAG, "invalid argument");
    xTimerDelete(mkbd_handle->debounce_timer, 0);
    dedic_gpio_del_bundle(mkbd_handle->col_bundle);
    dedic_gpio_del_bundle(mkbd_handle->row_bundle);
    free(mkbd_handle);
    return ESP_OK;
}

esp_err_t matrix_kbd_start(matrix_kbd_handle_t mkbd_handle)
{
    ESP_RETURN_ON_FALSE(mkbd_handle, ESP_ERR_INVALID_ARG, TAG, "invalid argument");

    // row lines set to high level
    dedic_gpio_bundle_write(mkbd_handle->row_bundle, (1 << mkbd_handle->nr_row_gpios) - 1, (1 << mkbd_handle->nr_row_gpios) - 1);
    // col lines set to low level
    dedic_gpio_bundle_write(mkbd_handle->col_bundle, (1 << mkbd_handle->nr_col_gpios) - 1, 0);

    for (int i = 0; i < mkbd_handle->nr_row_gpios; i++) {
        mkbd_handle->row_state[i] = (1 << mkbd_handle->nr_col_gpios) - 1;
    }

    // only enable row line interrupt
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd_handle->row_bundle, (1 << mkbd_handle->nr_row_gpios) - 1,
                                                 DEDIC_GPIO_INTR_BOTH_EDGE, matrix_kbd_row_isr_callback, mkbd_handle);

    return ESP_OK;
}

esp_err_t matrix_kbd_stop(matrix_kbd_handle_t mkbd_handle)
{
    ESP_RETURN_ON_FALSE(mkbd_handle, ESP_ERR_INVALID_ARG, TAG, "invalid argument");
    xTimerStop(mkbd_handle->debounce_timer, 0);

    // Disable interrupt
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd_handle->row_bundle, (1 << mkbd_handle->nr_row_gpios) - 1,
                                                 DEDIC_GPIO_INTR_NONE, NULL, NULL);
    dedic_gpio_bundle_set_interrupt_and_callback(mkbd_handle->col_bundle, (1 << mkbd_handle->nr_col_gpios) - 1,
                                                 DEDIC_GPIO_INTR_NONE, NULL, NULL);

    return ESP_OK;
}

esp_err_t matrix_kbd_register_event_handler(matrix_kbd_handle_t mkbd_handle, matrix_kbd_event_handler handler, void *args)
{
    ESP_RETURN_ON_FALSE(mkbd_handle, ESP_ERR_INVALID_ARG, TAG, "invalid argument");
    mkbd_handle->event_handler = handler;
    mkbd_handle->event_handler_args = args;
    return ESP_OK;
}


generic_gpio/CMakeLists.txt:
# The following lines of boilerplate have to be in your project's CMakeLists
# in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
# "Trim" the build. Include the minimal set of components, main, and anything it depends on.
idf_build_set_property(MINIMAL_BUILD ON)
project(generic_gpio)


generic_gpio/pytest_generic_gpio_example.py:
# SPDX-FileCopyrightText: 2022-2025 Espressif Systems (Shanghai) CO LTD
# SPDX-License-Identifier: CC0-1.0
from typing import Callable

import pytest
from pytest_embedded import Dut
from pytest_embedded_idf.utils import idf_parametrize


@pytest.mark.generic
@idf_parametrize('target', ['supported_targets'], indirect=['target'])
def test_generic_gpio_example(dut: Dut, log_minimum_free_heap_size: Callable[..., None]) -> None:
    log_minimum_free_heap_size()
    dut.expect(r'cnt: \d+')


generic_gpio/README.md:
| Supported Targets | ESP32 | ESP32-C2 | ESP32-C3 | ESP32-C5 | ESP32-C6 | ESP32-C61 | ESP32-H2 | ESP32-H21 | ESP32-P4 | ESP32-S2 | ESP32-S3 |
| ----------------- | ----- | -------- | -------- | -------- | -------- | --------- | -------- | --------- | -------- | -------- | -------- |

# Example: GPIO

(See the README.md file in the upper level 'examples' directory for more information about examples.)

This test code shows how to configure GPIO and how to use it with interruption.

## GPIO functions:

| GPIO                         | Direction | Configuration                                          |
| ---------------------------- | --------- | ------------------------------------------------------ |
| CONFIG_GPIO_OUTPUT_0         | output    |                                                        |
| CONFIG_GPIO_OUTPUT_1         | output    |                                                        |
| CONFIG_GPIO_INPUT_0          | input     | pulled up, interrupt from rising edge and falling edge |
| CONFIG_GPIO_INPUT_1          | input     | pulled up, interrupt from rising edge                  |

## Test:
 1. Connect CONFIG_GPIO_OUTPUT_0 with CONFIG_GPIO_INPUT_0
 2. Connect CONFIG_GPIO_OUTPUT_1 with CONFIG_GPIO_INPUT_1
 3. Generate pulses on CONFIG_GPIO_OUTPUT_0/1, that triggers interrupt on CONFIG_GPIO_INPUT_0/1

 **Note:** The following pin assignments are used by default, you can change them by `idf.py menuconfig` > `Example Configuration`.

|                           | CONFIG_GPIO_OUTPUT_0 | CONFIG_GPIO_OUTPUT_1 | CONFIG_GPIO_INPUT_0 | CONFIG_GPIO_INPUT_1 |
| ------------------------- | -------------------- | -------------------- | ------------------- | ------------------- |
| ESP32C2/H2/C5/C61/H21     | 8                    | 9                    | 4                   | 5                   |
| All other chips           | 18                   | 19                   | 4                   | 5                   |

## How to use example

Before project configuration and build, be sure to set the correct chip target using `idf.py set-target <chip_name>`.

### Hardware Required

* A development board with any Espressif SoC (e.g., ESP32-DevKitC, ESP-WROVER-KIT, etc.)
* A USB cable for Power supply and programming
* Some jumper wires to connect GPIOs.

### Configure the project

### Build and Flash

Build the project and flash it to the board, then run the monitor tool to view the serial output:

Run `idf.py -p PORT flash monitor` to build, flash and monitor the project.

(To exit the serial monitor, type ``Ctrl-]``.)

See the [Getting Started Guide](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/index.html) for full steps to configure and use ESP-IDF to build projects.

## Example Output

As you run the example, you will see the following log:

```
I (317) gpio: GPIO[18]| InputEn: 0| OutputEn: 1| OpenDrain: 0| Pullup: 0| Pulldown: 0| Intr:0
I (327) gpio: GPIO[19]| InputEn: 0| OutputEn: 1| OpenDrain: 0| Pullup: 0| Pulldown: 0| Intr:0
I (337) gpio: GPIO[4]| InputEn: 1| OutputEn: 0| OpenDrain: 0| Pullup: 1| Pulldown: 0| Intr:1
I (347) gpio: GPIO[5]| InputEn: 1| OutputEn: 0| OpenDrain: 0| Pullup: 1| Pulldown: 0| Intr:1
Minimum free heap size: 289892 bytes
cnt: 0
cnt: 1
GPIO[4] intr, val: 1
GPIO[5] intr, val: 1
cnt: 2
GPIO[4] intr, val: 0
cnt: 3
GPIO[4] intr, val: 1
GPIO[5] intr, val: 1
cnt: 4
GPIO[4] intr, val: 0
cnt: 5
GPIO[4] intr, val: 1
GPIO[5] intr, val: 1
cnt: 6
GPIO[4] intr, val: 0
cnt: 7
GPIO[4] intr, val: 1
GPIO[5] intr, val: 1
cnt: 8
GPIO[4] intr, val: 0
cnt: 9
GPIO[4] intr, val: 1
GPIO[5] intr, val: 1
cnt: 10
...
```

## Troubleshooting

For any technical queries, please open an [issue](https://github.com/espressif/esp-idf/issues) on GitHub. We will get back to you soon.


generic_gpio/main/CMakeLists.txt:
idf_component_register(SRCS "gpio_example_main.c"
                    INCLUDE_DIRS ".")


generic_gpio/main/gpio_example_main.c:
/*
 * SPDX-FileCopyrightText: 2020-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"

/**
 * Brief:
 * This test code shows how to configure gpio and how to use gpio interrupt.
 *
 * GPIO status:
 * GPIO18: output (ESP32C2/ESP32H2 uses GPIO8 as the second output pin)
 * GPIO19: output (ESP32C2/ESP32H2 uses GPIO9 as the second output pin)
 * GPIO4:  input, pulled up, interrupt from rising edge and falling edge
 * GPIO5:  input, pulled up, interrupt from rising edge.
 *
 * Note. These are the default GPIO pins to be used in the example. You can
 * change IO pins in menuconfig.
 *
 * Test:
 * Connect GPIO18(8) with GPIO4
 * Connect GPIO19(9) with GPIO5
 * Generate pulses on GPIO18(8)/19(9), that triggers interrupt on GPIO4/5
 *
 */

#define GPIO_OUTPUT_IO_0    CONFIG_GPIO_OUTPUT_0
#define GPIO_OUTPUT_IO_1    CONFIG_GPIO_OUTPUT_1
#define GPIO_OUTPUT_PIN_SEL  ((1ULL<<GPIO_OUTPUT_IO_0) | (1ULL<<GPIO_OUTPUT_IO_1))
/*
 * Let's say, GPIO_OUTPUT_IO_0=18, GPIO_OUTPUT_IO_1=19
 * In binary representation,
 * 1ULL<<GPIO_OUTPUT_IO_0 is equal to 0000000000000000000001000000000000000000 and
 * 1ULL<<GPIO_OUTPUT_IO_1 is equal to 0000000000000000000010000000000000000000
 * GPIO_OUTPUT_PIN_SEL                0000000000000000000011000000000000000000
 * */
#define GPIO_INPUT_IO_0     CONFIG_GPIO_INPUT_0
#define GPIO_INPUT_IO_1     CONFIG_GPIO_INPUT_1
#define GPIO_INPUT_PIN_SEL  ((1ULL<<GPIO_INPUT_IO_0) | (1ULL<<GPIO_INPUT_IO_1))
/*
 * Let's say, GPIO_INPUT_IO_0=4, GPIO_INPUT_IO_1=5
 * In binary representation,
 * 1ULL<<GPIO_INPUT_IO_0 is equal to 0000000000000000000000000000000000010000 and
 * 1ULL<<GPIO_INPUT_IO_1 is equal to 0000000000000000000000000000000000100000
 * GPIO_INPUT_PIN_SEL                0000000000000000000000000000000000110000
 * */
#define ESP_INTR_FLAG_DEFAULT 0

static QueueHandle_t gpio_evt_queue = NULL;

static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    uint32_t gpio_num = (uint32_t) arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
}

static void gpio_task_example(void* arg)
{
    uint32_t io_num;
    for (;;) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            printf("GPIO[%"PRIu32"] intr, val: %d\n", io_num, gpio_get_level(io_num));
        }
    }
}

void app_main(void)
{
    //zero-initialize the config structure.
    gpio_config_t io_conf = {};
    //disable interrupt
    io_conf.intr_type = GPIO_INTR_DISABLE;
    //set as output mode
    io_conf.mode = GPIO_MODE_OUTPUT;
    //bit mask of the pins that you want to set,e.g.GPIO18/19
    io_conf.pin_bit_mask = GPIO_OUTPUT_PIN_SEL;
    //disable pull-down mode
    io_conf.pull_down_en = 0;
    //disable pull-up mode
    io_conf.pull_up_en = 0;
    //configure GPIO with the given settings
    gpio_config(&io_conf);

    //interrupt of rising edge
    io_conf.intr_type = GPIO_INTR_POSEDGE;
    //bit mask of the pins, use GPIO4/5 here
    io_conf.pin_bit_mask = GPIO_INPUT_PIN_SEL;
    //set as input mode
    io_conf.mode = GPIO_MODE_INPUT;
    //enable pull-up mode
    io_conf.pull_up_en = 1;
    gpio_config(&io_conf);

    //change gpio interrupt type for one pin
    gpio_set_intr_type(GPIO_INPUT_IO_0, GPIO_INTR_ANYEDGE);

    //create a queue to handle gpio event from isr
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));
    //start gpio task
    xTaskCreate(gpio_task_example, "gpio_task_example", 2048, NULL, 10, NULL);

    //install gpio isr service
    gpio_install_isr_service(ESP_INTR_FLAG_DEFAULT);
    //hook isr handler for specific gpio pin
    gpio_isr_handler_add(GPIO_INPUT_IO_0, gpio_isr_handler, (void*) GPIO_INPUT_IO_0);
    //hook isr handler for specific gpio pin
    gpio_isr_handler_add(GPIO_INPUT_IO_1, gpio_isr_handler, (void*) GPIO_INPUT_IO_1);

    //remove isr handler for gpio number.
    gpio_isr_handler_remove(GPIO_INPUT_IO_0);
    //hook isr handler for specific gpio pin again
    gpio_isr_handler_add(GPIO_INPUT_IO_0, gpio_isr_handler, (void*) GPIO_INPUT_IO_0);

    printf("Minimum free heap size: %"PRIu32" bytes\n", esp_get_minimum_free_heap_size());

    int cnt = 0;
    while (1) {
        printf("cnt: %d\n", cnt++);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
        gpio_set_level(GPIO_OUTPUT_IO_0, cnt % 2);
        gpio_set_level(GPIO_OUTPUT_IO_1, cnt % 2);
    }
}


generic_gpio/main/Kconfig.projbuild:
menu "Example Configuration"

    orsource "$IDF_PATH/examples/common_components/env_caps/$IDF_TARGET/Kconfig.env_caps"

    config GPIO_OUTPUT_0
        int "GPIO output pin 0"
        range ENV_GPIO_RANGE_MIN ENV_GPIO_OUT_RANGE_MAX
        default 8 if IDF_TARGET_ESP32C2 || IDF_TARGET_ESP32H2 || IDF_TARGET_ESP32C5 || IDF_TARGET_ESP32C61
        default 8 if IDF_TARGET_ESP32H21
        default 18
        help
            GPIO pin number to be used as GPIO_OUTPUT_IO_0.

    config GPIO_OUTPUT_1
        int "GPIO output pin 1"
        range ENV_GPIO_RANGE_MIN ENV_GPIO_OUT_RANGE_MAX
        default 9 if IDF_TARGET_ESP32C2 || IDF_TARGET_ESP32H2 || IDF_TARGET_ESP32C5 || IDF_TARGET_ESP32C61
        default 9 if IDF_TARGET_ESP32H21
        default 19
        help
            GPIO pin number to be used as GPIO_OUTPUT_IO_1.

    config GPIO_INPUT_0
        int "GPIO input pin 0"
        range ENV_GPIO_RANGE_MIN ENV_GPIO_IN_RANGE_MAX
        default 4
        help
            GPIO pin number to be used as GPIO_INPUT_IO_0.

    config GPIO_INPUT_1
        int "GPIO input pin 1"
        range ENV_GPIO_RANGE_MIN ENV_GPIO_IN_RANGE_MAX
        default 5
        help
            GPIO pin number to be used as GPIO_INPUT_IO_1.

endmenu