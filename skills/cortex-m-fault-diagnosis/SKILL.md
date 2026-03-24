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

```gdb
p/x *(uint32_t *)0xE000ED28     # CFSR (Configurable Fault Status Register)
p/x *(uint32_t *)0xE000ED2C     # HFSR (HardFault Status Register)
p/x *(uint32_t *)0xE000ED34     # MMAR (MemManage Fault Address)
p/x *(uint32_t *)0xE000ED38     # BFAR (BusFault Address)
```

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

```
***** MPU FAULT *****
  Data Access Violation
  MMFAR Address: 0x00000000
***** HARD FAULT *****
Current thread: 0x20001234 (my_thread)
```

**Enable maximum fault info:**
```kconfig
CONFIG_FAULT_DUMP=2
CONFIG_EXTRA_EXCEPTION_INFO=y
CONFIG_RESET_ON_FATAL_ERROR=n    # Halt instead of reset (for debugging)
```

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
