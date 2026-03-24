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

```bash
# J-Link (most Nordic, some NXP boards)
west debug --runner jlink

# OpenOCD (most STM32, RP2040, many others)
west debug --runner openocd

# If runner auto-detection fails, check board.cmake:
cat $ZEPHYR_BASE/boards/<vendor>/<board>/board.cmake
```

### VSCode cortex-debug

Create `.vscode/launch.json`:

```json
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
```

**Finding SVD files:** SVD files enable peripheral register viewing in cortex-debug.
- CMSIS-Pack: https://developer.arm.com/tools-and-software/embedded/cmsis/cmsis-packs
- Community: https://github.com/posborne/cmsis-svd
- Vendor SDKs (STM32CubeMX, nRF Connect SDK, MCUXpresso)

**Build task** (`.vscode/tasks.json`):
```json
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
```

## Stack Overflow Detection

```kconfig
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
```

Thread analyzer output:
```
Thread: my_thread  Stack size: 1024  Used: 856  Unused: 168  Usage: 83%
```
Rule of thumb: if usage > 70%, increase stack size.

## Memory Corruption Debugging

1. **Enable MPU stack protection** — catches stack overflows at the moment they happen
2. **Use thread analyzer** — find threads close to stack limit
3. **Enable ASAN on native_sim** — catches buffer overflows, use-after-free:
   ```kconfig
   CONFIG_ASAN=y
   ```
4. **Hardware watchpoint on corrupted address:**
   ```gdb
   watch *(uint32_t *)0x20001234
   ```
5. **Memory fill patterns** — Zephyr fills unused stack with `0xAA`, check for corruption

## GDB Commands for Embedded

```gdb
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
```

## Kconfig for Maximum Debug Info

```kconfig
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
```
