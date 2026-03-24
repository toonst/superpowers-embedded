---
name: zephyr-contribution-style
description: Use when writing or modifying code in the Zephyr source tree for upstream contribution - covers C code style (tabs, C89 comments, braces), MISRA-C subset, naming conventions, commit message format, Doxygen style, Kconfig style, and inclusive language rules
---

# Zephyr Contribution Style

Coding standards for the Zephyr RTOS source tree. Only rules where Zephyr's convention differs from LLM defaults or requires Zephyr-specific knowledge are included.

## When to Use

- Writing or modifying code in the Zephyr source tree (drivers, subsystems, kernel, tests)
- Preparing patches for upstream submission

**When NOT to use:**
- Application code built on top of Zephyr (use zephyr-app-development)

## 1. C Code Style

### Tabs, 8-char width

Indent with tabs, not spaces. Tabs are 8 characters wide (GitHub defaults to 4 — Zephyr follows the Linux kernel convention).

```c
/* WRONG — spaces */
    if (ret) {
        return ret;
    }
```

```c
/* RIGHT — tabs, 8 chars wide */
	if (ret) {
		return ret;
	}
```

### 100-column line limit

Lines must be 100 columns or fewer. Not 80.

### C89 comments only

Use `/* */` for all comments. C99 `//` comments are not allowed. Use `/** */` for Doxygen.

```c
/* WRONG */
// initialize the device
```

```c
/* RIGHT */
/* initialize the device */
```

### Braces always required

Add braces to every `if`, `else`, `do`, `while`, `for`, and `switch` body, even single-line.

```c
/* WRONG */
if (ret)
	return ret;
```

```c
/* RIGHT */
if (ret) {
	return ret;
}
```

### No binary literals

Avoid constants starting with `0b`. Use hex instead.

```c
/* WRONG */
uint8_t mask = 0b00110000;
```

```c
/* RIGHT */
uint8_t mask = 0x30;
```

## 2. MISRA-C Subset

Rules from Zephyr's MISRA-C 2012 subset that the LLM is likely to violate.

### No recursion (Rule 17.2)

Functions shall not call themselves, either directly or indirectly.

### No variadic functions (Rule 17.1)

The features of `<stdarg.h>` shall not be used.

### Dynamic allocation (Dir 4.12, Rule 21.3)

Upstream Zephyr forbids stdlib `malloc`/`free`. Only use them if the project has `CONFIG_COMMON_LIBC_MALLOC=y` with `CONFIG_COMMON_LIBC_MALLOC_ARENA_SIZE != 0`.

Note: `k_malloc`/`k_free` use a separate kernel heap controlled by `CONFIG_HEAP_MEM_POOL_SIZE`.

### Error returns must be tested (Dir 4.7)

Every function that returns error information must have its return value checked.

```c
/* WRONG */
i2c_reg_write_byte_dt(&cfg->i2c, REG_CTRL, val);
```

```c
/* RIGHT */
int ret = i2c_reg_write_byte_dt(&cfg->i2c, REG_CTRL, val);
if (ret) {
	return ret;
}
```

### switch must have default (Rule 16.4)

Every `switch` statement must have a `default` label.

### if-else if chains must end with else (Rule 15.7)

All `if...else if` constructs must be terminated with an `else` statement.

```c
/* WRONG */
if (chan == SENSOR_CHAN_AMBIENT_TEMP) {
	/* ... */
} else if (chan == SENSOR_CHAN_HUMIDITY) {
	/* ... */
}
```

```c
/* RIGHT */
if (chan == SENSOR_CHAN_AMBIENT_TEMP) {
	/* ... */
} else if (chan == SENSOR_CHAN_HUMIDITY) {
	/* ... */
} else {
	return -ENOTSUP;
}
```

### goto only forward (Rule 15.2)

The `goto` statement shall jump to a label declared later in the same function. Used for error cleanup paths, never backward.

### Pointer nesting max 2 levels (Rule 18.5)

Declarations should contain no more than two levels of pointer nesting. No `***p`.

### No cast removing const/volatile (Rule 11.8)

A cast shall not remove any `const` or `volatile` qualification from the type pointed to by a pointer.

### NULL is the only null pointer constant (Rule 11.9)

The macro `NULL` shall be the only permitted form of integer null pointer constant. Not `(void *)0` or bare `0`.

## 3. Naming Conventions

All identifiers use `snake_case`.

Public APIs must use subsystem prefixes:

| Prefix | Subsystem |
|--------|-----------|
| `k_` | Kernel |
| `sys_` | System-wide |
| `net_` | Networking |
| `bt_` | Bluetooth |
| `i2c_` | I2C controller |
| `spi_` | SPI controller |
| `gpio_` | GPIO |
| `uart_` | UART |
| `sensor_` | Sensor subsystem |

Identifiers with overlapping visibility must be typographically distinct. Do not redefine common macros (`MIN`, `MAX`, `ARRAY_SIZE`).

## 4. Commit Messages

### Format

```
<area>: <summary, max 72 chars>

<Body: explain what and why. 75-char line wrap. Must not be empty.>

Signed-off-by: Full Name <email@example.com>
Assisted-by: Claude:claude-opus-4.6
```

### Rules

- **Area prefixes:** `drivers: sensor:`, `net: ethernet:`, `Bluetooth: Shell:`, `dts:`, `doc:`, `style:`
- **Title:** Single line, max 72 characters, followed by a blank line.
- **Body:** Must not be empty. Each line 75 characters or less. Explain what and why.
- **Signed-off-by:** Mandatory (Developer Certificate of Origin). Only humans add this — AI cannot certify the DCO. Must use legal name and real email matching the commit author.
- **Assisted-by:** Required when AI tools were used. Format: `Assisted-by: <Agent>:<model-version>`. Basic dev tools (git, gcc, editors) are not listed.
- **No fixup or merge commits.** Each commit must build cleanly (bisectability).

## 5. Doxygen

Use `@` commands, not `\` (e.g., `@param` not `\param`).

### Parameter documentation

- `@param` in declaration order
- `@param[out]` for pointers the function writes to
- `@param[in,out]` for pointers the function both reads and writes
- Direction specifiers may be omitted for const pointers and scalars (implicitly input-only)

### Return values

- `@return` for general description
- `@retval` for specific discrete return values

### File headers

Each public header must have an `@file` block after the SPDX license notice, with `@brief` and belonging to a `@defgroup`.

### Example

```c
/* WRONG */
/**
 * \brief Read temperature
 * \param dev device
 * \return 0 on success
 */
```

```c
/* RIGHT */
/**
 * @brief Read temperature from sensor.
 *
 * @param dev Pointer to the device structure.
 * @param[out] val Pointer to store the sensor value.
 *
 * @retval 0 on success.
 * @retval -ENOTSUP if the channel is not supported.
 */
```

## 6. Kconfig Style

Indent with tabs. Help text uses one tab plus two extra spaces. 100-column line limit.

Symbol naming uses subsystem prefixes: drivers use `{TYPE}_{NAME}` (e.g., `SENSOR_BME280`), samples use `SAMPLE_`, tests use `TEST_`, boards use `BOARD_`, SoCs use `SOC_`.

Use `menuconfig` for feature enablement. Encapsulate dependent symbols within `if` blocks.

```kconfig
# WRONG — lowercase, space indent, wrong help format
config mysensor
  bool "My sensor"
  help
    Enable my sensor driver.
```

```kconfig
# RIGHT — uppercase, tab indent, help at tab + 2 spaces
config SENSOR_MY_SENSOR
	bool "MY-SENSOR temperature and humidity sensor"
	default y
	depends on DT_HAS_MYVENDOR_MY_SENSOR_ENABLED
	select I2C
	help
	  Enable driver for MyVendor MY-SENSOR.
```

## 7. Inclusive Language

Do not introduce new usage of these terms:

| Banned | Replacements |
|--------|-------------|
| master/slave | primary/secondary, controller/device, central/peripheral |
| blacklist/whitelist | denylist/allowlist, blocklist/allowlist |
| sanity (as in "sanity check") | coherence, confidence |

Exceptions: maintaining existing implementations of externally-governed industry standard specifications.

## References

- https://docs.zephyrproject.org/latest/contribute/index.html
- https://docs.zephyrproject.org/latest/contribute/guidelines.html
- https://docs.zephyrproject.org/latest/contribute/coding_guidelines/index.html
