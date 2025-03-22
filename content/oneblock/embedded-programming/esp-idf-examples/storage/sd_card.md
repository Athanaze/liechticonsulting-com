sdspi/pytest_sdspi_card_example.py:
# SPDX-FileCopyrightText: 2022-2025 Espressif Systems (Shanghai) CO LTD
# SPDX-License-Identifier: Unlicense OR CC0-1.0
import logging
import re

import pytest
from pytest_embedded import Dut
from pytest_embedded_idf.utils import idf_parametrize


@pytest.mark.temp_skip_ci(targets=['esp32c61'], reason='C5 C61 GPSPI same, so testing on C5 is enough')
@pytest.mark.sdcard_spimode
@idf_parametrize('target', ['esp32', 'esp32s3', 'esp32c3', 'esp32p4', 'esp32c5'], indirect=['target'])
def test_examples_sd_card_sdspi(dut: Dut) -> None:
    dut.expect('example: Initializing SD card', timeout=20)
    dut.expect('example: Using SPI peripheral', timeout=20)

    # Provide enough time for possible SD card formatting
    dut.expect('Filesystem mounted', timeout=180)

    # These lines are matched separately because of ASCII color codes in the output
    name = dut.expect(re.compile(rb'Name: (\w+)\r'), timeout=20).group(1).decode()
    _type = dut.expect(re.compile(rb'Type: (\S+)'), timeout=20).group(1).decode()
    speed = dut.expect(re.compile(rb'Speed: (\S+)'), timeout=20).group(1).decode()
    size = dut.expect(re.compile(rb'Size: (\S+)'), timeout=20).group(1).decode()

    logging.info('Card {} {} {}MHz {} found'.format(name, _type, speed, size))

    message_list1 = (
        'Opening file /sdcard/hello.txt',
        'File written',
        'Renaming file /sdcard/hello.txt to /sdcard/foo.txt',
        'Reading file /sdcard/foo.txt',
        "Read from file: 'Hello {}!'".format(name),
    )
    sd_card_format = re.compile(str.encode('Formatting card, allocation unit size=\\S+'))
    message_list2 = (
        "file doesn't exist, formatting done",
        'Opening file /sdcard/nihao.txt',
        'File written',
        'Reading file /sdcard/nihao.txt',
        "Read from file: 'Nihao {}!'".format(name),
        'Card unmounted',
    )

    for msg in message_list1:
        dut.expect_exact(msg, timeout=30)
    dut.expect(sd_card_format, timeout=180)  # Provide enough time for SD card FATFS format operation
    for msg in message_list2:
        dut.expect_exact(msg, timeout=180)


sdspi/sdkconfig.defaults:
CONFIG_FATFS_VFS_FSTAT_BLKSIZE=4096


sdspi/CMakeLists.txt:
# The following lines of boilerplate have to be in your project's CMakeLists
# in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
# "Trim" the build. Include the minimal set of components, main, and anything it depends on.
idf_build_set_property(MINIMAL_BUILD ON)
project(sd_card)


sdspi/sdkconfig.ci:
CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED=y
CONFIG_FATFS_VFS_FSTAT_BLKSIZE=4096
CONFIG_ESP_TASK_WDT_EN=n
CONFIG_EXAMPLE_FORMAT_SD_CARD=y


sdspi/README.md:
| Supported Targets | ESP32 | ESP32-C2 | ESP32-C3 | ESP32-C5 | ESP32-C6 | ESP32-C61 | ESP32-H2 | ESP32-P4 | ESP32-S2 | ESP32-S3 |
| ----------------- | ----- | -------- | -------- | -------- | -------- | --------- | -------- | -------- | -------- | -------- |

# SD Card example (SDSPI)

(See the README.md file in the upper level 'examples' directory for more information about examples.)

__WARNING:__ This example can potentially delete all data from your SD card (when formatting is enabled). Back up your data first before proceeding.

This example demonstrates how to use an SD card with an ESP device over an SPI interface. Example does the following steps:

1. Use an "all-in-one" `esp_vfs_fat_sdspi_mount` function to:
    - initialize SDSPI peripheral,
    - probe and initialize the card connected to SPI bus (DMA channel 1, MOSI, MISO and CLK lines, chip-specific SPI host id),
    - mount FAT filesystem using FATFS library (and format card, if the filesystem cannot be mounted),
    - register FAT filesystem in VFS, enabling C standard library and POSIX functions to be used.
1. Print information about the card, such as name, type, capacity, and maximum supported frequency.
1. Create a file using `fopen` and write to it using `fprintf`.
1. Rename the file. Before renaming, check if destination file already exists using `stat` function, and remove it using `unlink` function.
1. Open renamed file for reading, read back the line, and print it to the terminal.
1. __OPTIONAL:__ Format the SD card, check if the file doesn't exist anymore.

This example support SD (SDSC, SDHC, SDXC) cards.

## Hardware

This example requires a development board with an SD card socket and and SD card.

Although it is possible to connect an SD card breakout adapter, keep in mind that connections using breakout cables are often unreliable and have poor signal integrity. You may need to use lower clock frequency when working with SD card breakout adapters.

It is recommended to get familiar with [the document about pullup requirements](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sd_pullup_requirements.html) to understand Pullup/down resistor support and compatibility of various ESP modules and development boards.

### Pin assignments

The GPIO pin numbers used to connect an SD card can be customized. This can be done in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open "SD SPI Example Configuration" menu.
2. In the source code: See the initialization of ``spi_bus_config_t`` and ``sdspi_device_config_t`` structures in the example code.

This example doesn't utilize card detect (CD) and write protect (WP) signals from SD card slot.

The table below shows the default pin assignments.

SD card pin | SPI pin | ESP32 pin     | ESP32-S2, ESP32-S3 | ESP32-P4 | ESP32-H2 | ESP32-C3 and other chips |  Notes
------------|---------|---------------|--------------------|----------|----------|--------------------------|------------
 D0         | MISO    | GPIO2         | GPIO37             | GPIO13   | GPIO0    | GPIO6                    |
 D3         | CS      | GPIO13 (MTCK) | GPIO34             | GPIO10   | GPIO1    | GPIO1                    |
 CLK        | SCK     | GPIO14 (MTMS) | GPIO36             | GPIO12   | GPIO4    | GPIO5                    |
 CMD        | MOSI    | GPIO15 (MTDO) | GPIO35             | GPIO11   | GPIO5    | GPIO4                    | 10k pullup


#### ESP32 related notes

With the default pin assignments, this example runs on ESP-WROVER-KIT boards without any extra modifications required. Only the SD card needs to be inserted into the slot.

For other development boards, adjust the pin assignments as explained above.

Some boards require specific manipulation to enable UART Download mode (GPIO2 low) - eg ESP32-Azure IoT Kit needs KEY_IO0 pressed down for the time of firmware flashing operation (sets IO0 and IO2 low). See troubleshooting section for more details

#### ESP32-S2 and ESP32-S3 related notes

With the default pin assignments, this example is compatible ESP32-S2-USB-OTG and ESP32-S3-USB-OTG development boards.

For other development boards, adjust the pin assignments as explained above.

#### ESP32-P4 related notes

On ESP32-P4, Slot 1 of the SDMMC peripheral is connected to GPIO pins using GPIO matrix. This allows arbitrary GPIOs to be used to connect an SD card. In this example, GPIOs can be configured in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open `SD SPI Example Configuration` menu.
2. In the source code: See the initialization of `sdmmc_slot_config_t slot_config` structure in the example code.

Default pins for SDSPI are listed in the table above [Pin assignments](#1-pin-assignments) and using them doesn't require any additional settings.

However on some development boards the SD card slot can be wired to default dedicated pins for SDMMC, which are listed in the table below.

SD card pin | ESP32-P4 pin 
------------|--------------     
D0  (MISO)  | GPIO39
D3  (CS)    | GPIO42
CLK (SCK)   | GPIO43
CMD (MOSI)  | GPIO44

These pins are able to connect to an ultra high-speed SD card (UHS-I) which requires 1.8V switching (instead of the regular 3.3V). This means the user has to provide an external LDO power supply to use them, or to enable and configure an internal LDO via `idf.py menuconfig` -> `SD/MMC Example Configuration` -> `SD power supply comes from internal LDO IO`.

When using different GPIO pins this is not required and `SD power supply comes from internal LDO IO` setting can be disabled.

#### Notes for ESP32-C3 and other chips

Espressif doesn't offer development boards with an SD card slot for these chips. Please check the pin assignments and adjust them for your board if necessary. The process to change pin assignments is described above.

### Build and flash

Build the project and flash it to the board, then run monitor tool to view serial output:

```
idf.py -p PORT flash monitor
```

(Replace PORT with serial port name.)

(To exit the serial monitor, type ``Ctrl-]``.)

See the Getting Started Guide for full steps to configure and use ESP-IDF to build projects.


## Example output

Here is an example console output. In this case a 64GB SDHC card was connected, and `EXAMPLE_FORMAT_IF_MOUNT_FAILED` menuconfig option enabled. Card was unformatted, so the initial mount has failed. Card was then partitioned, formatted, and mounted again.

```
I (336) example: Initializing SD card
I (336) example: Using SPI peripheral
I (336) gpio: GPIO[13]| InputEn: 0| OutputEn: 1| OpenDrain: 0| Pullup: 0| Pulldown: 0| Intr:0
W (596) vfs_fat_sdmmc: failed to mount card (13)
W (596) vfs_fat_sdmmc: partitioning card
W (596) vfs_fat_sdmmc: formatting card, allocation unit size=16384
W (7386) vfs_fat_sdmmc: mounting again
Name: XA0E5
Type: SDHC/SDXC
Speed: 20 MHz
Size: 61068MB
I (7386) example: Opening file /sdcard/hello.txt
I (7396) example: File written
I (7396) example: Renaming file /sdcard/hello.txt to /sdcard/foo.txt
I (7396) example: Reading file /sdcard/foo.txt
I (7396) example: Read from file: 'Hello XA0E5!'
I (7396) example: Card unmounted
```

## Troubleshooting

### Failure to mount filesystem

> The following error message is printed: `example: Failed to mount filesystem. If you want the card to be formatted, set the CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.`

The example will be able to mount only cards formatted using FAT32 filesystem. If the card is formatted as exFAT or some other filesystem, you have an option to format it in the example code. Enable the `CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED` menuconfig option, then build and flash the example.

> Once you've enabled the `CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED` option, if you continue to encounter the following error:

```
E (600) sdmmc_cmd: sdmmc_read_sectors_dma: sdmmc_send_cmd returned 0x108
E (600) diskio_sdmmc: sdmmc_read_blocks failed (264)
W (610) vfs_fat_sdmmc: failed to mount card (1)
E (610) vfs_fat_sdmmc: mount_to_vfs failed (0xffffffff).
I (620) gpio: GPIO[13]| InputEn: 1| OutputEn: 0| OpenDrain: 0| Pullup: 0| Pulldown: 0| Intr:0
E (630) example: Failed to mount filesystem. If you want the card to be formatted, set the CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.
```

Please ensure that your SD card is operational and not experiencing any malfunctions.


### Unable to flash the example, or serial port not available (ESP32 only)

> After the first successful flashing of the example firmware, it is not possible to flash again. Download mode not activated when running `idf.py flash` or the board's serial port disappears completely.

Some ESP32 boards require specific handling to activate the download mode after a system reset, due to GPIO2 pin now being used as both SDSPI (MISO) and an active-low bootstrapping signal for entering download mode. For instance, the ESP32-Azure IoT Kit requires KEY_IO0 button to remain pressed during whole firmware flashing operation, as it sets both GPIO0 and GPIO2 signals low.

Check you board documentation/schematics for appropriate procedure.

An attempt to download a new firmware under this conditions may also result in the board's serial port disappearing from your PC device list - rebooting your computer should fix the issue. After your device is back, use

`esptool --port PORT --before no_reset --baud 115200 --chip esp32 erase_flash`

to erase your board's flash, then flash the firmware again.

> If you insert an SD card into the slot and encounter issues when attempting to flash a supported target using the `idf.py flash` command, please consider removing the SD card and attempting to flash the target again. If the flashing process succeeds after removing the SD card, it suggests potential issues with power supply.

Ensure that the board and SD card adapter you are using are powered using the appropriate power source.


### Getting the following errors

> `vfs_fat_sdmmc: slot init failed (0x103)`

> `vfs_fat_sdmmc: sdmmc_card_init failed (0x102)`

> `sdmmc_init_ocr: send_op_cond (1) returned 0x107`

Attempt to reboot the board. This error may occur if you reset the ESP board or host controller without power-cycling it. In such cases, the card may remain in its previous state, causing it to potentially not respond to commands sent by the host controller.

Additionally, if the example works with certain SD cards but encounters issues with others, please confirm the read/write speed of the SD card. If the card is not compatible with the host frequency, consider lowering the host frequency and then attempting the operation again.

### Debug SD connections and pullup strength

> If the initialization of the SD card fails, initially follow the above options. If the issue persists, confirm the connection of pullups to the SD pins. To do this, enable the` Debug sd pin connections and pullup strength` option from menuconfig and rerun the code. This will provide the following result:

```
**** PIN recovery time ****

PIN 14 CLK   10049 cycles
PIN 15 MOSI  10034 cycles
PIN  2 MISO  10034 cycles
PIN 13 CS    10034 cycles

**** PIN recovery time with weak pullup ****

PIN 14 CLK   100 cycles
PIN 15 MOSI  100 cycles
PIN  2 MISO  100 cycles
PIN 13 CS    100 cycles

**** PIN voltage levels ****

PIN 14 CLK   0.6V
PIN 15 MOSI  0.4V
PIN  2 MISO  0.7V
PIN 13 CS    0.9V

**** PIN voltage levels with weak pullup ****

PIN 14 CLK   0.9V
PIN 15 MOSI  1.0V
PIN  2 MISO  1.0V
PIN 13 CS    1.2V

**** PIN cross-talk ****

             CLK    MOSI   MISO   CS
PIN 14 CLK    --    0.2V   0.2V   0.2V
PIN 15 MOSI  0.1V    --    0.1V   0.1V
PIN  2 MISO  0.1V   0.1V    --    0.1V
PIN 13 CS    0.1V   0.1V   0.1V    --

**** PIN cross-talk with weak pullup ****

             CLK    MOSI   MISO   CS
PIN 14 CLK    --    1.0V   1.1V   1.2V
PIN 15 MOSI  0.9V    --    1.0V   1.2V
PIN  2 MISO  0.9V   1.0V    --    1.2V
PIN 13 CS    0.9V   1.1V   1.0V    --

```
In the absence of connected pullups and having the weak pullups enabled, you can assess the pullup connections by comparing PIN recovery time measured in CPU cycles. To check pullup connections, configure the pin as open drain, set it to low state, and count the cpu cycles consumed before returning to high state. If a pullup is connected, the pin will get back to high state after reasonably small cycle count, typically around 50-300 cycles, depending on pullup strength. If no pullup is connected, the PIN stays low and the measurement times out after 10000 cycles.

It will also provide the voltage levels at the corresponding SD pins. By default, this information is provided for ESP32 chip only, and for other chipsets, verify the availability of ADC pins for the respective GPIO using [this](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/gpio.html#gpio-summary) and configure ADC mapped pins using menuconfig. Then test the voltage levels accordingly.

You can monitor the voltage levels of individual pins using `PIN voltage levels` and `PIN voltage levels with weak pullup`. However, if one pin being pulled low and experiencing interference with another pin, you can detect it through `PIN cross-talk` and `PIN cross-talk with weak pullup`. In the absence of pullups, voltage levels at each pin should range from 0 to 0.3V. With 10k pullups connected, the voltage will be between 3.1V to 3.3V, contingent on the connection between ADC pins and SD pins, and with weak pullups connected, it can fluctuate between 0.8V to 1.2V, depending on pullup strength.


sdspi/main/CMakeLists.txt:
set(srcs "sd_card_example_main.c")

idf_component_register(SRCS ${srcs}
                       INCLUDE_DIRS "."
                       REQUIRES fatfs sd_card
                       WHOLE_ARCHIVE)


sdspi/main/sd_card_example_main.c:
/* SD card and FAT filesystem example.
   This example uses SPI peripheral to communicate with SD card.

   This example code is in the Public Domain (or CC0 licensed, at your option.)

   Unless required by applicable law or agreed to in writing, this
   software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
   CONDITIONS OF ANY KIND, either express or implied.
*/

#include <string.h>
#include <sys/unistd.h>
#include <sys/stat.h>
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "sd_test_io.h"
#if SOC_SDMMC_IO_POWER_EXTERNAL
#include "sd_pwr_ctrl_by_on_chip_ldo.h"
#endif

#define EXAMPLE_MAX_CHAR_SIZE    64

static const char *TAG = "example";

#define MOUNT_POINT "/sdcard"

#ifdef CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS
const char* names[] = {"CLK ", "MOSI", "MISO", "CS  "};
const int pins[] = {CONFIG_EXAMPLE_PIN_CLK,
                    CONFIG_EXAMPLE_PIN_MOSI,
                    CONFIG_EXAMPLE_PIN_MISO,
                    CONFIG_EXAMPLE_PIN_CS};

const int pin_count = sizeof(pins)/sizeof(pins[0]);
#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
const int adc_channels[] = {CONFIG_EXAMPLE_ADC_PIN_CLK,
                            CONFIG_EXAMPLE_ADC_PIN_MOSI,
                            CONFIG_EXAMPLE_ADC_PIN_MISO,
                            CONFIG_EXAMPLE_ADC_PIN_CS};
#endif //CONFIG_EXAMPLE_ENABLE_ADC_FEATURE

pin_configuration_t config = {
    .names = names,
    .pins = pins,
#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
    .adc_channels = adc_channels,
#endif
};
#endif //CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS

// Pin assignments can be set in menuconfig, see "SD SPI Example Configuration" menu.
// You can also change the pin assignments here by changing the following 4 lines.
#define PIN_NUM_MISO  CONFIG_EXAMPLE_PIN_MISO
#define PIN_NUM_MOSI  CONFIG_EXAMPLE_PIN_MOSI
#define PIN_NUM_CLK   CONFIG_EXAMPLE_PIN_CLK
#define PIN_NUM_CS    CONFIG_EXAMPLE_PIN_CS

static esp_err_t s_example_write_file(const char *path, char *data)
{
    ESP_LOGI(TAG, "Opening file %s", path);
    FILE *f = fopen(path, "w");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        return ESP_FAIL;
    }
    fprintf(f, data);
    fclose(f);
    ESP_LOGI(TAG, "File written");

    return ESP_OK;
}

static esp_err_t s_example_read_file(const char *path)
{
    ESP_LOGI(TAG, "Reading file %s", path);
    FILE *f = fopen(path, "r");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for reading");
        return ESP_FAIL;
    }
    char line[EXAMPLE_MAX_CHAR_SIZE];
    fgets(line, sizeof(line), f);
    fclose(f);

    // strip newline
    char *pos = strchr(line, '\n');
    if (pos) {
        *pos = '\0';
    }
    ESP_LOGI(TAG, "Read from file: '%s'", line);

    return ESP_OK;
}

void app_main(void)
{
    esp_err_t ret;

    // Options for mounting the filesystem.
    // If format_if_mount_failed is set to true, SD card will be partitioned and
    // formatted in case when mounting fails.
    esp_vfs_fat_sdmmc_mount_config_t mount_config = {
#ifdef CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED
        .format_if_mount_failed = true,
#else
        .format_if_mount_failed = false,
#endif // EXAMPLE_FORMAT_IF_MOUNT_FAILED
        .max_files = 5,
        .allocation_unit_size = 16 * 1024
    };
    sdmmc_card_t *card;
    const char mount_point[] = MOUNT_POINT;
    ESP_LOGI(TAG, "Initializing SD card");

    // Use settings defined above to initialize SD card and mount FAT filesystem.
    // Note: esp_vfs_fat_sdmmc/sdspi_mount is all-in-one convenience functions.
    // Please check its source code and implement error recovery when developing
    // production applications.
    ESP_LOGI(TAG, "Using SPI peripheral");

    // By default, SD card frequency is initialized to SDMMC_FREQ_DEFAULT (20MHz)
    // For setting a specific frequency, use host.max_freq_khz (range 400kHz - 20MHz for SDSPI)
    // Example: for fixed frequency of 10MHz, use host.max_freq_khz = 10000;
    sdmmc_host_t host = SDSPI_HOST_DEFAULT();

    // For SoCs where the SD power can be supplied both via an internal or external (e.g. on-board LDO) power supply.
    // When using specific IO pins (which can be used for ultra high-speed SDMMC) to connect to the SD card
    // and the internal LDO power supply, we need to initialize the power supply first.
#if CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
    sd_pwr_ctrl_ldo_config_t ldo_config = {
        .ldo_chan_id = CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_IO_ID,
    };
    sd_pwr_ctrl_handle_t pwr_ctrl_handle = NULL;

    ret = sd_pwr_ctrl_new_on_chip_ldo(&ldo_config, &pwr_ctrl_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to create a new on-chip LDO power control driver");
        return;
    }
    host.pwr_ctrl_handle = pwr_ctrl_handle;
#endif

    spi_bus_config_t bus_cfg = {
        .mosi_io_num = PIN_NUM_MOSI,
        .miso_io_num = PIN_NUM_MISO,
        .sclk_io_num = PIN_NUM_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 4000,
    };

    ret = spi_bus_initialize(host.slot, &bus_cfg, SDSPI_DEFAULT_DMA);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to initialize bus.");
        return;
    }

    // This initializes the slot without card detect (CD) and write protect (WP) signals.
    // Modify slot_config.gpio_cd and slot_config.gpio_wp if your board has these signals.
    sdspi_device_config_t slot_config = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot_config.gpio_cs = PIN_NUM_CS;
    slot_config.host_id = host.slot;

    ESP_LOGI(TAG, "Mounting filesystem");
    ret = esp_vfs_fat_sdspi_mount(mount_point, &host, &slot_config, &mount_config, &card);

    if (ret != ESP_OK) {
        if (ret == ESP_FAIL) {
            ESP_LOGE(TAG, "Failed to mount filesystem. "
                     "If you want the card to be formatted, set the CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.");
        } else {
            ESP_LOGE(TAG, "Failed to initialize the card (%s). "
                     "Make sure SD card lines have pull-up resistors in place.", esp_err_to_name(ret));
#ifdef CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS
            check_sd_card_pins(&config, pin_count);
#endif
        }
        return;
    }
    ESP_LOGI(TAG, "Filesystem mounted");

    // Card has been initialized, print its properties
    sdmmc_card_print_info(stdout, card);

    // Use POSIX and C standard library functions to work with files.

    // First create a file.
    const char *file_hello = MOUNT_POINT"/hello.txt";
    char data[EXAMPLE_MAX_CHAR_SIZE];
    snprintf(data, EXAMPLE_MAX_CHAR_SIZE, "%s %s!\n", "Hello", card->cid.name);
    ret = s_example_write_file(file_hello, data);
    if (ret != ESP_OK) {
        return;
    }

    const char *file_foo = MOUNT_POINT"/foo.txt";

    // Check if destination file exists before renaming
    struct stat st;
    if (stat(file_foo, &st) == 0) {
        // Delete it if it exists
        unlink(file_foo);
    }

    // Rename original file
    ESP_LOGI(TAG, "Renaming file %s to %s", file_hello, file_foo);
    if (rename(file_hello, file_foo) != 0) {
        ESP_LOGE(TAG, "Rename failed");
        return;
    }

    ret = s_example_read_file(file_foo);
    if (ret != ESP_OK) {
        return;
    }

    // Format FATFS
#ifdef CONFIG_EXAMPLE_FORMAT_SD_CARD
    ret = esp_vfs_fat_sdcard_format(mount_point, card);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to format FATFS (%s)", esp_err_to_name(ret));
        return;
    }

    if (stat(file_foo, &st) == 0) {
        ESP_LOGI(TAG, "file still exists");
        return;
    } else {
        ESP_LOGI(TAG, "file doesn't exist, formatting done");
    }
#endif // CONFIG_EXAMPLE_FORMAT_SD_CARD

    const char *file_nihao = MOUNT_POINT"/nihao.txt";
    memset(data, 0, EXAMPLE_MAX_CHAR_SIZE);
    snprintf(data, EXAMPLE_MAX_CHAR_SIZE, "%s %s!\n", "Nihao", card->cid.name);
    ret = s_example_write_file(file_nihao, data);
    if (ret != ESP_OK) {
        return;
    }

    //Open file for reading
    ret = s_example_read_file(file_nihao);
    if (ret != ESP_OK) {
        return;
    }

    // All done, unmount partition and disable SPI peripheral
    esp_vfs_fat_sdcard_unmount(mount_point, card);
    ESP_LOGI(TAG, "Card unmounted");

    //deinitialize the bus after all devices are removed
    spi_bus_free(host.slot);

    // Deinitialize the power control driver if it was used
#if CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
    ret = sd_pwr_ctrl_del_on_chip_ldo(pwr_ctrl_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to delete the on-chip LDO power control driver");
        return;
    }
#endif
}


sdspi/main/idf_component.yml:
dependencies:
  sd_card:
    path: ${IDF_PATH}/examples/storage/sd_card/sdmmc/components/sd_card


sdspi/main/Kconfig.projbuild:
menu "SD SPI Example Configuration"

    config EXAMPLE_FORMAT_IF_MOUNT_FAILED
        bool "Format the card if mount failed"
        default n
        help
            If this config item is set, format_if_mount_failed will be set to true and the card will be formatted if
            the mount has failed.

    config EXAMPLE_FORMAT_SD_CARD
        bool "Format the card as a part of the example"
        default n
        help
            If this config item is set, the card will be formatted as a part of the example.

    config EXAMPLE_PIN_MOSI
        int "MOSI GPIO number"
        default 15 if IDF_TARGET_ESP32
        default 35 if IDF_TARGET_ESP32S2
        default 4 if IDF_TARGET_ESP32S3
        default 5  if IDF_TARGET_ESP32H2
        default 36 if IDF_TARGET_ESP32P4
        default 4  # C3 and others

    config EXAMPLE_PIN_MISO
        int "MISO GPIO number"
        default 2 if IDF_TARGET_ESP32
        default 37 if IDF_TARGET_ESP32S2
        default 5 if IDF_TARGET_ESP32S3
        default 0  if IDF_TARGET_ESP32H2
        default 47 if IDF_TARGET_ESP32P4
        default 6  # C3 and others

    config EXAMPLE_PIN_CLK
        int "CLK GPIO number"
        default 14 if IDF_TARGET_ESP32
        default 36 if IDF_TARGET_ESP32S2
        default 2 if IDF_TARGET_ESP32S3
        default 4  if IDF_TARGET_ESP32H2
        default 53 if IDF_TARGET_ESP32P4
        default 5  # C3 and others

    config EXAMPLE_PIN_CS
        int "CS GPIO number"
        default 13 if IDF_TARGET_ESP32
        default 34 if IDF_TARGET_ESP32S2
        default 8 if IDF_TARGET_ESP32S3
        default 33 if IDF_TARGET_ESP32P4
        default 1  # C3 and others

    config EXAMPLE_DEBUG_PIN_CONNECTIONS
        bool "Debug sd pin connections and pullup strength"
        default n

    config EXAMPLE_ENABLE_ADC_FEATURE
        bool "Enable ADC feature"
        depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
        default y if IDF_TARGET_ESP32
        default n

    config EXAMPLE_ADC_UNIT
        int "ADC Unit"
        depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
        default 1 if IDF_TARGET_ESP32
        default 0 if IDF_TARGET_ESP32S3
        default 1

    config EXAMPLE_ADC_PIN_MOSI
        int "MOSI mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 3 if IDF_TARGET_ESP32
        default 7 if IDF_TARGET_ESP32S3
        default 1

    config EXAMPLE_ADC_PIN_MISO
        int "MISO mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 2 if IDF_TARGET_ESP32
        default 1 if IDF_TARGET_ESP32S3
        default 1

    config EXAMPLE_ADC_PIN_CLK
        int "CLK mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 6 if IDF_TARGET_ESP32
        default 0 if IDF_TARGET_ESP32S3
        default 1

    config EXAMPLE_ADC_PIN_CS
        int "CS mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 4 if IDF_TARGET_ESP32
        default 6 if IDF_TARGET_ESP32S3
        default 1

    config EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
        depends on SOC_SDMMC_IO_POWER_EXTERNAL
        bool "SD power supply comes from internal LDO IO (READ HELP!)"
        default n
        help
            Only needed when the SD card is connected to specific IO pins which can be used for high-speed SDMMC.
            Please read the schematic first and check if the SD VDD is connected to any internal LDO output.
            Unselect this option if the SD card is powered by an external power supply.

    config EXAMPLE_SD_PWR_CTRL_LDO_IO_ID
        depends on SOC_SDMMC_IO_POWER_EXTERNAL && EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
        int "LDO ID"
        default 4 if IDF_TARGET_ESP32P4
        help
            Please read the schematic first and input your LDO ID.
endmenu


sdmmc/sdkconfig.defaults:
CONFIG_FATFS_VFS_FSTAT_BLKSIZE=4096


sdmmc/pytest_sdmmc_card_example.py:
# SPDX-FileCopyrightText: 2022-2025 Espressif Systems (Shanghai) CO LTD
# SPDX-License-Identifier: Unlicense OR CC0-1.0
import logging
import re

import pytest
from pytest_embedded import Dut
from pytest_embedded_idf.utils import idf_parametrize


@pytest.mark.sdcard_sdmode
@idf_parametrize('target', ['esp32'], indirect=['target'])
def test_examples_sd_card_sdmmc(dut: Dut) -> None:
    dut.expect('example: Initializing SD card', timeout=20)
    dut.expect('example: Using SDMMC peripheral', timeout=10)

    # Provide enough time for possible SD card formatting
    dut.expect('Filesystem mounted', timeout=60)

    # These lines are matched separately because of ASCII color codes in the output
    name = dut.expect(re.compile(rb'Name: (\w+)\r'), timeout=10).group(1).decode()
    _type = dut.expect(re.compile(rb'Type: (\S+)'), timeout=10).group(1).decode()
    speed = dut.expect(re.compile(rb'Speed: (\S+)'), timeout=10).group(1).decode()
    size = dut.expect(re.compile(rb'Size: (\S+)'), timeout=10).group(1).decode()

    logging.info('Card {} {} {}MHz {} found'.format(name, _type, speed, size))

    message_list1 = (
        'Opening file /sdcard/hello.txt',
        'File written',
        'Renaming file /sdcard/hello.txt to /sdcard/foo.txt',
        'Reading file /sdcard/foo.txt',
        "Read from file: 'Hello {}!'".format(name),
    )
    sd_card_format = re.compile(str.encode('Formatting card, allocation unit size=\\S+'))
    message_list2 = (
        "file doesn't exist, formatting done",
        'Opening file /sdcard/nihao.txt',
        'File written',
        'Reading file /sdcard/nihao.txt',
        "Read from file: 'Nihao {}!'".format(name),
        'Card unmounted',
    )

    for msg in message_list1:
        dut.expect_exact(msg, timeout=30)
    dut.expect(sd_card_format, timeout=180)  # Provide enough time for SD card FATFS format operation
    for msg in message_list2:
        dut.expect_exact(msg, timeout=30)


sdmmc/CMakeLists.txt:
# The following lines of boilerplate have to be in your project's CMakeLists
# in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
# "Trim" the build. Include the minimal set of components, main, and anything it depends on.
idf_build_set_property(MINIMAL_BUILD ON)

project(sd_card)


sdmmc/sdkconfig.ci:
CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED=y
CONFIG_FATFS_VFS_FSTAT_BLKSIZE=4096
CONFIG_ESP_TASK_WDT_EN=n
CONFIG_EXAMPLE_FORMAT_SD_CARD=y


sdmmc/README.md:
| Supported Targets | ESP32 | ESP32-P4 | ESP32-S3 |
| ----------------- | ----- | -------- | -------- |

# SD Card example (SDMMC)

(See the README.md file in the upper level 'examples' directory for more information about examples.)

__WARNING:__ This example can potentially delete all data from your SD card (when formatting is enabled). Back up your data first before proceeding.

This example demonstrates how to use an SD card with an ESP device. Example does the following steps:

1. Use an "all-in-one" `esp_vfs_fat_sdmmc_mount` function to:
    - initialize SDMMC peripheral,
    - probe and initialize an SD card,
    - mount FAT filesystem using FATFS library (and format card, if the filesystem cannot be mounted),
    - register FAT filesystem in VFS, enabling C standard library and POSIX functions to be used.
1. Print information about the card, such as name, type, capacity, and maximum supported frequency.
1. Create a file using `fopen` and write to it using `fprintf`.
1. Rename the file. Before renaming, check if destination file already exists using `stat` function, and remove it using `unlink` function.
1. Open renamed file for reading, read back the line, and print it to the terminal.
1. __OPTIONAL:__ Format the SD card, check if the file doesn't exist anymore.

This example supports SD (SDSC, SDHC, SDXC) cards and eMMC chips.

## Hardware

This example requires an ESP32 or ESP32-S3 development board with an SD card slot and an SD card.

Although it is possible to connect an SD card breakout adapter, keep in mind that connections using breakout cables are often unreliable and have poor signal integrity. You may need to use lower clock frequency when working with SD card breakout adapters.

This example doesn't utilize card detect (CD) and write protect (WP) signals from SD card slot.

### Pin assignments for ESP32

On ESP32, SDMMC peripheral is connected to specific GPIO pins using the IO MUX. GPIO pins cannot be customized. Please see the table below for the pin connections.

When using an ESP-WROVER-KIT board, this example runs without any extra modifications required. Only an SD card needs to be inserted into the slot.

ESP32 pin     | SD card pin | Notes
--------------|-------------|------------
GPIO14 (MTMS) | CLK         | 10k pullup in SD mode
GPIO15 (MTDO) | CMD         | 10k pullup in SD mode
GPIO2         | D0          | 10k pullup in SD mode, pull low to go into download mode (see Note about GPIO2 below!)
GPIO4         | D1          | not used in 1-line SD mode; 10k pullup in 4-line SD mode
GPIO12 (MTDI) | D2          | not used in 1-line SD mode; 10k pullup in 4-line SD mode (see Note about GPIO12 below!)
GPIO13 (MTCK) | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup


### Pin assignments for ESP32-S3

On ESP32-S3, SDMMC peripheral is connected to GPIO pins using GPIO matrix. This allows arbitrary GPIOs to be used to connect an SD card. In this example, GPIOs can be configured in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open "SD/MMC Example Configuration" menu.
2. In the source code: See the initialization of `sdmmc_slot_config_t slot_config` structure in the example code.

The table below lists the default pin assignments.

When using an ESP32-S3-USB-OTG board, this example runs without any extra modifications required. Only an SD card needs to be inserted into the slot.

ESP32-S3 pin  | SD card pin | Notes
--------------|-------------|------------
GPIO36        | CLK         | 10k pullup
GPIO35        | CMD         | 10k pullup
GPIO37        | D0          | 10k pullup
GPIO38        | D1          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO33        | D2          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO34        | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup

### Pin assignments for ESP32-P4

On ESP32-P4, Slot 1 of the SDMMC peripheral is connected to GPIO pins using GPIO matrix. This allows arbitrary GPIOs to be used to connect an SD card. In this example, GPIOs can be configured in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open `SD/MMC Example Configuration` menu.
2. In the source code: See the initialization of `sdmmc_slot_config_t slot_config` structure in the example code.

The table below lists the default pin assignments.

ESP32-P4 pin  | SD card pin | Notes
--------------|-------------|------------
GPIO43        | CLK         | 10k pullup
GPIO44        | CMD         | 10k pullup
GPIO39        | D0          | 10k pullup
GPIO40        | D1          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO41        | D2          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO42        | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup

Default dedicated pins on ESP32-P4 are able to connect to an ultra high-speed SD card (UHS-I) which requires 1.8V switching (instead of the regular 3.3V). This means the user has to provide an external LDO power supply to use them, or to enable and configure an internal LDO via `idf.py menuconfig` -> `SD/MMC Example Configuration` -> `SD power supply comes from internal LDO IO`.

When using different GPIO pins this is not required and `SD power supply comes from internal LDO IO` setting can be disabled.

### 4-line and 1-line SD modes

By default, this example uses 4 line SD mode, utilizing 6 pins: CLK, CMD, D0 - D3. It is possible to use 1-line mode (CLK, CMD, D0) by changing "SD/MMC bus width" in the example configuration menu (see `CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_1`).

Note that even if card's D3 line is not connected to the ESP chip, it still has to be pulled up, otherwise the card will go into SPI protocol mode.

### Note about GPIO2 (ESP32 only)

GPIO2 pin is used as a bootstrapping pin, and should be low to enter UART download mode. One way to do this is to connect GPIO0 and GPIO2 using a jumper, and then the auto-reset circuit on most development boards will pull GPIO2 low along with GPIO0, when entering download mode.

- Some boards have pulldown and/or LED on GPIO2. LED is usually ok, but pulldown will interfere with D0 signals and must be removed. Check the schematic of your development board for anything connected to GPIO2.

### Note about GPIO12 (ESP32 only)

GPIO12 is used as a bootstrapping pin to select output voltage of an internal regulator which powers the flash chip (VDD_SDIO). This pin has an internal pulldown so if left unconnected it will read low at reset (selecting default 3.3V operation). When adding a pullup to this pin for SD card operation, consider the following:

- For boards which don't use the internal regulator (VDD_SDIO) to power the flash, GPIO12 can be pulled high.
- For boards which use 1.8V flash chip, GPIO12 needs to be pulled high at reset. This is fully compatible with SD card operation.
- On boards which use the internal regulator and a 3.3V flash chip, GPIO12 must be low at reset. This is incompatible with SD card operation.
    * In most cases, external pullup can be omitted and an internal pullup can be enabled using a `gpio_pullup_en(GPIO_NUM_12);` call. Most SD cards work fine when an internal pullup on GPIO12 line is enabled. Note that if ESP32 experiences a power-on reset while the SD card is sending data, high level on GPIO12 can be latched into the bootstrapping register, and ESP32 will enter a boot loop until external reset with correct GPIO12 level is applied.
    * Another option is to burn the flash voltage selection efuses. This will permanently select 3.3V output voltage for the internal regulator, and GPIO12 will not be used as a bootstrapping pin. Then it is safe to connect a pullup resistor to GPIO12. This option is suggested for production use.

The following command can be used to program flash voltage selection efuses **to 3.3V**:

```sh
    components/esptool_py/esptool/espefuse.py set_flash_voltage 3.3V
```

This command will burn the `XPD_SDIO_TIEH`, `XPD_SDIO_FORCE`, and `XPD_SDIO_REG` efuses. With all three burned to value 1, the internal VDD_SDIO flash voltage regulator is permanently enabled at 3.3V. See the technical reference manual for more details.

`espefuse.py` has a `--do-not-confirm` option if running from an automated flashing script.

See [the document about pullup requirements](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sd_pullup_requirements.html) for more details about pullup support and compatibility of modules and development boards.

## How to use example

### Build and flash

Build the project and flash it to the board, then run monitor tool to view serial output:

```
idf.py -p PORT flash monitor
```

(Replace PORT with serial port name.)

(To exit the serial monitor, type ``Ctrl-]``.)

See the Getting Started Guide for full steps to configure and use ESP-IDF to build projects.


## Example output

Here is an example console output. In this case a 128MB SDSC card was connected, and `EXAMPLE_FORMAT_IF_MOUNT_FAILED` menuconfig option enabled. Card was unformatted, so the initial mount has failed. Card was then partitioned, formatted, and mounted again.

```
I (336) example: Initializing SD card
I (336) example: Using SDMMC peripheral
I (336) gpio: GPIO[13]| InputEn: 0| OutputEn: 1| OpenDrain: 0| Pullup: 0| Pulldown: 0| Intr:0
W (596) vfs_fat_sdmmc: failed to mount card (13)
W (596) vfs_fat_sdmmc: partitioning card
W (596) vfs_fat_sdmmc: formatting card, allocation unit size=16384
W (7386) vfs_fat_sdmmc: mounting again
Name: XA0E5
Type: SDHC/SDXC
Speed: 20 MHz
Size: 61068MB
I (7386) example: Opening file /sdcard/hello.txt
I (7396) example: File written
I (7396) example: Renaming file /sdcard/hello.txt to /sdcard/foo.txt
I (7396) example: Reading file /sdcard/foo.txt
I (7396) example: Read from file: 'Hello XA0E5!'
I (7396) example: Card unmounted
```

## Troubleshooting

### Failure to download the example

```
Connecting........_____....._____....._____....._____....._____....._____....._____

A fatal error occurred: Failed to connect to Espressif device: Invalid head of packet (0x34)
```

Disconnect the SD card D0/MISO line from GPIO2 and try uploading again. Read "Note about GPIO2" above.

### Card fails to initialize with `sdmmc_init_sd_scr: send_scr (1) returned 0x107` error

Check connections between the card and the ESP32. For example, if you have disconnected GPIO2 to work around the flashing issue, connect it back and reset the ESP32 (using a button on the development board, or by pressing Ctrl-T Ctrl-R in IDF Monitor).

### Card fails to initialize with `sdmmc_check_scr: send_scr returned 0xffffffff` error

Connections between the card and the ESP32 are too long for the frequency used. Try using shorter connections, or try reducing the clock speed of SD interface.

### Failure to mount filesystem

```
example: Failed to mount filesystem. If you want the card to be formatted, set the EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.
```
The example will be able to mount only cards formatted using FAT32 filesystem. If the card is formatted as exFAT or some other filesystem, you have an option to format it in the example code. Enable the `EXAMPLE_FORMAT_IF_MOUNT_FAILED` menuconfig option, then build and flash the example.

### Debug SD connections and pullup strength

> If the initialization of the SD card fails, initially follow the above options. If the issue persists, confirm the connection of pullups to the SD pins. To do this, enable the` Debug sd pin connections and pullup strength` option from menuconfig and rerun the code. This will provide the following result:

```
**** PIN recovery time ****

PIN 14 CLK  10044 cycles
PIN 15 CMD  10034 cycles
PIN  2  D0  10034 cycles
PIN  4  D1  10034 cycles
PIN 12  D2  10034 cycles
PIN 13  D3  10034 cycles

**** PIN recovery time with weak pullup ****

PIN 14 CLK  100 cycles
PIN 15 CMD  100 cycles
PIN  2  D0  100 cycles
PIN  4  D1  100 cycles
PIN 12  D2  100 cycles
PIN 13  D3  100 cycles

**** PIN voltage levels ****

PIN 14 CLK  0.6V
PIN 15 CMD  0.3V
PIN  2  D0  0.8V
PIN  4  D1  0.6V
PIN 12  D2  0.4V
PIN 13  D3  0.8V

**** PIN voltage levels with weak pullup ****

PIN 14 CLK  1.0V
PIN 15 CMD  1.1V
PIN  2  D0  1.0V
PIN  4  D1  1.0V
PIN 12  D2  1.0V
PIN 13  D3  1.2V

**** PIN cross-talk ****

              CLK   CMD    D0    D1    D2    D3
PIN 14 CLK     --   0.2V  0.1V  0.1V  0.1V  0.2V
PIN 15 CMD    0.1V   --   0.1V  0.1V  0.1V  0.1V
PIN  2  D0    0.1V  0.1V   --   0.2V  0.1V  0.1V
PIN  4  D1    0.1V  0.1V  0.3V   --   0.1V  0.1V
PIN 12  D2    0.1V  0.2V  0.2V  0.1V   --   0.1V
PIN 13  D3    0.1V  0.2V  0.1V  0.1V  0.1V   --

**** PIN cross-talk with weak pullup ****

              CLK   CMD    D0    D1    D2    D3
PIN 14 CLK     --   1.0V  1.0V  1.0V  1.0V  1.2V
PIN 15 CMD    0.9V   --   1.0V  1.0V  1.0V  1.2V
PIN  2  D0    0.9V  1.0V   --   1.0V  1.0V  1.2V
PIN  4  D1    0.9V  1.0V  1.2V   --   1.0V  1.2V
PIN 12  D2    0.9V  1.1V  1.2V  0.9V   --   1.2V
PIN 13  D3    0.9V  1.2V  1.1V  0.9V  0.9V   --
I (845) main_task: Returned from app_main()
```

In the absence of connected pullups and having the weak pullups enabled, you can assess the pullup connections by comparing PIN recovery time measured in CPU cycles. To check pullup connections, configure the pin as open drain, set it to low state, and count the cpu cycles consumed before returning to high state. If a pullup is connected, the pin will get back to high state after reasonably small cycle count, typically around 50-300 cycles, depending on pullup strength. If no pullup is connected, the PIN stays low and the measurement times out after 10000 cycles.

It will also provide the voltage levels at the corresponding SD pins. By default, this information is provided for ESP32 chip only, and for other chipsets, verify the availability of ADC pins for the respective GPIO using [this](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/gpio.html#gpio-summary) and configure ADC mapped pins using menuconfig. Then test the voltage levels accordingly.

You can monitor the voltage levels of individual pins using `PIN voltage levels` and `PIN voltage levels with weak pullup`. However, if one pin being pulled low and experiencing interference with another pin, you can detect it through `PIN cross-talk` and `PIN cross-talk with weak pullup`. In the absence of pullups, voltage levels at each pin should range from 0 to 0.3V. With 10k pullups connected, the voltage will be between 3.1V to 3.3V, contingent on the connection between ADC pins and SD pins, and with weak pullups connected, it can fluctuate between 0.8V to 1.2V, depending on pullup strength.

sdmmc/main/CMakeLists.txt:
set(srcs "sd_card_example_main.c")

idf_component_register(SRCS ${srcs}
                       INCLUDE_DIRS "."
                       REQUIRES fatfs sd_card
                       WHOLE_ARCHIVE)

if(NOT CONFIG_SOC_SDMMC_HOST_SUPPORTED)
    fail_at_build_time(sdmmc ""
                             "Only ESP32 and ESP32-S3 targets are supported."
                             "Please refer README.md for more details")
endif()


sdmmc/main/sd_card_example_main.c:
/* SD card and FAT filesystem example.
   This example code is in the Public Domain (or CC0 licensed, at your option.)

   Unless required by applicable law or agreed to in writing, this
   software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
   CONDITIONS OF ANY KIND, either express or implied.
*/

// This example uses SDMMC peripheral to communicate with SD card.

#include <string.h>
#include <sys/unistd.h>
#include <sys/stat.h>
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "driver/sdmmc_host.h"
#include "sd_test_io.h"
#if SOC_SDMMC_IO_POWER_EXTERNAL
#include "sd_pwr_ctrl_by_on_chip_ldo.h"
#endif

#define EXAMPLE_MAX_CHAR_SIZE    64

static const char *TAG = "example";

#define MOUNT_POINT "/sdcard"
#define EXAMPLE_IS_UHS1    (CONFIG_EXAMPLE_SDMMC_SPEED_UHS_I_SDR50 || CONFIG_EXAMPLE_SDMMC_SPEED_UHS_I_DDR50)

#ifdef CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS
const char* names[] = {"CLK", "CMD", "D0", "D1", "D2", "D3"};
const int pins[] = {CONFIG_EXAMPLE_PIN_CLK,
                    CONFIG_EXAMPLE_PIN_CMD,
                    CONFIG_EXAMPLE_PIN_D0
                    #ifdef CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4
                    ,CONFIG_EXAMPLE_PIN_D1,
                    CONFIG_EXAMPLE_PIN_D2,
                    CONFIG_EXAMPLE_PIN_D3
                    #endif
                    };

const int pin_count = sizeof(pins)/sizeof(pins[0]);

#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
const int adc_channels[] = {CONFIG_EXAMPLE_ADC_PIN_CLK,
                            CONFIG_EXAMPLE_ADC_PIN_CMD,
                            CONFIG_EXAMPLE_ADC_PIN_D0
                            #ifdef CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4
                            ,CONFIG_EXAMPLE_ADC_PIN_D1,
                            CONFIG_EXAMPLE_ADC_PIN_D2,
                            CONFIG_EXAMPLE_ADC_PIN_D3
                            #endif
                            };
#endif //CONFIG_EXAMPLE_ENABLE_ADC_FEATURE

pin_configuration_t config = {
    .names = names,
    .pins = pins,
#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
    .adc_channels = adc_channels,
#endif
};
#endif //CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS

static esp_err_t s_example_write_file(const char *path, char *data)
{
    ESP_LOGI(TAG, "Opening file %s", path);
    FILE *f = fopen(path, "w");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        return ESP_FAIL;
    }
    fprintf(f, data);
    fclose(f);
    ESP_LOGI(TAG, "File written");

    return ESP_OK;
}

static esp_err_t s_example_read_file(const char *path)
{
    ESP_LOGI(TAG, "Reading file %s", path);
    FILE *f = fopen(path, "r");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for reading");
        return ESP_FAIL;
    }
    char line[EXAMPLE_MAX_CHAR_SIZE];
    fgets(line, sizeof(line), f);
    fclose(f);

    // strip newline
    char *pos = strchr(line, '\n');
    if (pos) {
        *pos = '\0';
    }
    ESP_LOGI(TAG, "Read from file: '%s'", line);

    return ESP_OK;
}

void app_main(void)
{
    esp_err_t ret;

    // Options for mounting the filesystem.
    // If format_if_mount_failed is set to true, SD card will be partitioned and
    // formatted in case when mounting fails.
    esp_vfs_fat_sdmmc_mount_config_t mount_config = {
#ifdef CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED
        .format_if_mount_failed = true,
#else
        .format_if_mount_failed = false,
#endif // EXAMPLE_FORMAT_IF_MOUNT_FAILED
        .max_files = 5,
        .allocation_unit_size = 16 * 1024
    };
    sdmmc_card_t *card;
    const char mount_point[] = MOUNT_POINT;
    ESP_LOGI(TAG, "Initializing SD card");

    // Use settings defined above to initialize SD card and mount FAT filesystem.
    // Note: esp_vfs_fat_sdmmc/sdspi_mount is all-in-one convenience functions.
    // Please check its source code and implement error recovery when developing
    // production applications.

    ESP_LOGI(TAG, "Using SDMMC peripheral");

    // By default, SD card frequency is initialized to SDMMC_FREQ_DEFAULT (20MHz)
    // For setting a specific frequency, use host.max_freq_khz (range 400kHz - 40MHz for SDMMC)
    // Example: for fixed frequency of 10MHz, use host.max_freq_khz = 10000;
    sdmmc_host_t host = SDMMC_HOST_DEFAULT();
#if CONFIG_EXAMPLE_SDMMC_SPEED_HS
    host.max_freq_khz = SDMMC_FREQ_HIGHSPEED;
#elif CONFIG_EXAMPLE_SDMMC_SPEED_UHS_I_SDR50
    host.slot = SDMMC_HOST_SLOT_0;
    host.max_freq_khz = SDMMC_FREQ_SDR50;
    host.flags &= ~SDMMC_HOST_FLAG_DDR;
#elif CONFIG_EXAMPLE_SDMMC_SPEED_UHS_I_DDR50
    host.slot = SDMMC_HOST_SLOT_0;
    host.max_freq_khz = SDMMC_FREQ_DDR50;
#endif

    // For SoCs where the SD power can be supplied both via an internal or external (e.g. on-board LDO) power supply.
    // When using specific IO pins (which can be used for ultra high-speed SDMMC) to connect to the SD card
    // and the internal LDO power supply, we need to initialize the power supply first.
#if CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
    sd_pwr_ctrl_ldo_config_t ldo_config = {
        .ldo_chan_id = CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_IO_ID,
    };
    sd_pwr_ctrl_handle_t pwr_ctrl_handle = NULL;

    ret = sd_pwr_ctrl_new_on_chip_ldo(&ldo_config, &pwr_ctrl_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to create a new on-chip LDO power control driver");
        return;
    }
    host.pwr_ctrl_handle = pwr_ctrl_handle;
#endif

    // This initializes the slot without card detect (CD) and write protect (WP) signals.
    // Modify slot_config.gpio_cd and slot_config.gpio_wp if your board has these signals.
    sdmmc_slot_config_t slot_config = SDMMC_SLOT_CONFIG_DEFAULT();
#if EXAMPLE_IS_UHS1
    slot_config.flags |= SDMMC_SLOT_FLAG_UHS1;
#endif

    // Set bus width to use:
#ifdef CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4
    slot_config.width = 4;
#else
    slot_config.width = 1;
#endif

    // On chips where the GPIOs used for SD card can be configured, set them in
    // the slot_config structure:
#ifdef CONFIG_SOC_SDMMC_USE_GPIO_MATRIX
    slot_config.clk = CONFIG_EXAMPLE_PIN_CLK;
    slot_config.cmd = CONFIG_EXAMPLE_PIN_CMD;
    slot_config.d0 = CONFIG_EXAMPLE_PIN_D0;
#ifdef CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4
    slot_config.d1 = CONFIG_EXAMPLE_PIN_D1;
    slot_config.d2 = CONFIG_EXAMPLE_PIN_D2;
    slot_config.d3 = CONFIG_EXAMPLE_PIN_D3;
#endif  // CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4
#endif  // CONFIG_SOC_SDMMC_USE_GPIO_MATRIX

    // Enable internal pullups on enabled pins. The internal pullups
    // are insufficient however, please make sure 10k external pullups are
    // connected on the bus. This is for debug / example purpose only.
    slot_config.flags |= SDMMC_SLOT_FLAG_INTERNAL_PULLUP;

    ESP_LOGI(TAG, "Mounting filesystem");
    ret = esp_vfs_fat_sdmmc_mount(mount_point, &host, &slot_config, &mount_config, &card);

    if (ret != ESP_OK) {
        if (ret == ESP_FAIL) {
            ESP_LOGE(TAG, "Failed to mount filesystem. "
                     "If you want the card to be formatted, set the EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.");
        } else {
            ESP_LOGE(TAG, "Failed to initialize the card (%s). "
                     "Make sure SD card lines have pull-up resistors in place.", esp_err_to_name(ret));
#ifdef CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS
            check_sd_card_pins(&config, pin_count);
#endif
        }
        return;
    }
    ESP_LOGI(TAG, "Filesystem mounted");

    // Card has been initialized, print its properties
    sdmmc_card_print_info(stdout, card);

    // Use POSIX and C standard library functions to work with files:

    // First create a file.
    const char *file_hello = MOUNT_POINT"/hello.txt";
    char data[EXAMPLE_MAX_CHAR_SIZE];
    snprintf(data, EXAMPLE_MAX_CHAR_SIZE, "%s %s!\n", "Hello", card->cid.name);
    ret = s_example_write_file(file_hello, data);
    if (ret != ESP_OK) {
        return;
    }

    const char *file_foo = MOUNT_POINT"/foo.txt";
    // Check if destination file exists before renaming
    struct stat st;
    if (stat(file_foo, &st) == 0) {
        // Delete it if it exists
        unlink(file_foo);
    }

    // Rename original file
    ESP_LOGI(TAG, "Renaming file %s to %s", file_hello, file_foo);
    if (rename(file_hello, file_foo) != 0) {
        ESP_LOGE(TAG, "Rename failed");
        return;
    }

    ret = s_example_read_file(file_foo);
    if (ret != ESP_OK) {
        return;
    }

    // Format FATFS
#ifdef CONFIG_EXAMPLE_FORMAT_SD_CARD
    ret = esp_vfs_fat_sdcard_format(mount_point, card);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to format FATFS (%s)", esp_err_to_name(ret));
        return;
    }

    if (stat(file_foo, &st) == 0) {
        ESP_LOGI(TAG, "file still exists");
        return;
    } else {
        ESP_LOGI(TAG, "file doesn't exist, formatting done");
    }
#endif // CONFIG_EXAMPLE_FORMAT_SD_CARD

    const char *file_nihao = MOUNT_POINT"/nihao.txt";
    memset(data, 0, EXAMPLE_MAX_CHAR_SIZE);
    snprintf(data, EXAMPLE_MAX_CHAR_SIZE, "%s %s!\n", "Nihao", card->cid.name);
    ret = s_example_write_file(file_nihao, data);
    if (ret != ESP_OK) {
        return;
    }

    //Open file for reading
    ret = s_example_read_file(file_nihao);
    if (ret != ESP_OK) {
        return;
    }

    // All done, unmount partition and disable SDMMC peripheral
    esp_vfs_fat_sdcard_unmount(mount_point, card);
    ESP_LOGI(TAG, "Card unmounted");

    // Deinitialize the power control driver if it was used
#if CONFIG_EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
    ret = sd_pwr_ctrl_del_on_chip_ldo(pwr_ctrl_handle);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to delete the on-chip LDO power control driver");
        return;
    }
#endif
}


sdmmc/main/Kconfig.projbuild:
menu "SD/MMC Example Configuration"

    config EXAMPLE_FORMAT_IF_MOUNT_FAILED
        bool "Format the card if mount failed"
        default n
        help
            If this config item is set, format_if_mount_failed will be set to true and the card will be formatted if
            the mount has failed.

    config EXAMPLE_FORMAT_SD_CARD
        bool "Format the card as a part of the example"
        default n
        help
            If this config item is set, the card will be formatted as a part of the example.

    choice EXAMPLE_SDMMC_BUS_WIDTH
        prompt "SD/MMC bus width"
        default EXAMPLE_SDMMC_BUS_WIDTH_4
        help
            Select the bus width of SD or MMC interface.
            Note that even if 1 line mode is used, D3 pin of the SD card must have a pull-up resistor connected.
            Otherwise the card may enter SPI mode, the only way to recover from which is to cycle power to the card.

        config EXAMPLE_SDMMC_BUS_WIDTH_4
            bool "4 lines (D0 - D3)"

        config EXAMPLE_SDMMC_BUS_WIDTH_1
            bool "1 line (D0)"
    endchoice

    choice EXAMPLE_SDMMC_SPEED_MODE
        prompt "SD/MMC speed mode"
        default EXAMPLE_SDMMC_SPEED_DS

        config EXAMPLE_SDMMC_SPEED_DS
            bool "Default Speed"
        config EXAMPLE_SDMMC_SPEED_HS
            bool "High Speed"
        config EXAMPLE_SDMMC_SPEED_UHS_I_SDR50
            bool "UHS-I SDR50 (100 MHz, 50 MB/s)"
            depends on SOC_SDMMC_UHS_I_SUPPORTED
        config EXAMPLE_SDMMC_SPEED_UHS_I_DDR50
            bool "UHS-I DDR50 (50 MHz, 50 MB/s)"
            depends on SOC_SDMMC_UHS_I_SUPPORTED
    endchoice

    if SOC_SDMMC_USE_GPIO_MATRIX

        config EXAMPLE_PIN_CMD
            int "CMD GPIO number"
            default 35 if IDF_TARGET_ESP32S3
            default 44 if IDF_TARGET_ESP32P4

        config EXAMPLE_PIN_CLK
            int "CLK GPIO number"
            default 36 if IDF_TARGET_ESP32S3
            default 43 if IDF_TARGET_ESP32P4

        config EXAMPLE_PIN_D0
            int "D0 GPIO number"
            default 37 if IDF_TARGET_ESP32S3
            default 39 if IDF_TARGET_ESP32P4

        if EXAMPLE_SDMMC_BUS_WIDTH_4

            config EXAMPLE_PIN_D1
                int "D1 GPIO number"
                default 38 if IDF_TARGET_ESP32S3
                default 40 if IDF_TARGET_ESP32P4

            config EXAMPLE_PIN_D2
                int "D2 GPIO number"
                default 33 if IDF_TARGET_ESP32S3
                default 41 if IDF_TARGET_ESP32P4

            config EXAMPLE_PIN_D3
                int "D3 GPIO number"
                default 34 if IDF_TARGET_ESP32S3
                default 42 if IDF_TARGET_ESP32P4

        endif  # EXAMPLE_SDMMC_BUS_WIDTH_4

    endif  # SOC_SDMMC_USE_GPIO_MATRIX

    config EXAMPLE_DEBUG_PIN_CONNECTIONS
        bool "Debug sd pin connections and pullup strength"
        default n

    if !SOC_SDMMC_USE_GPIO_MATRIX
        config EXAMPLE_PIN_CMD
            int
            depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
            default 15 if IDF_TARGET_ESP32

        config EXAMPLE_PIN_CLK
            int
            depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
            default 14 if IDF_TARGET_ESP32

        config EXAMPLE_PIN_D0
            int
            depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
            default 2 if IDF_TARGET_ESP32

        if EXAMPLE_SDMMC_BUS_WIDTH_4

            config EXAMPLE_PIN_D1
                int
                depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
                default 4 if IDF_TARGET_ESP32

            config EXAMPLE_PIN_D2
                int
                depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
                default 12 if IDF_TARGET_ESP32

            config EXAMPLE_PIN_D3
                int
                depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
                default 13 if IDF_TARGET_ESP32

        endif  # EXAMPLE_SDMMC_BUS_WIDTH_4
    endif

    config EXAMPLE_ENABLE_ADC_FEATURE
        bool "Enable ADC feature"
        depends on EXAMPLE_DEBUG_PIN_CONNECTIONS
        default y if IDF_TARGET_ESP32
        default n

    config EXAMPLE_ADC_UNIT
        int "ADC Unit"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 1 if IDF_TARGET_ESP32
        default 1

    config EXAMPLE_ADC_PIN_CLK
        int "CLK mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 6 if IDF_TARGET_ESP32
        default 1

    config EXAMPLE_ADC_PIN_CMD
        int "CMD mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 3 if IDF_TARGET_ESP32
        default 1

    config EXAMPLE_ADC_PIN_D0
        int "D0 mapped ADC pin"
        depends on EXAMPLE_ENABLE_ADC_FEATURE
        default 2 if IDF_TARGET_ESP32
        default 1

    if EXAMPLE_SDMMC_BUS_WIDTH_4

        config EXAMPLE_ADC_PIN_D1
            int "D1 mapped ADC pin"
            depends on EXAMPLE_ENABLE_ADC_FEATURE
            default 0 if IDF_TARGET_ESP32
            default 1

        config EXAMPLE_ADC_PIN_D2
            int "D2 mapped ADC pin"
            depends on EXAMPLE_ENABLE_ADC_FEATURE
            default 5 if IDF_TARGET_ESP32
            default 1

        config EXAMPLE_ADC_PIN_D3
            int "D3 mapped ADC pin"
            depends on EXAMPLE_ENABLE_ADC_FEATURE
            default 4 if IDF_TARGET_ESP32
            default 1

    endif  # EXAMPLE_SDMMC_BUS_WIDTH_4

    config EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
        depends on SOC_SDMMC_IO_POWER_EXTERNAL
        bool "SD power supply comes from internal LDO IO (READ HELP!)"
        default y
        help
            Only needed when the SD card is connected to specific IO pins which can be used for high-speed SDMMC.
            Please read the schematic first and check if the SD VDD is connected to any internal LDO output.
            Unselect this option if the SD card is powered by an external power supply.

    config EXAMPLE_SD_PWR_CTRL_LDO_IO_ID
        depends on SOC_SDMMC_IO_POWER_EXTERNAL && EXAMPLE_SD_PWR_CTRL_LDO_INTERNAL_IO
        int "LDO ID"
        default 4 if IDF_TARGET_ESP32P4
        help
            Please read the schematic first and input your LDO ID.
endmenu


sdmmc/components/sd_card/CMakeLists.txt:
idf_component_register(SRCS "sd_test_io.c"
                       INCLUDE_DIRS "."
                       REQUIRES fatfs esp_adc
                       WHOLE_ARCHIVE)


sdmmc/components/sd_card/sd_test_io.h:
/*
 * SPDX-FileCopyrightText: 2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Unlicense OR CC0-1.0
 */

#pragma once

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    const char** names;
    const int* pins;
#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
    const int *adc_channels;
#endif
} pin_configuration_t;


void check_sd_card_pins(pin_configuration_t *config, const int pin_count);

#ifdef __cplusplus
}
#endif


sdmmc/components/sd_card/sd_test_io.c:
/*
 * SPDX-FileCopyrightText: 2023-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Unlicense OR CC0-1.0
 */
#include <stdio.h>
#include <inttypes.h>
#include <unistd.h>
#include "esp_adc/adc_oneshot.h"
#include "driver/gpio.h"
#include "esp_cpu.h"
#include "esp_log.h"
#include "sd_test_io.h"

const static char *TAG = "SD_TEST";

#define ADC_ATTEN_DB              ADC_ATTEN_DB_12
#define GPIO_INPUT_PIN_SEL(pin)   (1ULL<<pin)

#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
static bool adc_calibration_init(adc_unit_t unit, adc_channel_t channel, adc_atten_t atten, adc_cali_handle_t *out_handle)
{
    adc_cali_handle_t handle = NULL;
    esp_err_t ret = ESP_FAIL;
    bool calibrated = false;

#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    if (!calibrated) {
        adc_cali_curve_fitting_config_t cali_config = {
            .unit_id = unit,
            .chan = channel,
            .atten = atten,
            .bitwidth = ADC_BITWIDTH_DEFAULT,
        };
        ret = adc_cali_create_scheme_curve_fitting(&cali_config, &handle);
        if (ret == ESP_OK) {
            calibrated = true;
        }
    }
#endif //ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED

#if ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED
    if (!calibrated) {
        adc_cali_line_fitting_config_t cali_config = {
            .unit_id = unit,
            .atten = atten,
            .bitwidth = ADC_BITWIDTH_DEFAULT,
        };
        ret = adc_cali_create_scheme_line_fitting(&cali_config, &handle);
        if (ret == ESP_OK) {
            calibrated = true;
        }
    }
#endif //ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED

    *out_handle = handle;
    if (ret != ESP_OK) {
        if (ret == ESP_ERR_NOT_SUPPORTED || !calibrated) {
            ESP_LOGW(TAG, "eFuse not burnt, skip software calibration");
        } else {
            ESP_LOGE(TAG, "Invalid arg or no memory");
        }
    }

    return calibrated;
}

static void example_adc_calibration_deinit(adc_cali_handle_t handle)
{
#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    ESP_ERROR_CHECK(adc_cali_delete_scheme_curve_fitting(handle));

#elif ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED
    ESP_ERROR_CHECK(adc_cali_delete_scheme_line_fitting(handle));
#endif
}

static float get_pin_voltage(int i, adc_oneshot_unit_handle_t adc_handle, bool do_calibration, adc_cali_handle_t adc_cali_handle)
{
    int voltage = 0;
    int val;
    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB,
    };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc_handle, i, &config));

    ESP_ERROR_CHECK(adc_oneshot_read(adc_handle, i, &val));
    if (do_calibration) {
        ESP_ERROR_CHECK(adc_cali_raw_to_voltage(adc_cali_handle, val, &voltage));
    }

    return (float)voltage/1000;
}
#endif //CONFIG_EXAMPLE_ENABLE_ADC_FEATURE

static uint32_t get_cycles_until_pin_level(int i, int level, int timeout) {
    uint32_t start = esp_cpu_get_cycle_count();
    while(gpio_get_level(i) == !level && esp_cpu_get_cycle_count() - start < timeout) {
        ;
    }
    uint32_t end = esp_cpu_get_cycle_count();
    return end - start;
}

void check_sd_card_pins(pin_configuration_t *config, const int pin_count)
{
    ESP_LOGI(TAG, "Testing SD pin connections and pullup strength");
    gpio_config_t io_conf = {};
    for (int i = 0; i < pin_count; ++i) {
        io_conf.intr_type = GPIO_INTR_DISABLE;
        io_conf.mode = GPIO_MODE_INPUT_OUTPUT_OD;
        io_conf.pin_bit_mask = GPIO_INPUT_PIN_SEL(config->pins[i]);
        io_conf.pull_down_en = 0;
        io_conf.pull_up_en = 0;
        gpio_config(&io_conf);
    }

    printf("\n**** PIN recovery time ****\n\n");

    for (int i = 0; i < pin_count; ++i) {
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT_OUTPUT_OD);
        gpio_set_level(config->pins[i], 0);
        usleep(100);
        gpio_set_level(config->pins[i], 1);
        uint32_t cycles = get_cycles_until_pin_level(config->pins[i], 1, 10000);
        printf("PIN %2d %3s  %"PRIu32" cycles\n", config->pins[i], config->names[i], cycles);
    }

    printf("\n**** PIN recovery time with weak pullup ****\n\n");

    for (int i = 0; i < pin_count; ++i) {
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT_OUTPUT_OD);
        gpio_pullup_en(config->pins[i]);
        gpio_set_level(config->pins[i], 0);
        usleep(100);
        gpio_set_level(config->pins[i], 1);
        uint32_t cycles = get_cycles_until_pin_level(config->pins[i], 1, 10000);
        printf("PIN %2d %3s  %"PRIu32" cycles\n", config->pins[i], config->names[i], cycles);
        gpio_pullup_dis(config->pins[i]);
    }

#if CONFIG_EXAMPLE_ENABLE_ADC_FEATURE

    adc_oneshot_unit_handle_t adc_handle;
    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = CONFIG_EXAMPLE_ADC_UNIT,
    };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config, &adc_handle));

    adc_cali_handle_t *adc_cali_handle = (adc_cali_handle_t *)malloc(sizeof(adc_cali_handle_t)*pin_count);
    bool *do_calibration = (bool *)malloc(sizeof(bool)*pin_count);

    for (int i = 0; i < pin_count; i++) {
        do_calibration[i] = adc_calibration_init(CONFIG_EXAMPLE_ADC_UNIT, i, ADC_ATTEN_DB, &adc_cali_handle[i]);
    }

    printf("\n**** PIN voltage levels ****\n\n");

    for (int i = 0; i < pin_count; ++i) {
        float voltage = get_pin_voltage(config->adc_channels[i], adc_handle, do_calibration[i], adc_cali_handle[i]);
        printf("PIN %2d %3s  %.1fV\n", config->pins[i], config->names[i], voltage);
    }

    printf("\n**** PIN voltage levels with weak pullup ****\n\n");

    for (int i = 0; i < pin_count; ++i) {
        gpio_pullup_en(config->pins[i]);
        usleep(100);
        float voltage = get_pin_voltage(config->adc_channels[i], adc_handle, do_calibration[i], adc_cali_handle[i]);
        printf("PIN %2d %3s  %.1fV\n", config->pins[i], config->names[i], voltage);
        gpio_pullup_dis(config->pins[i]);
    }

    printf("\n**** PIN cross-talk ****\n\n");

    printf("              ");
    for (int i = 0; i < pin_count; ++i) {
        printf("%3s   ", config->names[i]);
    }
    printf("\n");
    for (int i = 0; i < pin_count; ++i) {
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT_OUTPUT_OD);
        gpio_set_level(config->pins[i], 0);
        printf("PIN %2d %3s    ", config->pins[i], config->names[i]);
        for (int j = 0; j < pin_count; ++j) {
            if (j == i) {
                printf(" --   ");
                continue;
            }
            usleep(100);
            float voltage = get_pin_voltage(config->adc_channels[j], adc_handle, do_calibration[i], adc_cali_handle[i]);
            printf("%1.1fV  ", voltage);
        }
        printf("\n");
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT);
    }

    printf("\n**** PIN cross-talk with weak pullup ****\n\n");

    printf("              ");
    for (int i = 0; i < pin_count; ++i) {
        printf("%3s   ", config->names[i]);
    }
    printf("\n");
    for (int i = 0; i < pin_count; ++i) {
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT_OUTPUT_OD);
        gpio_set_level(config->pins[i], 0);
        printf("PIN %2d %3s    ", config->pins[i], config->names[i]);
        for (int j = 0; j < pin_count; ++j) {
            if (j == i) {
                printf(" --   ");
                continue;
            }
            gpio_pullup_en(config->pins[j]);
            usleep(100);
            float voltage = get_pin_voltage(config->adc_channels[j], adc_handle, do_calibration[i], adc_cali_handle[i]);
            printf("%1.1fV  ", voltage);
            gpio_pullup_dis(config->pins[j]);
        }
        printf("\n");
        gpio_set_direction(config->pins[i], GPIO_MODE_INPUT);
    }

    for (int i = 0; i < pin_count; ++i) {
        if (do_calibration[i]) {
            example_adc_calibration_deinit(adc_cali_handle[i]);
        }
    }
#endif //CONFIG_EXAMPLE_ENABLE_ADC_FEATURE
}


