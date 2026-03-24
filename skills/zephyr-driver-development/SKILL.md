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
- zephyr-contribution-style (upstream coding standards, for in-tree contributions)

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

```
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
```

### In-tree (for upstream)

```
zephyr/
├── drivers/sensor/my_sensor/
│   ├── CMakeLists.txt
│   ├── Kconfig
│   └── my_sensor.c
├── dts/bindings/sensor/
│   └── myvendor,my-sensor.yaml
└── tests/drivers/sensor/my_sensor/
```

## Devicetree Binding

```yaml
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
```

**Key rules:**
- `compatible` must match `DT_DRV_COMPAT` (commas→underscores, hyphens→underscores)
- `include` base bindings for the subsystem
- Vendor prefix must be in `dts/bindings/vendor-prefixes.txt` for upstream

## Driver Implementation (I2C Sensor)

```c
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
```

### Kconfig

```kconfig
config MY_SENSOR
    bool "MY-SENSOR temperature and humidity sensor"
    default y
    depends on DT_HAS_MYVENDOR_MY_SENSOR_ENABLED
    select I2C
    select SENSOR
    help
      Enable driver for MyVendor MY-SENSOR.
```

### CMakeLists.txt

```cmake
zephyr_library()
zephyr_library_sources(my_sensor.c)
```

### Parent integration (in-tree)

```kconfig
# drivers/sensor/Kconfig — add:
rsource "my_sensor/Kconfig"
```

```cmake
# drivers/sensor/CMakeLists.txt — add:
add_subdirectory_ifdef(CONFIG_MY_SENSOR my_sensor)
```

## Testing with Twister

### testcase.yaml

```yaml
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
```

### Running twister

```bash
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
```

## Upstreaming to Zephyr

### Workflow

1. **Fork and branch:**
   ```bash
   cd zephyr
   git remote add my-fork https://github.com/<you>/zephyr.git
   git fetch origin
   git checkout -b add-my-sensor-driver origin/main
   ```

2. **Add driver files** (binding, source, Kconfig, CMake, tests)

3. **Run checkpatch:**
   ```bash
   git format-patch origin/main
   $ZEPHYR_BASE/scripts/checkpatch.pl *.patch
   ```

4. **Run tests:**
   ```bash
   west twister -T tests/drivers/sensor/my_sensor/ -p native_sim
   ```

5. **Push and PR:**
   ```bash
   git push my-fork add-my-sensor-driver
   gh pr create --repo zephyrproject-rtos/zephyr \
       --title "drivers: sensor: add MY-SENSOR driver" \
       --body "Add driver for MyVendor MY-SENSOR."
   ```

### West manifest for fork-based development

```yaml
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
```

### Managing out-of-tree during review

While PR is pending, keep driver in your west module. Once merged:
1. Update `west.yml` to Zephyr revision containing your driver
2. Remove driver from out-of-tree module
3. Update Kconfig/overlay to use in-tree driver
