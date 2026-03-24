---
name: zephyr-app-development
description: Use when developing Zephyr RTOS applications - project structure, build and flash commands, using Zephyr APIs for peripherals (GPIO, SPI, I2C, UART, PWM, ADC), adapting code from zephyr samples, or troubleshooting build errors
---

# Zephyr App Development

## Overview

Workflow for developing Zephyr RTOS applications using west + CMake. Covers the core development loop: project structure, building and flashing, using Zephyr APIs, and adapting sample code into your application.

**REQUIRED BACKGROUND:** You MUST understand superpowers:test-driven-development before writing application code.

## When to Use

- Starting a new Zephyr application or adding features
- Using Zephyr peripheral APIs
- Adapting code from `zephyr/samples/` into an existing app
- Build/flash command issues

**When NOT to use:**
- Kconfig questions (use zephyr-kconfig)
- Devicetree overlay questions (use zephyr-devicetree)
- Writing a new device driver (use zephyr-driver-development)
- West manifest or workspace issues (use west-workspace-management)

## Project Structure Convention

```
my-app/
├── CMakeLists.txt          # Top-level build file
├── prj.conf                # Default Kconfig (all boards)
├── boards/
│   ├── <board_name>.conf   # Board-specific Kconfig overrides
│   └── <board_name>.overlay # Board-specific DT overlay
├── src/
│   └── main.c
├── include/
│   └── app/
├── dts/
│   └── bindings/           # App-specific DT bindings (rare)
├── tests/
│   ├── unit/               # Native sim unit tests
│   └── integration/        # On-target or QEMU tests
└── west.yml                # (if app is a west workspace root)
```

## Zephyr API Patterns

### GPIO

```c
#include <zephyr/drivers/gpio.h>

static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

int init_gpio(void)
{
    if (!gpio_is_ready_dt(&led)) {
        return -ENODEV;
    }
    return gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
}

void toggle_led(void)
{
    gpio_pin_toggle_dt(&led);
}
```

### SPI

```c
#include <zephyr/drivers/spi.h>

static const struct spi_dt_spec spi_dev =
    SPI_DT_SPEC_GET(DT_NODELABEL(my_sensor),
                     SPI_OP_MODE_MASTER | SPI_WORD_SET(8), 0);

int spi_read_reg(uint8_t reg, uint8_t *val)
{
    uint8_t tx[] = {reg | 0x80, 0x00};
    uint8_t rx[2];

    struct spi_buf tx_buf = {.buf = tx, .len = sizeof(tx)};
    struct spi_buf rx_buf = {.buf = rx, .len = sizeof(rx)};
    struct spi_buf_set tx_set = {.buffers = &tx_buf, .count = 1};
    struct spi_buf_set rx_set = {.buffers = &rx_buf, .count = 1};

    int ret = spi_transceive_dt(&spi_dev, &tx_set, &rx_set);
    if (ret == 0) {
        *val = rx[1];
    }
    return ret;
}
```

### I2C

```c
#include <zephyr/drivers/i2c.h>

static const struct i2c_dt_spec i2c_dev =
    I2C_DT_SPEC_GET(DT_NODELABEL(my_i2c_device));

int i2c_read_reg(uint8_t reg, uint8_t *val)
{
    return i2c_write_read_dt(&i2c_dev, &reg, 1, val, 1);
}
```

### Logging

```c
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(my_module, CONFIG_MY_MODULE_LOG_LEVEL);

void some_function(void)
{
    LOG_INF("Initializing with value %d", val);
    LOG_WRN("Buffer nearly full: %u/%u", used, total);
    LOG_ERR("SPI transaction failed: %d", ret);
    LOG_DBG("Register 0x%02x = 0x%02x", reg, val);
    LOG_HEXDUMP_DBG(buf, len, "Raw data");
}
```

### Threads and synchronization

```c
#include <zephyr/kernel.h>

/* Work queue pattern (preferred for deferred work) */
static void my_work_handler(struct k_work *work)
{
    /* Do work outside ISR context */
}
K_WORK_DEFINE(my_work, my_work_handler);

/* From ISR or callback: */
k_work_submit(&my_work);

/* Semaphore for ISR-to-thread signaling */
K_SEM_DEFINE(data_ready, 0, 1);

void isr_callback(const struct device *dev, ...)
{
    k_sem_give(&data_ready);
}

void consumer_thread(void)
{
    while (true) {
        k_sem_take(&data_ready, K_FOREVER);
        /* Process data */
    }
}

/* Dedicated thread */
#define STACK_SIZE 1024
#define PRIORITY 5
K_THREAD_STACK_DEFINE(my_stack, STACK_SIZE);
struct k_thread my_thread_data;

void start_thread(void)
{
    k_thread_create(&my_thread_data, my_stack, STACK_SIZE,
                    my_thread_entry, NULL, NULL, NULL,
                    PRIORITY, 0, K_NO_WAIT);
}
```

## Adapting Zephyr Samples

### Process

1. **Find relevant sample:**
   ```bash
   find $ZEPHYR_BASE/samples -name "*.rst" -o -name "README*" | xargs grep -l "keyword"
   ```

2. **Read the sample's README and prj.conf** — note required Kconfig and DT requirements

3. **Copy API usage, not structure** — samples are demos, not app architecture:
   - Extract the API calls and initialization patterns
   - Adapt to your app's thread model and error handling
   - Don't copy `main()` wholesale

4. **Check board compatibility:**
   ```bash
   cat $ZEPHYR_BASE/samples/<sample>/sample.yaml
   ```

5. **Merge Kconfig:** Add sample's `prj.conf` entries to yours (see zephyr-kconfig)

6. **Merge DT requirements:** Add required nodes to your overlay (see zephyr-devicetree)

### Common pitfalls adapting samples

| Pitfall | Solution |
|---------|----------|
| Sample uses `DT_ALIAS` you don't have | Add alias to your overlay |
| Sample assumes single-threaded | Wrap in proper thread or work queue |
| Sample blocks in `main()` forever | Factor into a module with init/start |
| Sample uses board-specific node paths | Use `DT_NODELABEL` or `DT_ALIAS` instead |

## Build and Flash

```bash
# Basic build
west build -b <board> -p auto

# Build with board-specific overlay (auto-detected from boards/ dir)
west build -b nucleo_f411re -p auto

# Build with explicit overlay
west build -b <board> -p auto -- -DDTC_OVERLAY_FILE=custom.overlay

# Build with extra Kconfig
west build -b <board> -p auto -- -DEXTRA_CONF_FILE=debug.conf

# Flash
west flash

# Flash with specific runner
west flash --runner jlink
west flash --runner openocd
```

## Common Build Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Devicetree node not found` | DT_ALIAS/NODELABEL nonexistent | Check overlay, verify in `build/zephyr/zephyr.dts` |
| `undefined reference to z_impl_*` | Missing Kconfig for subsystem | Enable subsystem in `prj.conf` |
| `'CONFIG_FOO' undeclared` | Using symbol in C without guard | Use `IS_ENABLED(CONFIG_FOO)` or `#ifdef` |
| `multiple definition of` | Duplicate DT node | Check base DTS + overlay for duplicate labels |
| `not ready` on device API call | Device not init'd or missing DT config | Check `status = "okay"` and bus config |

## Your Job

1. Create the directory and write the file with EXACTLY the content above (everything between the --- frontmatter markers through the end)
2. Make sure all code fences use proper triple backticks
3. Verify the file
4. Commit
