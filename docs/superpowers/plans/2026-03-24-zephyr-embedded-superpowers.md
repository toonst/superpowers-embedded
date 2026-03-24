# Zephyr Embedded Systems Superpowers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a suite of 8 embedded systems development skills for Zephyr RTOS projects, covering app development, Kconfig, devicetree, debugging, fault diagnosis, west workspace management, TDD, and driver development.

**Architecture:** Each skill is a standalone SKILL.md (with optional supporting reference files) following the superpowers flat namespace convention. Skills are domain-specific (Zephyr/embedded) and cross-reference each other plus existing superpowers skills. Each skill has a clear type (Reference, Workflow, Discipline) and distinct trigger signals so they load only when relevant.

**Tech Stack:** Zephyr RTOS, west, CMake, Kconfig, devicetree, J-Link/OpenOCD, cortex-debug, pytest, twister

---

## Skill Suite Overview

| # | Skill Name | Type | Triggers on |
|---|-----------|------|-------------|
| 1 | `zephyr-app-development` | Workflow | Project structure, build/flash, adapting samples, Zephyr API patterns |
| 2 | `zephyr-kconfig` | Reference | Kconfig errors, symbol lookup, prj.conf, menuconfig, dependencies |
| 3 | `zephyr-devicetree` | Reference | Overlay syntax, DT macros, binding errors, node access in C |
| 4 | `embedded-debugging` | Workflow | cortex-debug setup, VSCode launch.json, west debug, J-Link/OpenOCD, GDB |
| 5 | `cortex-m-fault-diagnosis` | Reference | HardFault, MemManage, BusFault, UsageFault, CFSR, stack overflow |
| 6 | `west-workspace-management` | Reference/Workflow | Manifests, modules, version pinning, multi-repo, west update issues |
| 7 | `embedded-tdd` | Discipline | Writing tests for firmware, ztest/native_sim, pytest HIL, mocking hardware |
| 8 | `zephyr-driver-development` | Workflow | Writing drivers, DT bindings, DT_INST macros, twister, upstream PRs |

**Cross-reference map:**
```
zephyr-app-development
  ├── references: zephyr-kconfig (config patterns)
  ├── references: zephyr-devicetree (overlay patterns)
  ├── uses: west-workspace-management (build context)
  ├── uses: embedded-tdd (testing)
  └── uses: embedded-debugging (when things go wrong)

embedded-debugging
  ├── references: cortex-m-fault-diagnosis (when crash occurs)
  └── uses: superpowers:systematic-debugging (general methodology)

embedded-tdd
  └── extends: superpowers:test-driven-development (general TDD discipline)

zephyr-driver-development
  ├── references: zephyr-kconfig (driver Kconfig)
  ├── references: zephyr-devicetree (DT bindings)
  ├── uses: embedded-tdd (driver testing)
  └── uses: west-workspace-management (manifest/version)
```

---

## Chunk 1: Zephyr App Development Skill

Project structure, build/flash commands, Zephyr API quick reference, adapting samples.

### Task 1: Create SKILL.md for zephyr-app-development

**Files:**
- Create: `skills/zephyr-app-development/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/zephyr-app-development
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/zephyr-app-development/SKILL.md` with this content:

```markdown
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

` ` `
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
` ` `

## Zephyr API Patterns

### GPIO

` ` `c
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
` ` `

### SPI

` ` `c
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
` ` `

### I2C

` ` `c
#include <zephyr/drivers/i2c.h>

static const struct i2c_dt_spec i2c_dev =
    I2C_DT_SPEC_GET(DT_NODELABEL(my_i2c_device));

int i2c_read_reg(uint8_t reg, uint8_t *val)
{
    return i2c_write_read_dt(&i2c_dev, &reg, 1, val, 1);
}
` ` `

### Logging

` ` `c
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
` ` `

### Threads and synchronization

` ` `c
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
` ` `

## Adapting Zephyr Samples

### Process

1. **Find relevant sample:**
   ` ` `bash
   find $ZEPHYR_BASE/samples -name "*.rst" -o -name "README*" | xargs grep -l "keyword"
   ` ` `

2. **Read the sample's README and prj.conf** — note required Kconfig and DT requirements

3. **Copy API usage, not structure** — samples are demos, not app architecture:
   - Extract the API calls and initialization patterns
   - Adapt to your app's thread model and error handling
   - Don't copy `main()` wholesale

4. **Check board compatibility:**
   ` ` `bash
   cat $ZEPHYR_BASE/samples/<sample>/sample.yaml
   ` ` `

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

` ` `bash
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
` ` `

## Common Build Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Devicetree node not found` | DT_ALIAS/NODELABEL nonexistent | Check overlay, verify in `build/zephyr/zephyr.dts` |
| `undefined reference to z_impl_*` | Missing Kconfig for subsystem | Enable subsystem in `prj.conf` |
| `'CONFIG_FOO' undeclared` | Using symbol in C without guard | Use `IS_ENABLED(CONFIG_FOO)` or `#ifdef` |
| `multiple definition of` | Duplicate DT node | Check base DTS + overlay for duplicate labels |
| `not ready` on device API call | Device not init'd or missing DT config | Check `status = "okay"` and bus config |
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/zephyr-app-development/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/zephyr-app-development/SKILL.md
git commit -m "feat: add zephyr-app-development skill"
```

---

## Chunk 2: Zephyr Kconfig Skill

Dedicated Kconfig reference — symbol lookup, configuration patterns, debugging dependency chains.

### Task 2: Create SKILL.md for zephyr-kconfig

**Files:**
- Create: `skills/zephyr-kconfig/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/zephyr-kconfig
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/zephyr-kconfig/SKILL.md` with this content:

```markdown
---
name: zephyr-kconfig
description: Use when working with Zephyr Kconfig - prj.conf configuration, board-specific conf files, Kconfig symbol errors, dependency chains, menuconfig, finding where a CONFIG symbol is defined, or debugging why a symbol has an unexpected value
---

# Zephyr Kconfig

## Overview

Reference for Zephyr's Kconfig build configuration system. Covers where configuration lives, common patterns, custom Kconfig symbols, and debugging symbol resolution.

## When to Use

- Writing or modifying `prj.conf` or board-specific `.conf` files
- Build error mentioning `CONFIG_*` symbols
- Need to find where a Kconfig symbol is defined
- Debugging why a symbol has unexpected value
- Creating custom app-level Kconfig options

**When NOT to use:**
- Devicetree issues (use zephyr-devicetree)
- General build/flash commands (use zephyr-app-development)

## Where Kconfig Lives

| File | Purpose | Scope |
|------|---------|-------|
| `prj.conf` | Default config for all boards | Project-wide |
| `boards/<board>.conf` | Board-specific overrides | Per-board |
| `Kconfig` (app root) | App-specific custom symbols | Custom options |

**Merge order (last wins):** Board defconfig → Zephyr defaults → `prj.conf` → `boards/<board>.conf` → CMake `-DEXTRA_CONF_FILE`

## Common Patterns

` ` `kconfig
# prj.conf — enable subsystems
CONFIG_GPIO=y
CONFIG_SPI=y
CONFIG_I2C=y
CONFIG_SENSOR=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3

# Enable a specific driver
CONFIG_BME280=y

# Thread/memory configuration
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_HEAP_MEM_POOL_SIZE=4096
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
` ` `

## Custom App Kconfig

` ` `kconfig
# Kconfig (in app root)
menu "My Application"

config MY_APP_FEATURE_X
    bool "Enable feature X"
    default y
    help
      Enables the feature X subsystem.

config MY_APP_SENSOR_POLL_INTERVAL_MS
    int "Sensor polling interval in ms"
    default 1000
    range 100 60000

config MY_APP_LOG_LEVEL
    int "Log level for app modules"
    default 3
    range 0 4

endmenu
` ` `

**Use in C:**
` ` `c
#if IS_ENABLED(CONFIG_MY_APP_FEATURE_X)
    /* Feature X code */
#endif

#define POLL_MS CONFIG_MY_APP_SENSOR_POLL_INTERVAL_MS
` ` `

## Debugging Kconfig

` ` `bash
# Interactive symbol browser
west build -t menuconfig          # Terminal
west build -t guiconfig           # GUI

# Check resolved value of a symbol
grep CONFIG_SPI build/zephyr/.config

# Find where a symbol is defined
grep -rn "config SPI$" $ZEPHYR_BASE/drivers/spi/Kconfig

# See full dependency chain for a symbol
# (use menuconfig → search with / → shows depends on, selected by)

# Check all symbols that differ from defaults
diff <(grep "^CONFIG" build/zephyr/.config) <(grep "^CONFIG" build/zephyr/.config.old) 2>/dev/null

# See what selected/forced a symbol
grep -rn "select SPI" $ZEPHYR_BASE/ --include="Kconfig*"
grep -rn "depends on SPI" $ZEPHYR_BASE/ --include="Kconfig*"
` ` `

## Common Kconfig Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `warning: SPI (=n) was assigned the value y` | Unmet dependency | Check `depends on` in Kconfig definition |
| `'CONFIG_FOO' undeclared` in C | Symbol not in `.config` | Verify symbol is enabled, use `IS_ENABLED()` guard |
| Symbol stuck at `n` despite `=y` in prj.conf | Missing dependency | Use menuconfig search to find deps |
| `undefined reference to z_impl_*` | Subsystem not enabled | Enable the parent subsystem Kconfig |
| Board .conf not being applied | Wrong filename | Must match exact board name (e.g., `nucleo_f411re.conf`) |
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/zephyr-kconfig/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/zephyr-kconfig/SKILL.md
git commit -m "feat: add zephyr-kconfig skill"
```

---

## Chunk 3: Zephyr Devicetree Skill

Dedicated devicetree reference — overlays, macros, bindings, debugging DT issues.

### Task 3: Create SKILL.md for zephyr-devicetree

**Files:**
- Create: `skills/zephyr-devicetree/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/zephyr-devicetree
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/zephyr-devicetree/SKILL.md` with this content:

```markdown
---
name: zephyr-devicetree
description: Use when working with Zephyr devicetree - writing overlay files, DT macros like DT_ALIAS DT_NODELABEL DT_INST, GPIO_DT_SPEC_GET, devicetree binding errors, accessing DT properties in C, or debugging missing or incorrect DT nodes
---

# Zephyr Devicetree

## Overview

Reference for Zephyr's devicetree system. Covers overlay syntax, accessing nodes in C, DT macro patterns, and debugging devicetree issues.

## When to Use

- Writing or modifying `.overlay` files
- Build errors mentioning `DT_`, nodes, or bindings
- Need to access devicetree properties from C code
- Debugging why a node isn't being picked up

**When NOT to use:**
- Writing DT bindings for a new driver (use zephyr-driver-development)
- Kconfig issues (use zephyr-kconfig)

## Where Devicetree Lives

| File | Purpose |
|------|---------|
| `boards/<board>.overlay` | Board-specific overlay (auto-detected) |
| `app.overlay` | App-wide overlay (auto-detected in app root) |
| CMake `-DDTC_OVERLAY_FILE=` | Explicit overlay path |
| `$ZEPHYR_BASE/boards/<vendor>/<board>/<board>.dts` | Base board DTS |
| `$ZEPHYR_BASE/dts/bindings/` | Binding YAML files |

**Merge order:** SoC DTSI → Board DTS → Board overlay → App overlay → CMake overlays

## Overlay Patterns

### Enable and configure a bus peripheral

` ` `dts
/* boards/nucleo_f411re.overlay */
&spi1 {
    status = "okay";
    pinctrl-0 = <&spi1_sck_pa5 &spi1_miso_pa6 &spi1_mosi_pa7>;
    pinctrl-names = "default";
    cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>;

    my_sensor: my_sensor@0 {
        compatible = "bosch,bme280";
        reg = <0>;
        spi-max-frequency = <1000000>;
    };
};
` ` `

### Add GPIO-controlled devices

` ` `dts
/ {
    leds {
        compatible = "gpio-leds";
        status_led: status_led {
            gpios = <&gpiob 3 GPIO_ACTIVE_HIGH>;
            label = "Status LED";
        };
    };

    aliases {
        status-led = &status_led;
    };
};
` ` `

### Add a chosen node

` ` `dts
/ {
    chosen {
        my-app-storage = &flash0;
    };
};
` ` `

### Delete/disable a node from base DTS

` ` `dts
/* Disable a peripheral you don't use */
&usart3 {
    status = "disabled";
};

/* Delete a node entirely (rare, prefer disabling) */
/delete-node/ &some_unwanted_node;
` ` `

## Accessing DT Nodes in C

` ` `c
/* By alias (preferred for app-level access) */
#define STATUS_LED_NODE DT_ALIAS(status_led)

/* By nodelabel (preferred for specific peripherals) */
#define SPI_DEV_NODE DT_NODELABEL(my_sensor)

/* By chosen (for system-level resources) */
#define CONSOLE_NODE DT_CHOSEN(zephyr_console)

/* By path (avoid — brittle, breaks on SoC changes) */
#define NODE DT_PATH(soc, spi_40013000)
` ` `

## DT Property Macros

` ` `c
/* Get a property value */
#define SPI_FREQ DT_PROP(SPI_DEV_NODE, spi_max_frequency)

/* Check node exists (compile time) */
#if DT_NODE_EXISTS(DT_ALIAS(status_led))
    /* ... */
#endif

/* Check node has a property */
#if DT_NODE_HAS_PROP(MY_NODE, high_resolution)
    /* ... */
#endif

/* Get GPIO spec from DT (most common for GPIO access) */
static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(STATUS_LED_NODE, gpios);

/* Get SPI spec from DT */
static const struct spi_dt_spec spi_dev =
    SPI_DT_SPEC_GET(DT_NODELABEL(my_sensor),
                     SPI_OP_MODE_MASTER | SPI_WORD_SET(8), 0);

/* Get I2C spec from DT */
static const struct i2c_dt_spec i2c_dev =
    I2C_DT_SPEC_GET(DT_NODELABEL(my_i2c_device));

/* Get a string property */
const char *label = DT_PROP(MY_NODE, label);

/* Get an array property element */
uint32_t val = DT_PROP_BY_IDX(MY_NODE, my_array_prop, 0);

/* Get phandle reference */
#define PARENT_NODE DT_PHANDLE(MY_NODE, parent_bus)
` ` `

## Debugging Devicetree

` ` `bash
# Final merged DT (most useful — shows what the build actually sees)
cat build/zephyr/zephyr.dts

# Check a specific node exists
grep "status_led" build/zephyr/zephyr.dts

# Generated C macros (all DT_* macros resolve here)
# Useful when a macro gives unexpected value
cat build/zephyr/include/generated/zephyr/devicetree_generated.h

# Search for a specific generated macro
grep "MY_SENSOR" build/zephyr/include/generated/zephyr/devicetree_generated.h

# Validate overlay syntax without full build
dtc -I dts -O dtb -o /dev/null boards/my_board.overlay
` ` `

## Common DT Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `node not found: DT_ALIAS(foo)` | Alias not in overlay or base DTS | Add `/{ aliases { foo = &node; }; };` to overlay |
| `node not found: DT_NODELABEL(foo)` | Label doesn't exist | Check spelling, verify in `build/zephyr/zephyr.dts` |
| `Duplicate node name` | Overlay creates node that already exists | Use `&existing_label { ... };` to modify, don't recreate |
| `Could not find binding for ...` | Missing or wrong `compatible` | Check binding exists in `dts/bindings/`, spelling matches |
| `property not found` | Property name mismatch | Check binding YAML for exact property name (hyphens not underscores) |
| DT macro returns wrong value | Stale build cache | `west build -p always` for pristine rebuild |
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/zephyr-devicetree/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/zephyr-devicetree/SKILL.md
git commit -m "feat: add zephyr-devicetree skill"
```

---

## Chunk 4: Embedded Debugging Skill

Debug probe setup, cortex-debug VSCode config, GDB commands, memory corruption workflow.

### Task 4: Create SKILL.md for embedded-debugging

**Files:**
- Create: `skills/embedded-debugging/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/embedded-debugging
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/embedded-debugging/SKILL.md` with this content:

```markdown
---
name: embedded-debugging
description: Use when debugging embedded firmware on Zephyr - setting up cortex-debug in VSCode, west debug not connecting, J-Link or OpenOCD configuration, GDB commands for embedded, memory corruption diagnosis, or inspecting peripheral registers on ARM Cortex-M targets
---

# Embedded Debugging

## Overview

Debugging workflow for Zephyr applications on ARM Cortex-M targets. Covers GDB via west debug, VSCode cortex-debug configuration, and memory corruption diagnosis.

**REQUIRED BACKGROUND:** You MUST understand superpowers:systematic-debugging. That skill defines the 4-phase root cause investigation. This skill adapts it to embedded-specific tools and failure modes.

## When to Use

- Need to set up cortex-debug in VSCode for a Zephyr project
- `west debug` won't connect or behaves unexpectedly
- Memory corruption symptoms (corrupted variables, stack overflow)
- Need to inspect peripheral registers live
- Need GDB commands specific to embedded targets

**When NOT to use:**
- HardFault/BusFault/MemManage crashes (use cortex-m-fault-diagnosis)
- Build errors (use zephyr-app-development)
- Test failures on native_sim (use standard debugging)

## Debug Probe Setup

### west debug (command line GDB)

` ` `bash
# J-Link (most Nordic, some NXP boards)
west debug --runner jlink

# OpenOCD (most STM32, RP2040, many others)
west debug --runner openocd

# If runner auto-detection fails, check board.cmake:
cat $ZEPHYR_BASE/boards/<vendor>/<board>/board.cmake
` ` `

### VSCode cortex-debug

Create `.vscode/launch.json`:

` ` `json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug (J-Link)",
            "type": "cortex-debug",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/zephyr/zephyr.elf",
            "servertype": "jlink",
            "device": "STM32F411RE",
            "interface": "swd",
            "runToEntryPoint": "main",
            "svdFile": "${workspaceFolder}/svd/STM32F411.svd",
            "preLaunchTask": "west build",
            "showDevDebugOutput": "raw"
        },
        {
            "name": "Cortex Debug (OpenOCD)",
            "type": "cortex-debug",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/zephyr/zephyr.elf",
            "servertype": "openocd",
            "configFiles": [
                "interface/stlink.cfg",
                "target/stm32f4x.cfg"
            ],
            "runToEntryPoint": "main",
            "svdFile": "${workspaceFolder}/svd/STM32F411.svd",
            "preLaunchTask": "west build",
            "showDevDebugOutput": "raw"
        }
    ]
}
` ` `

**Finding SVD files:** SVD files enable peripheral register viewing in cortex-debug.
- CMSIS-Pack: https://developer.arm.com/tools-and-software/embedded/cmsis/cmsis-packs
- Community: https://github.com/posborne/cmsis-svd
- Vendor SDKs (STM32CubeMX, nRF Connect SDK, MCUXpresso)

**Build task** (`.vscode/tasks.json`):
` ` `json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "west build",
            "type": "shell",
            "command": "west build -p auto",
            "group": { "kind": "build", "isDefault": true },
            "problemMatcher": ["$gcc"]
        }
    ]
}
` ` `

## Stack Overflow Detection

` ` `kconfig
# prj.conf - enable stack overflow detection
CONFIG_THREAD_STACK_INFO=y
CONFIG_THREAD_ANALYZER=y
CONFIG_THREAD_ANALYZER_USE_PRINTK=y
CONFIG_THREAD_ANALYZER_AUTO=y
CONFIG_THREAD_ANALYZER_AUTO_INTERVAL=5

# Hardware stack protection (Cortex-M with MPU)
CONFIG_HW_STACK_PROTECTION=y

# Stack sentinel (software, no MPU needed)
CONFIG_STACK_SENTINEL=y
` ` `

Thread analyzer output:
` ` `
Thread: my_thread  Stack size: 1024  Used: 856  Unused: 168  Usage: 83%
` ` `
Rule of thumb: if usage > 70%, increase stack size.

## Memory Corruption Debugging

1. **Enable MPU stack protection** — catches stack overflows at the moment they happen
2. **Use thread analyzer** — find threads close to stack limit
3. **Enable ASAN on native_sim** — catches buffer overflows, use-after-free:
   ` ` `kconfig
   CONFIG_ASAN=y
   ` ` `
4. **Hardware watchpoint on corrupted address:**
   ` ` `gdb
   watch *(uint32_t *)0x20001234
   ` ` `
5. **Memory fill patterns** — Zephyr fills unused stack with `0xAA`, check for corruption

## GDB Commands for Embedded

` ` `gdb
# Backtrace
bt

# Thread inspection
info threads
thread <id>

# Inspect peripheral register (e.g., GPIOA ODR on STM32)
x/x 0x40020014

# Read memory region
x/16xw 0x20000000

# Zephyr kernel objects
p k_current_get()
p *(struct k_thread *)0x20001234

# Monitor a variable
display my_variable

# Hardware breakpoint (for flash-resident code)
hbreak function_name

# Reset target
monitor reset halt

# Continue from reset
monitor reset run
` ` `

## Kconfig for Maximum Debug Info

` ` `kconfig
# prj.conf - debug build
CONFIG_DEBUG=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_OPTIMIZATIONS=y    # -Og instead of -Os
CONFIG_FAULT_DUMP=2
CONFIG_EXTRA_EXCEPTION_INFO=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4      # DBG level
CONFIG_THREAD_NAME=y            # Named threads in debugger
CONFIG_THREAD_MONITOR=y
` ` `
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/embedded-debugging/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/embedded-debugging/SKILL.md
git commit -m "feat: add embedded-debugging skill"
```

---

## Chunk 5: Cortex-M Fault Diagnosis Skill

Fault register decode reference, common fault scenarios, stack overflow detection.

### Task 5: Create SKILL.md for cortex-m-fault-diagnosis

**Files:**
- Create: `skills/cortex-m-fault-diagnosis/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/cortex-m-fault-diagnosis
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/cortex-m-fault-diagnosis/SKILL.md` with this content:

```markdown
---
name: cortex-m-fault-diagnosis
description: Use when an ARM Cortex-M target crashes with HardFault, MemManage fault, BusFault, UsageFault, or mysterious resets - covers CFSR HFSR register decode, Zephyr fault handler output, stack overflow crashes, and common fault scenarios
---

# Cortex-M Fault Diagnosis

## Overview

Reference for diagnosing ARM Cortex-M fault exceptions on Zephyr. Decode fault registers, interpret Zephyr's fault handler output, and map symptoms to root causes.

## When to Use

- Application crashes with HardFault, MemManage, BusFault, or UsageFault
- Zephyr prints a fault dump you need to interpret
- Mysterious resets or watchdog triggers
- HardFault on function return (stack overflow)

**When NOT to use:**
- Setting up a debug session (use embedded-debugging)
- No crash — just need to step through code (use embedded-debugging)

## Reading Fault Registers

In GDB or cortex-debug console:

` ` `gdb
p/x *(uint32_t *)0xE000ED28     # CFSR (Configurable Fault Status Register)
p/x *(uint32_t *)0xE000ED2C     # HFSR (HardFault Status Register)
p/x *(uint32_t *)0xE000ED34     # MMAR (MemManage Fault Address)
p/x *(uint32_t *)0xE000ED38     # BFAR (BusFault Address)
` ` `

## CFSR Bit Decode

### MemManage Fault (bits [7:0])

| Bit | Name | Meaning |
|-----|------|---------|
| [0] | IACCVIOL | Instruction access violation |
| [1] | DACCVIOL | Data access violation |
| [3] | MUNSTKERR | Unstacking error on return |
| [4] | MSTKERR | Stacking error on exception entry |
| [7] | MMARVALID | MMAR holds valid fault address |

### BusFault (bits [15:8])

| Bit | Name | Meaning |
|-----|------|---------|
| [8] | IBUSERR | Instruction bus error |
| [9] | PRECISERR | Precise data bus error (BFAR valid) |
| [10] | IMPRECISERR | Imprecise data bus error |
| [11] | UNSTKERR | Unstacking error |
| [12] | STKERR | Stacking error |
| [15] | BFARVALID | BFAR holds valid fault address |

### UsageFault (bits [25:16])

| Bit | Name | Meaning |
|-----|------|---------|
| [16] | UNDEFINSTR | Undefined instruction |
| [17] | INVSTATE | Invalid state (Thumb bit) |
| [18] | INVPC | Invalid PC load |
| [19] | NOCP | No coprocessor |
| [24] | UNALIGNED | Unaligned access |
| [25] | DIVBYZERO | Divide by zero |

### HFSR (HardFault Status Register, 0xE000ED2C)

| Bit | Name | Meaning |
|-----|------|---------|
| [1] | VECTTBL | Bus fault on vector table read |
| [30] | FORCED | Escalated from configurable fault |
| [31] | DEBUGEVT | Debug event |

**FORCED=1** means a MemManage, BusFault, or UsageFault was escalated to HardFault. Read CFSR for the actual cause.

## Zephyr Fault Handler Output

Zephyr prints fault info automatically with `CONFIG_FAULT_DUMP=2`:

` ` `
***** MPU FAULT *****
  Data Access Violation
  MMFAR Address: 0x00000000
***** HARD FAULT *****
Current thread: 0x20001234 (my_thread)
` ` `

**Enable maximum fault info:**
` ` `kconfig
CONFIG_FAULT_DUMP=2
CONFIG_EXTRA_EXCEPTION_INFO=y
CONFIG_RESET_ON_FATAL_ERROR=n    # Halt instead of reset (for debugging)
` ` `

## Common Fault Scenarios

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| HardFault at `0x00000000` | NULL pointer dereference | Check MMAR, backtrace to find the dereference |
| HardFault on function return | Stack overflow | Check thread stack usage: `CONFIG_THREAD_ANALYZER=y` |
| BusFault PRECISERR | Access to unmapped/disabled peripheral | Check peripheral clock enable, DT `status = "okay"` |
| UsageFault UNALIGNED | Packed struct or wrong pointer cast | Check struct packing, pointer alignment |
| UsageFault UNDEFINSTR | Corrupted function pointer or flash | Check function pointers, verify flash integrity |
| Silent hang (no fault) | Deadlock or infinite loop | Enable watchdog, check semaphore/mutex usage |
| Reset loop | Fault in fault handler, or watchdog | Increase stack sizes, set `CONFIG_RESET_ON_FATAL_ERROR=n` |
| FORCED=1 in HFSR | Escalated configurable fault | Read CFSR — the real cause is there |

## Diagnosis Flowchart

1. **Read HFSR** — is FORCED set?
   - Yes → read CFSR for real cause
   - No → VECTTBL (vector table corruption) or DEBUGEVT
2. **Read CFSR** — which fault type?
   - MemManage → check MMAR if MMARVALID
   - BusFault → check BFAR if BFARVALID
   - UsageFault → check specific bit for cause
3. **Get backtrace** — `bt` in GDB
4. **Check the faulting address** against memory map:
   - `0x00000000` → NULL dereference
   - `0x20xxxxxx` → RAM (stack overflow? buffer overrun?)
   - `0x40xxxxxx` → Peripheral (not enabled? wrong address?)
   - `0x08xxxxxx` → Flash (corruption? bad function pointer?)
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/cortex-m-fault-diagnosis/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/cortex-m-fault-diagnosis/SKILL.md
git commit -m "feat: add cortex-m-fault-diagnosis skill"
```

---

## Chunk 6: West Workspace Management Skill

West manifests, module management, multi-repo projects, Zephyr version management.

### Task 6: Create SKILL.md for west-workspace-management

**Files:**
- Create: `skills/west-workspace-management/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/west-workspace-management
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/west-workspace-management/SKILL.md` with this content:

```markdown
---
name: west-workspace-management
description: Use when managing west workspaces, west manifests, Zephyr version pinning, adding west modules, multi-repo Zephyr project setup, or troubleshooting west update and west init issues
---

# West Workspace Management

## Overview

Managing Zephyr workspaces with west — manifest files, module dependencies, Zephyr version pinning, and multi-repo project organization.

## When to Use

- Setting up a new Zephyr project workspace
- Adding or updating west modules
- Pinning or updating the Zephyr version
- Multi-repo project setup with custom modules
- `west update` failures or version conflicts

**When NOT to use:**
- Build/flash commands (use zephyr-app-development)
- Debugging sessions (use embedded-debugging)

## Workspace Layout (Application-Centric)

` ` `
my-workspace/
├── my-app/                 # ← manifest repo (your code)
│   ├── west.yml
│   ├── CMakeLists.txt
│   ├── prj.conf
│   └── src/
├── zephyr/                 # Managed by west
├── modules/                # Managed by west
└── .west/
    └── config
` ` `

### Init from scratch

` ` `bash
mkdir my-workspace && cd my-workspace
git init my-app
# (create west.yml in my-app)
west init -l my-app
west update
` ` `

## West Manifest (west.yml)

### Minimal manifest

` ` `yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos

  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true              # Pull in Zephyr's own dependencies

  self:
    path: my-app
` ` `

### Adding a custom module

` ` `yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
    - name: my-org
      url-base: https://github.com/my-org

  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true

    - name: my-driver-library
      remote: my-org
      revision: main
      path: modules/lib/my-driver-library

  self:
    path: my-app
` ` `

### Overriding an imported project's revision

` ` `yaml
  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.1.0
      import: true

    # Overrides the revision imported by Zephyr
    - name: hal_stm32
      remote: zephyrproject-rtos
      revision: my-custom-branch
      path: modules/hal/stm32
` ` `

Manifest repo declarations take priority over imported ones.

### Filtering imports (reduce clone time)

` ` `yaml
    - name: zephyr
      import:
        name-allowlist:
          - cmsis
          - hal_stm32
          - hal_nordic
` ` `

## Common West Operations

` ` `bash
west update                          # Update all to manifest
west update zephyr                   # Update single project
west list                            # Show manifest state
west list zephyr --format "{abspath}" # Show project path
west diff                            # Manifest vs actual
west manifest --resolve              # Show import resolution
west manifest --validate             # Check for errors
` ` `

## Updating Zephyr Version

1. Check current: `cat zephyr/VERSION && west list zephyr`
2. Read release notes / migration guide for target version
3. Update `revision` in `west.yml`
4. `west update`
5. Check for breaking changes: `west build -b <board> -p auto 2>&1 | grep -i "deprecated\|binding"`
6. Build and test on all target boards

### Version pinning strategy

| Strategy | When |
|----------|------|
| Release tag (`v4.1.0`) | Production — stable, tested |
| Branch (`main`) | Development only — tracking latest |
| Commit SHA | Maximum reproducibility — CI/CD |

## Making a Library a West Module

` ` `
my-library/
├── zephyr/
│   └── module.yml          # ← Required
├── CMakeLists.txt
├── Kconfig
├── include/
└── src/
` ` `

**`zephyr/module.yml`:**
` ` `yaml
build:
  cmake: .
  kconfig: Kconfig
` ` `

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `west update` fails | Revision not found | `git ls-remote <url> <revision>` |
| Module not found by build | Missing `zephyr/module.yml` | Create it with cmake/kconfig paths |
| Import cycle | Manifests import each other | Use explicit deps, not circular imports |
| `west: unknown command 'build'` | Zephyr not in workspace | `west update`, check `west list` |
| Stale build after update | CMake cache stale | `west build -p always` |
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/west-workspace-management/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/west-workspace-management/SKILL.md
git commit -m "feat: add west-workspace-management skill"
```

---

## Chunk 7: Embedded TDD Skill

TDD for embedded — ztest/native_sim unit tests, pytest HIL testing. No twister (that's in driver-development).

### Task 7: Create SKILL.md for embedded-tdd

**Files:**
- Create: `skills/embedded-tdd/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/embedded-tdd
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/embedded-tdd/SKILL.md` with this content:

```markdown
---
name: embedded-tdd
description: Use when writing tests for Zephyr firmware - unit tests with ztest on native_sim or QEMU, hardware-in-the-loop HIL testing with pytest over serial, mocking hardware dependencies, or setting up test infrastructure for an embedded project
---

# Embedded TDD

## Overview

Test-Driven Development adapted for embedded systems on Zephyr. Two test tiers: unit tests (native_sim/QEMU with ztest) and hardware-in-the-loop (pytest over serial/debug probe).

**REQUIRED BACKGROUND:** You MUST understand superpowers:test-driven-development. Same RED-GREEN-REFACTOR discipline applies. This skill adds embedded-specific test infrastructure.

## When to Use

- Writing any new Zephyr application feature (TDD from the start)
- Adding unit tests for existing Zephyr modules
- Building pytest HIL test harness for on-target testing
- Mocking hardware dependencies for testability

**When NOT to use:**
- Multi-board test matrix with twister (use zephyr-driver-development)
- Debugging a crash (use embedded-debugging / cortex-m-fault-diagnosis)

## The Iron Law

` ` `
NO FIRMWARE CODE WITHOUT A FAILING TEST FIRST
` ` `

Same exceptions as superpowers:test-driven-development. No additional exceptions for embedded.

"But I can't test hardware interactions without hardware" — yes you can. native_sim, QEMU, mocks, and fakes exist. Only the final HIL validation requires real hardware.

## Test Tiers

| Tier | Tool | Target | Speed | What it catches |
|------|------|--------|-------|-----------------|
| Unit | ztest + native_sim | Host or QEMU | Seconds | Logic bugs, API misuse, edge cases |
| HIL | pytest + serial | Real hardware | Minutes+ | Peripheral behavior, timing, electrical |

## Tier 1: Unit Tests with ztest

### Test project structure

` ` `
tests/unit/my_module/
├── CMakeLists.txt
├── prj.conf
├── testcase.yaml
└── src/
    └── main.c
` ` `

### CMakeLists.txt

` ` `cmake
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(test_my_module)

target_sources(app PRIVATE
    src/main.c
    ${CMAKE_CURRENT_SOURCE_DIR}/../../../src/my_module.c
)
target_include_directories(app PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/../../../include
)
` ` `

### prj.conf

` ` `kconfig
CONFIG_ZTEST=y
CONFIG_ZTEST_NEW_API=y
` ` `

### Test code

` ` `c
#include <zephyr/ztest.h>
#include "app/my_module.h"

ZTEST_SUITE(my_module, NULL, NULL, NULL, NULL, NULL);

ZTEST(my_module, test_init_returns_zero)
{
    int ret = my_module_init();
    zassert_equal(ret, 0, "Expected 0, got %d", ret);
}

ZTEST(my_module, test_process_valid_input)
{
    struct my_data data = {.value = 42};
    int ret = my_module_process(&data);
    zassert_equal(ret, 0);
    zassert_equal(data.result, 84, "Expected 84, got %d", data.result);
}

ZTEST(my_module, test_process_null_input)
{
    int ret = my_module_process(NULL);
    zassert_equal(ret, -EINVAL, "Expected -EINVAL for NULL input");
}
` ` `

### testcase.yaml

` ` `yaml
tests:
  my_app.unit.my_module:
    platform_allow:
      - native_sim
      - qemu_cortex_m3
    tags: unit
    integration_platforms:
      - native_sim
` ` `

### Running unit tests

` ` `bash
# Build and run on native_sim
west build -b native_sim tests/unit/my_module -p auto && west build -t run

# Run specific test suite
west build -b native_sim tests/unit/my_module -p auto -- -DCONFIG_ZTEST_SHUFFLE=y
` ` `

### Mocking hardware dependencies

For code that calls Zephyr driver APIs, create thin wrappers:

` ` `c
/* app_gpio.h — testable wrapper */
int app_gpio_set(const struct gpio_dt_spec *spec, int value);
int app_gpio_get(const struct gpio_dt_spec *spec);
` ` `

` ` `c
/* app_gpio.c — real implementation */
#include <zephyr/drivers/gpio.h>
#include "app/app_gpio.h"

int app_gpio_set(const struct gpio_dt_spec *spec, int value)
{
    return gpio_pin_set_dt(spec, value);
}
` ` `

` ` `c
/* test_doubles/fake_gpio.c — test double */
static int fake_pin_state[32];

int app_gpio_set(const struct gpio_dt_spec *spec, int value)
{
    fake_pin_state[spec->pin] = value;
    return 0;
}

/* Test helper */
int fake_gpio_get_state(int pin) { return fake_pin_state[pin]; }
` ` `

Link the fake in your test CMakeLists.txt instead of the real implementation.

## Tier 2: HIL Testing with pytest

### Architecture

` ` `
pytest (host)  ──serial/USB──►  DUT (device under test)
    │                              │
    ├── sends commands ──────────► parses & executes
    ├── reads responses ◄──────── sends results
    └── asserts on behavior        runs firmware test shell
` ` `

### Firmware side: Zephyr shell

` ` `kconfig
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
` ` `

` ` `c
#include <zephyr/shell/shell.h>

static int cmd_read_sensor(const struct shell *sh, size_t argc, char **argv)
{
    struct sensor_value val;
    int ret = sensor_sample_fetch(sensor_dev);
    if (ret) {
        shell_error(sh, "fetch failed: %d", ret);
        return ret;
    }
    sensor_channel_get(sensor_dev, SENSOR_CHAN_AMBIENT_TEMP, &val);
    shell_print(sh, "TEMP:%d.%06d", val.val1, val.val2);
    return 0;
}

SHELL_CMD_REGISTER(read_sensor, NULL, "Read sensor value", cmd_read_sensor);
` ` `

### pytest side

` ` `python
# tests/hil/conftest.py
import pytest
import serial
import time

def pytest_addoption(parser):
    parser.addoption("--serial-port", default="/dev/ttyACM0")
    parser.addoption("--baud", default=115200, type=int)

@pytest.fixture(scope="session")
def dut_serial(request):
    """Connect to Device Under Test over serial."""
    port = request.config.getoption("--serial-port")
    baud = request.config.getoption("--baud")
    ser = serial.Serial(port, baud, timeout=5)
    time.sleep(2)  # Wait for boot
    ser.reset_input_buffer()
    yield ser
    ser.close()

def send_command(ser, cmd, timeout=5):
    """Send shell command and return response lines."""
    ser.write(f"{cmd}\n".encode())
    time.sleep(0.1)
    lines = []
    deadline = time.time() + timeout
    while time.time() < deadline:
        line = ser.readline().decode(errors="replace").strip()
        if line:
            lines.append(line)
        if "uart:~$" in line:
            break
    return lines
` ` `

` ` `python
# tests/hil/test_sensor.py

def test_sensor_read_returns_valid_temp(dut_serial):
    lines = send_command(dut_serial, "read_sensor")
    temp_lines = [l for l in lines if l.startswith("TEMP:")]
    assert len(temp_lines) == 1, f"Expected one TEMP line, got: {lines}"

    temp_str = temp_lines[0].split(":")[1]
    temp = float(temp_str)
    assert -40.0 < temp < 85.0, f"Temperature {temp} out of plausible range"

def test_sensor_read_is_stable(dut_serial):
    def read_temp():
        lines = send_command(dut_serial, "read_sensor")
        temp_line = [l for l in lines if l.startswith("TEMP:")][0]
        return float(temp_line.split(":")[1])

    t1 = read_temp()
    t2 = read_temp()
    assert abs(t1 - t2) < 1.0, f"Readings differ too much: {t1} vs {t2}"
` ` `

### Running HIL tests

` ` `bash
pytest tests/hil/ -v --serial-port /dev/ttyACM0
pytest tests/hil/ -v --junitxml=reports/hil.xml   # CI report
` ` `

## TDD Workflow for Embedded

### RED
1. Write a ztest unit test that exercises the behavior
2. Build for native_sim: `west build -b native_sim tests/unit/my_module -p auto`
3. Run: `west build -t run`
4. **Verify it fails**

### GREEN
5. Write minimal implementation
6. Build and run — **verify it passes**

### REFACTOR
7. Clean up, extract functions
8. Run tests — **verify still passing**
9. Commit

### Then promote
10. Add pytest HIL test if feature involves real peripheral behavior

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Can't test hardware without hardware" | native_sim + mocks cover 80% of logic |
| "Serial parsing is too fragile to test" | Fragile code needs tests most |
| "Unit tests are pointless for embedded" | They catch logic bugs in seconds vs. minutes of flash+debug |
| "I'll test on hardware later" | Later = never. Write native_sim test now |
| "The driver is from Zephyr, I don't test it" | You're testing YOUR usage, not the driver |
| "Test infrastructure takes too long" | Setup once, benefit for entire project |
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/embedded-tdd/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/embedded-tdd/SKILL.md
git commit -m "feat: add embedded-tdd skill"
```

---

## Chunk 8: Zephyr Driver Development Skill

Writing drivers, DT bindings, instance macros, twister for multi-board testing, upstream workflow.

### Task 8: Create SKILL.md for zephyr-driver-development

**Files:**
- Create: `skills/zephyr-driver-development/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/zephyr-driver-development
```

- [ ] **Step 2: Write the SKILL.md**

Create `skills/zephyr-driver-development/SKILL.md` with this content:

```markdown
---
name: zephyr-driver-development
description: Use when writing Zephyr device drivers - implementing the device model, writing devicetree bindings, creating driver instances with DT_INST macros, testing drivers with twister across boards, upstreaming drivers to the Zephyr project, or managing a forked Zephyr with custom drivers
---

# Zephyr Driver Development

## Overview

Writing Zephyr device drivers following the official device model. Covers DT bindings, instance-based macros, driver testing with twister, and the upstream PR workflow.

**REQUIRED BACKGROUND:**
- zephyr-devicetree (DT overlay and binding patterns)
- zephyr-kconfig (driver Kconfig patterns)
- embedded-tdd (driver testing approach)
- west-workspace-management (manifest for fork/upstream)

## When to Use

- Writing a new device driver for a sensor, actuator, or IC
- Porting an existing driver from another RTOS to Zephyr
- Creating custom devicetree bindings for a new device
- Testing a driver across multiple boards with twister
- Preparing a driver for upstream submission
- Managing out-of-tree drivers in a west module

**When NOT to use:**
- Using existing Zephyr drivers (use zephyr-app-development)
- Just writing an overlay (use zephyr-devicetree)

## Driver File Structure

### Out-of-tree (in your module)

` ` `
my-drivers/
├── zephyr/
│   └── module.yml
├── drivers/sensor/my_sensor/
│   ├── CMakeLists.txt
│   ├── Kconfig
│   └── my_sensor.c
├── dts/bindings/sensor/
│   └── myvendor,my-sensor.yaml
└── tests/drivers/my_sensor/
    ├── CMakeLists.txt
    ├── prj.conf
    ├── testcase.yaml
    └── src/main.c
` ` `

### In-tree (for upstream)

` ` `
zephyr/
├── drivers/sensor/my_sensor/
│   ├── CMakeLists.txt
│   ├── Kconfig
│   └── my_sensor.c
├── dts/bindings/sensor/
│   └── myvendor,my-sensor.yaml
└── tests/drivers/sensor/my_sensor/
` ` `

## Devicetree Binding

` ` `yaml
# dts/bindings/sensor/myvendor,my-sensor.yaml
description: MyVendor MY-SENSOR temperature and humidity sensor

compatible: "myvendor,my-sensor"

include: [sensor-device.yaml, i2c-device.yaml]

properties:
  odr:
    type: int
    default: 1
    description: Output data rate in Hz
    enum: [1, 10, 25, 50, 100]

  high-resolution:
    type: boolean
    description: Enable high-resolution mode
` ` `

**Key rules:**
- `compatible` must match `DT_DRV_COMPAT` (commas→underscores, hyphens→underscores)
- `include` base bindings for the subsystem
- Vendor prefix must be in `dts/bindings/vendor-prefixes.txt` for upstream

## Driver Implementation (I2C Sensor)

` ` `c
#define DT_DRV_COMPAT myvendor_my_sensor

#include <zephyr/device.h>
#include <zephyr/drivers/i2c.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(my_sensor, CONFIG_SENSOR_LOG_LEVEL);

struct my_sensor_config {
    struct i2c_dt_spec i2c;
    uint8_t odr;
    bool high_resolution;
};

struct my_sensor_data {
    int32_t temperature;    /* 0.001 °C */
    int32_t humidity;       /* 0.001 % */
};

static int my_sensor_sample_fetch(const struct device *dev,
                                   enum sensor_channel chan)
{
    const struct my_sensor_config *cfg = dev->config;
    struct my_sensor_data *data = dev->data;

    if (chan != SENSOR_CHAN_ALL &&
        chan != SENSOR_CHAN_AMBIENT_TEMP &&
        chan != SENSOR_CHAN_HUMIDITY) {
        return -ENOTSUP;
    }

    uint8_t raw[4];
    int ret = i2c_burst_read_dt(&cfg->i2c, 0x00, raw, sizeof(raw));
    if (ret) {
        LOG_ERR("Read failed: %d", ret);
        return ret;
    }

    data->temperature = ((int16_t)(raw[0] << 8 | raw[1]) * 1000) / 100;
    data->humidity = ((int16_t)(raw[2] << 8 | raw[3]) * 1000) / 100;
    return 0;
}

static int my_sensor_channel_get(const struct device *dev,
                                  enum sensor_channel chan,
                                  struct sensor_value *val)
{
    struct my_sensor_data *data = dev->data;

    switch (chan) {
    case SENSOR_CHAN_AMBIENT_TEMP:
        val->val1 = data->temperature / 1000;
        val->val2 = (data->temperature % 1000) * 1000;
        break;
    case SENSOR_CHAN_HUMIDITY:
        val->val1 = data->humidity / 1000;
        val->val2 = (data->humidity % 1000) * 1000;
        break;
    default:
        return -ENOTSUP;
    }
    return 0;
}

static int my_sensor_init(const struct device *dev)
{
    const struct my_sensor_config *cfg = dev->config;

    if (!i2c_is_ready_dt(&cfg->i2c)) {
        LOG_ERR("I2C bus not ready");
        return -ENODEV;
    }

    uint8_t chip_id;
    int ret = i2c_reg_read_byte_dt(&cfg->i2c, 0xFF, &chip_id);
    if (ret || chip_id != 0xAB) {
        LOG_ERR("Chip ID check failed: ret=%d id=0x%02x", ret, chip_id);
        return ret ? ret : -ENODEV;
    }

    ret = i2c_reg_write_byte_dt(&cfg->i2c, 0x01, cfg->odr);
    if (ret) {
        LOG_ERR("ODR config failed: %d", ret);
        return ret;
    }

    LOG_INF("Initialized (ODR=%d, hi-res=%d)", cfg->odr, cfg->high_resolution);
    return 0;
}

static const struct sensor_driver_api my_sensor_api = {
    .sample_fetch = my_sensor_sample_fetch,
    .channel_get = my_sensor_channel_get,
};

#define MY_SENSOR_INST(inst)                                          \
    static struct my_sensor_data my_sensor_data_##inst;               \
    static const struct my_sensor_config my_sensor_config_##inst = {  \
        .i2c = I2C_DT_SPEC_INST_GET(inst),                          \
        .odr = DT_INST_PROP(inst, odr),                              \
        .high_resolution = DT_INST_PROP(inst, high_resolution),      \
    };                                                                \
    SENSOR_DEVICE_DT_INST_DEFINE(inst,                               \
        my_sensor_init, NULL,                                         \
        &my_sensor_data_##inst,                                       \
        &my_sensor_config_##inst,                                     \
        POST_KERNEL,                                                  \
        CONFIG_SENSOR_INIT_PRIORITY,                                  \
        &my_sensor_api);

DT_INST_FOREACH_STATUS_OKAY(MY_SENSOR_INST)
` ` `

### Kconfig

` ` `kconfig
config MY_SENSOR
    bool "MY-SENSOR temperature and humidity sensor"
    default y
    depends on DT_HAS_MYVENDOR_MY_SENSOR_ENABLED
    select I2C
    select SENSOR
    help
      Enable driver for MyVendor MY-SENSOR.
` ` `

### CMakeLists.txt

` ` `cmake
zephyr_library()
zephyr_library_sources(my_sensor.c)
` ` `

### Parent integration (in-tree)

` ` `kconfig
# drivers/sensor/Kconfig — add:
rsource "my_sensor/Kconfig"
` ` `

` ` `cmake
# drivers/sensor/CMakeLists.txt — add:
add_subdirectory_ifdef(CONFIG_MY_SENSOR my_sensor)
` ` `

## Testing with Twister

### testcase.yaml

` ` `yaml
tests:
  drivers.sensor.my_sensor:
    platform_allow:
      - native_sim
      - nucleo_f411re
      - nrf52840dk/nrf52840
    depends_on: i2c
    tags: drivers sensor
    harness: ztest
    extra_configs:
      - CONFIG_MY_SENSOR=y
` ` `

### Running twister

` ` `bash
# All supported platforms (CI)
west twister -T tests/drivers/my_sensor/

# Specific board
west twister -T tests/drivers/my_sensor/ -p nucleo_f411re

# Native sim only (fast, no hardware)
west twister -T tests/drivers/my_sensor/ -p native_sim

# With connected hardware
west twister -T tests/drivers/my_sensor/ -p nucleo_f411re \
    --device-testing --device-serial /dev/ttyACM0

# Retry failures and generate report
west twister -T tests/drivers/my_sensor/ --retry-failed 2 --report-dir reports/
` ` `

## Upstreaming to Zephyr

### Prerequisites

- [ ] Driver follows Zephyr coding style: `$ZEPHYR_BASE/scripts/checkpatch.pl`
- [ ] DT binding in `dts/bindings/<subsystem>/`
- [ ] Vendor prefix in `dts/bindings/vendor-prefixes.txt`
- [ ] Tests in `tests/drivers/<subsystem>/<driver>/`
- [ ] Tests pass on native_sim + at least one real board
- [ ] `MAINTAINERS.yml` entry
- [ ] Commit messages: `drivers: sensor: my_sensor: add driver for ...`

### Workflow

1. **Fork and branch:**
   ` ` `bash
   cd zephyr
   git remote add my-fork https://github.com/<you>/zephyr.git
   git fetch origin
   git checkout -b add-my-sensor-driver origin/main
   ` ` `

2. **Add driver files** (binding, source, Kconfig, CMake, tests)

3. **Run checkpatch:**
   ` ` `bash
   git format-patch origin/main
   $ZEPHYR_BASE/scripts/checkpatch.pl *.patch
   ` ` `

4. **Run tests:**
   ` ` `bash
   west twister -T tests/drivers/sensor/my_sensor/ -p native_sim
   ` ` `

5. **Push and PR:**
   ` ` `bash
   git push my-fork add-my-sensor-driver
   gh pr create --repo zephyrproject-rtos/zephyr \
       --title "drivers: sensor: add MY-SENSOR driver" \
       --body "Add driver for MyVendor MY-SENSOR."
   ` ` `

### West manifest for fork-based development

` ` `yaml
# During development — point to your fork
- name: zephyr
  remote: my-fork
  revision: add-my-sensor-driver
  import: true

# After merge — switch back to upstream
- name: zephyr
  remote: zephyrproject-rtos
  revision: v4.2.0
  import: true
` ` `

### Managing out-of-tree during review

While PR is pending, keep driver in your west module. Once merged:
1. Update `west.yml` to Zephyr revision containing your driver
2. Remove driver from out-of-tree module
3. Update Kconfig/overlay to use in-tree driver
```

- [ ] **Step 3: Verify file was created**

```bash
wc -l skills/zephyr-driver-development/SKILL.md
```

- [ ] **Step 4: Commit**

```bash
git add skills/zephyr-driver-development/SKILL.md
git commit -m "feat: add zephyr-driver-development skill"
```

---

## Chunk 9: Update Plugin Metadata

### Task 9: Update plugin.json

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update plugin.json keywords and description**

Update the `description` and `keywords` fields in `.claude-plugin/plugin.json`, preserving all other existing fields (`name`, `version`, `author`, `homepage`, `repository`, `license`):

- Change `description` to: `"Core skills library for Claude Code, extended with Zephyr RTOS embedded systems development workflows"`
- Add to `keywords` array: `"embedded"`, `"zephyr"`, `"rtos"`, `"firmware"`, `"arm"`, `"cortex-m"`, `"devicetree"`, `"kconfig"`, `"west"`

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: update plugin metadata for embedded focus"
```

---

## Summary

| Chunk | Skill | Type | Key Content |
|-------|-------|------|-------------|
| 1 | `zephyr-app-development` | Workflow | Project structure, build/flash, API patterns, sample adaptation |
| 2 | `zephyr-kconfig` | Reference | prj.conf, custom symbols, debugging resolution, common errors |
| 3 | `zephyr-devicetree` | Reference | Overlays, DT macros, property access, debugging DT |
| 4 | `embedded-debugging` | Workflow | cortex-debug, GDB, stack overflow, memory corruption |
| 5 | `cortex-m-fault-diagnosis` | Reference | CFSR/HFSR decode, fault scenarios, diagnosis flowchart |
| 6 | `west-workspace-management` | Reference/Workflow | Manifests, modules, version pinning, multi-repo |
| 7 | `embedded-tdd` | Discipline | ztest/native_sim, pytest HIL, mocking, rationalizations |
| 8 | `zephyr-driver-development` | Workflow | Device model, DT bindings, twister, upstream PRs |
| 9 | Plugin metadata | — | Updated keywords and description |

**Total new files:** 8 SKILL.md files + 1 modified plugin.json
