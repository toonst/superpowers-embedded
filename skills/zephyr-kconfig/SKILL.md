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

```kconfig
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
```

## Custom App Kconfig

```kconfig
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
```

**Use in C:**
```c
#if IS_ENABLED(CONFIG_MY_APP_FEATURE_X)
    /* Feature X code */
#endif

#define POLL_MS CONFIG_MY_APP_SENSOR_POLL_INTERVAL_MS
```

## Debugging Kconfig

```bash
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
```

## Common Kconfig Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `warning: SPI (=n) was assigned the value y` | Unmet dependency | Check `depends on` in Kconfig definition |
| `'CONFIG_FOO' undeclared` in C | Symbol not in `.config` | Verify symbol is enabled, use `IS_ENABLED()` guard |
| Symbol stuck at `n` despite `=y` in prj.conf | Missing dependency | Use menuconfig search to find deps |
| `undefined reference to z_impl_*` | Subsystem not enabled | Enable the parent subsystem Kconfig |
| Board .conf not being applied | Wrong filename | Must match exact board name (e.g., `nucleo_f411re.conf`) |
