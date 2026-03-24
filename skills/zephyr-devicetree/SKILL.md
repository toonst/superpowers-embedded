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

```dts
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
```

### Add GPIO-controlled devices

```dts
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
```

### Add a chosen node

```dts
/ {
    chosen {
        my-app-storage = &flash0;
    };
};
```

### Delete/disable a node from base DTS

```dts
/* Disable a peripheral you don't use */
&usart3 {
    status = "disabled";
};

/* Delete a node entirely (rare, prefer disabling) */
/delete-node/ &some_unwanted_node;
```

## Accessing DT Nodes in C

```c
/* By alias (preferred for app-level access) */
#define STATUS_LED_NODE DT_ALIAS(status_led)

/* By nodelabel (preferred for specific peripherals) */
#define SPI_DEV_NODE DT_NODELABEL(my_sensor)

/* By chosen (for system-level resources) */
#define CONSOLE_NODE DT_CHOSEN(zephyr_console)

/* By path (avoid — brittle, breaks on SoC changes) */
#define NODE DT_PATH(soc, spi_40013000)
```

## DT Property Macros

```c
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
```

## Debugging Devicetree

```bash
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
```

## Common DT Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `node not found: DT_ALIAS(foo)` | Alias not in overlay or base DTS | Add `/{ aliases { foo = &node; }; };` to overlay |
| `node not found: DT_NODELABEL(foo)` | Label doesn't exist | Check spelling, verify in `build/zephyr/zephyr.dts` |
| `Duplicate node name` | Overlay creates node that already exists | Use `&existing_label { ... };` to modify, don't recreate |
| `Could not find binding for ...` | Missing or wrong `compatible` | Check binding exists in `dts/bindings/`, spelling matches |
| `property not found` | Property name mismatch | Check binding YAML for exact property name (hyphens not underscores) |
| DT macro returns wrong value | Stale build cache | `west build -p always` for pristine rebuild |
