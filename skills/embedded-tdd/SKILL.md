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

```
NO FIRMWARE CODE WITHOUT A FAILING TEST FIRST
```

Same exceptions as superpowers:test-driven-development. No additional exceptions for embedded.

"But I can't test hardware interactions without hardware" — yes you can. native_sim, QEMU, mocks, and fakes exist. Only the final HIL validation requires real hardware.

## Test Tiers

| Tier | Tool | Target | Speed | What it catches |
|------|------|--------|-------|-----------------|
| Unit | ztest + native_sim | Host or QEMU | Seconds | Logic bugs, API misuse, edge cases |
| HIL | pytest + serial | Real hardware | Minutes+ | Peripheral behavior, timing, electrical |

## Tier 1: Unit Tests with ztest

### Test project structure

```
tests/unit/my_module/
├── CMakeLists.txt
├── prj.conf
├── testcase.yaml
└── src/
    └── main.c
```

### CMakeLists.txt

```cmake
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
```

### prj.conf

```kconfig
CONFIG_ZTEST=y
CONFIG_ZTEST_NEW_API=y
```

### Test code

```c
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
```

### testcase.yaml

```yaml
tests:
  my_app.unit.my_module:
    platform_allow:
      - native_sim
      - qemu_cortex_m3
    tags: unit
    integration_platforms:
      - native_sim
```

### Running unit tests

```bash
# Build and run on native_sim
west build -b native_sim tests/unit/my_module -p auto && west build -t run

# Run specific test suite
west build -b native_sim tests/unit/my_module -p auto -- -DCONFIG_ZTEST_SHUFFLE=y
```

### Mocking hardware dependencies

For code that calls Zephyr driver APIs, create thin wrappers:

```c
/* app_gpio.h — testable wrapper */
int app_gpio_set(const struct gpio_dt_spec *spec, int value);
int app_gpio_get(const struct gpio_dt_spec *spec);
```

```c
/* app_gpio.c — real implementation */
#include <zephyr/drivers/gpio.h>
#include "app/app_gpio.h"

int app_gpio_set(const struct gpio_dt_spec *spec, int value)
{
    return gpio_pin_set_dt(spec, value);
}
```

```c
/* test_doubles/fake_gpio.c — test double */
static int fake_pin_state[32];

int app_gpio_set(const struct gpio_dt_spec *spec, int value)
{
    fake_pin_state[spec->pin] = value;
    return 0;
}

/* Test helper */
int fake_gpio_get_state(int pin) { return fake_pin_state[pin]; }
```

Link the fake in your test CMakeLists.txt instead of the real implementation.

## Tier 2: HIL Testing with pytest

### Architecture

```
pytest (host)  ──serial/USB──►  DUT (device under test)
    │                              │
    ├── sends commands ──────────► parses & executes
    ├── reads responses ◄──────── sends results
    └── asserts on behavior        runs firmware test shell
```

### Firmware side: Zephyr shell

```kconfig
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
```

```c
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
```

### pytest side

```python
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
```

```python
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
```

### Running HIL tests

```bash
pytest tests/hil/ -v --serial-port /dev/ttyACM0
pytest tests/hil/ -v --junitxml=reports/hil.xml   # CI report
```

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
