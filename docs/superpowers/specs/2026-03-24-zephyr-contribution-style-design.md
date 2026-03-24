# Zephyr Contribution Style Skill

New skill `zephyr-contribution-style` that teaches the LLM Zephyr upstream coding standards, plus a small update to `zephyr-driver-development` to reference it.

## Motivation

When generating code for the Zephyr source tree (drivers, subsystems, tests), the LLM's defaults conflict with Zephyr's coding standards in predictable ways: it uses `//` comments, indents with spaces, omits braces on single-line blocks, and doesn't know the commit message format. These are caught by CI (checkpatch, clang-format) but waste contributor time on fixups.

A skill that front-loads these rules — filtered to only what the LLM would get wrong — produces upstream-ready code on the first pass.

## Filtering Principle

Only include rules where **Zephyr's convention differs from what the LLM would naturally produce**, or where **Zephyr-specific knowledge is required** (Kconfig symbols, subsystem prefixes, commit trailer format). General embedded best practices the LLM already knows (e.g., "check malloc return for NULL") are omitted.

## Deliverables

### 1. New skill: `zephyr-contribution-style`

Location: `skills/zephyr-contribution-style/SKILL.md`

**Trigger:** Writing or modifying code in the Zephyr source tree — drivers, subsystems, kernel, tests, documentation. Not for application code built on top of Zephyr (that's `zephyr-app-development`).

**Dependencies:** None. This is a leaf skill that other Zephyr source-level skills reference.

**Size target:** ~300 lines (rules + do/don't example pairs).

### 2. Update: `zephyr-driver-development`

- Add `zephyr-contribution-style` to the `REQUIRED BACKGROUND` list
- Remove the upstreaming prerequisites checklist (currently lines 283-289) since those checks are now covered by the new skill
- Keep the upstream workflow steps (fork, branch, push, PR) — that's process, not style

## Skill Content: Section-by-Section

### Section 1: C Code Style

Five rules, each with a WRONG/RIGHT code example.

**Tabs, 8-char width.** The LLM defaults to spaces or 4-space tabs.
```c
/* WRONG */
    if (ret) {
        return ret;
    }

/* RIGHT */
→       if (ret) {
→       →       return ret;
→       }
```

**100-column line limit.** The LLM often targets 80.

**C89 comments only.** The LLM defaults to `//`.
```c
/* WRONG */
// initialize the device

/* RIGHT */
/* initialize the device */
```

**Braces always required.** Even on single-line bodies.
```c
/* WRONG */
if (ret)
    return ret;

/* RIGHT */
if (ret) {
    return ret;
}
```

**No binary literals.** The LLM uses `0b...` for register bitmasks.
```c
/* WRONG */
uint8_t mask = 0b00110000;

/* RIGHT */
uint8_t mask = 0x30;
```

### Section 2: MISRA-C Subset

Ten rules the LLM is likely to violate without being told. Each with a brief rationale and example where useful.

1. **No recursion.** LLM reaches for recursive solutions naturally.

2. **No variadic functions.** No `<stdarg.h>`.

3. **Dynamic allocation.** Upstream Zephyr forbids stdlib `malloc`/`free` (MISRA Dir 4.12). Only use them if the project has `CONFIG_COMMON_LIBC_MALLOC=y` with `CONFIG_COMMON_LIBC_MALLOC_ARENA_SIZE != 0`. Note: `k_malloc`/`k_free` use a separate kernel heap controlled by `CONFIG_HEAP_MEM_POOL_SIZE`.

4. **Error returns must be tested.** Every function that returns an error code must have its return value checked.
```c
/* WRONG */
i2c_reg_write_byte_dt(&cfg->i2c, REG_CTRL, val);

/* RIGHT */
int ret = i2c_reg_write_byte_dt(&cfg->i2c, REG_CTRL, val);
if (ret) {
    return ret;
}
```

5. **`switch` must have `default`.** Every switch statement needs a default label.

6. **`if-else if` chains must end with `else`.** Even if the else body is empty or just a comment.
```c
/* WRONG */
if (chan == SENSOR_CHAN_AMBIENT_TEMP) {
    /* ... */
} else if (chan == SENSOR_CHAN_HUMIDITY) {
    /* ... */
}

/* RIGHT */
if (chan == SENSOR_CHAN_AMBIENT_TEMP) {
    /* ... */
} else if (chan == SENSOR_CHAN_HUMIDITY) {
    /* ... */
} else {
    return -ENOTSUP;
}
```

7. **`goto` only forward, same function.** Used for error cleanup paths, never backward.

8. **Pointer nesting max 2 levels.** No `***p`.

9. **No cast removing `const`/`volatile`.** Don't cast away qualifiers for convenience.

10. **`NULL` is the only null pointer constant.** Not `(void *)0` or bare `0`.

### Section 3: Naming Conventions

- All identifiers use `snake_case`
- Public APIs must use subsystem prefixes:

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

- Identifiers with overlapping visibility must be typographically distinct
- Do not redefine common macros (`MIN`, `MAX`, `ARRAY_SIZE`)

### Section 4: Commit Messages

Format template and rules:
```
<area>: <summary, max 72 chars>

<Body: explain what and why. 75-char line wrap. Must not be empty.>

Signed-off-by: Full Name <email@example.com>
Assisted-by: Claude <tool-identifier>
```

- Area prefixes: `drivers: sensor:`, `net: ethernet:`, `Bluetooth: Shell:`, `dts:`, `doc:`, `style:`
- `Signed-off-by` is mandatory (DCO). Only humans add this — AI cannot certify DCO.
- `Assisted-by` is required when AI tools were used.
- No fixup or merge commits.
- Each commit must build cleanly (bisectability).

### Section 5: Doxygen

- Use `@` commands, not `\` (i.e., `@param` not `\param`)
- `@param` in declaration order; use `@param[in]`, `@param[out]`, `@param[in,out]` for pointers
- `@return` for general description, `@retval` for specific values
- File headers: `@file`, `@brief`, `@ingroup`
- Units: SI with space (`10 ms` not `10ms`)

```c
/* WRONG */
/**
 * \brief Read temperature
 * \param dev device
 * \return 0 on success
 */

/* RIGHT */
/**
 * @brief Read temperature from sensor.
 *
 * @param[in] dev Pointer to the device structure.
 * @param[out] val Pointer to store the sensor value.
 *
 * @retval 0 on success.
 * @retval -ENOTSUP if the channel is not supported.
 */
```

### Section 6: Kconfig Style

- 100-column line limit
- Indent with tabs
- Help text: one tab + two spaces
- Symbol naming uses subsystem prefixes: `SENSOR_`, `TEST_`, `BOARD_`, `SOC_`, `SAMPLE_`
- Use `menuconfig` for feature enablement; encapsulate dependents in `if` blocks

```kconfig
# WRONG
config mysensor
  bool "My sensor"
  help
    Enable my sensor driver.

# RIGHT
config MY_SENSOR
	bool "MY-SENSOR temperature and humidity sensor"
	default y
	depends on DT_HAS_MYVENDOR_MY_SENSOR_ENABLED
	select I2C
	help
	  Enable driver for MyVendor MY-SENSOR.
```

### Section 7: Inclusive Language

Banned terms with required replacements:

| Banned | Replacement |
|--------|-------------|
| master/slave | primary/secondary, controller/device, central/peripheral |
| blacklist/whitelist | denylist/allowlist |
| sanity (as in "sanity check") | coherence, confidence |

## What's Excluded (and Why)

- **Full 147 MISRA rules** — Most are already in the LLM's training data or rarely triggered. Only the ~15 most likely to be violated are included.
- **Contribution process guidance** — PR etiquette, reviewer interaction, RFC workflow. Out of scope; this skill is about code quality, not process.
- **General embedded best practices** — "Check malloc for NULL", "prefer stack allocation". The LLM already knows these.
- **KeepSorted, Coverity, CI details** — Runtime tooling the contributor interacts with directly, not something the LLM needs to encode.

## Sources

The rules in this spec are derived from the Zephyr contribution documentation. The implementing LLM should consult these for exact details and edge cases:

- Contributing overview: https://docs.zephyrproject.org/latest/contribute/index.html
- Contribution guidelines: https://docs.zephyrproject.org/latest/contribute/guidelines.html
- Coding guidelines: https://docs.zephyrproject.org/latest/contribute/coding_guidelines/index.html
