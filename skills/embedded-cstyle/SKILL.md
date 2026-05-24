---
name: embedded-cstyle
description: >
  Use when generating, modifying, reviewing, or refactoring C code for embedded firmware projects
  (STM32, GD32, ESP32, FreeRTOS, bare-metal, driver development, robotics). Enforces engineering
  coding standards — naming, types, functions, variables, macros, headers, memory, concurrency,
  error handling, and documentation. Catches AI-common mistakes: global variables, magic numbers,
  missing static/const, volatile abuse, ignored return values, unchecked null pointers, and HAL
  calls leaking into business logic.
---

# Embedded C Style

## Purpose

This skill defines mandatory coding standards for embedded firmware projects.

Goals:

- readability
- maintainability
- portability
- safety
- deterministic behavior

The style is inspired by MISRA-C, Linux Kernel Style, AUTOSAR C Guidelines, and STM32 firmware best practices — adapted for practical use in small-to-medium embedded projects rather than rigid compliance.

This skill applies whenever generating, modifying, reviewing, or refactoring C code.

Priority order when making trade-offs: **Safety > Determinism > Maintainability > Readability**.

## Interface Boundary Rule

Application and business modules shall never call HAL APIs directly.

Forbidden in `app/` and `module/`:

```c
HAL_GPIO_WritePin()
HAL_UART_Transmit()
HAL_SPI_Transmit()
HAL_I2C_Master_Transmit()
```

Only `interface/` and `bsp/` layers may access vendor HAL APIs.

## Naming Rules

### Type Naming

Format: `typename_t`

Required:

```c
motor_t
imu_t
pid_t
uart_config_t
```

Forbidden:

```c
Motor
IMU
PIDStruct
UART_CONFIG
```

### Function Naming

Format: `module_action()` — lowercase, underscore-separated.

Required:

```c
motor_init()
motor_set_speed()
imu_read()
pid_update()
```

Forbidden:

```c
InitMotor()
ReadIMU()
PID()
```

Internal (file-scope) functions must be declared `static`:

```c
static void motor_update(void);
```

### Macro Naming

Format: `MODULE_DESCRIPTION` — uppercase, underscore-separated.

Required:

```c
MOTOR_MAX_SPEED_RPM
UART_RX_BUFFER_SIZE
PID_DEFAULT_KP
```

Forbidden:

```c
MaxSpeed
bufferSize
```

## Type Rules

Always use `<stdint.h>` fixed-width types.

Forbidden:

```c
int
short
long
unsigned
```

Required:

```c
uint8_t
uint16_t
uint32_t
int8_t
int16_t
int32_t
```

Booleans must use `<stdbool.h>`:

```c
#include <stdbool.h>

bool ready;
```

Forbidden as boolean substitutes:

```c
u8 flag;
uint8_t status;
```

## Variable Rules

### Global Variables

Plain global variables are forbidden:

```c
int speed;  // forbidden
```

File-private statics are allowed:

```c
static int32_t speed;  // allowed
```

Cross-module access must go through interface functions:

```c
motor_get_speed()  // correct
```

Not:

```c
extern int speed;  // forbidden
```

### `const` Preference

Immutable data must be declared `const`:

```c
static const uint8_t crc_table[256];
static const float pid_gains[3] = {1.0f, 0.1f, 0.05f};
```

### `volatile` Discipline

`volatile` is only allowed for:

- ISR-shared variables
- DMA status variables
- Memory-mapped peripheral registers

```c
volatile bool uart_rx_done;  // correct
```

Forbidden on ordinary variables:

```c
volatile int count;  // forbidden
```

## Function Rules

### Parameter Validation

Pointer parameters must be checked before dereference:

```c
if (ptr == NULL) {
    return ERR_INVALID_PARAM;
}
```

Forbidden:

```c
ptr->data = 1;  // no null check
```

### Return Value Checking

Callers must check return values:

```c
ret = uart_write(...);
if (ret != ERR_OK) {
    return ret;
}
```

Forbidden:

```c
uart_write(...);  // return value ignored
```

## Macro Rules

### No Magic Numbers

Forbidden:

```c
if (speed > 123)
```

Required:

```c
#define MOTOR_MAX_SPEED_RPM (123U)

if (speed > MOTOR_MAX_SPEED_RPM)
```

### Parenthesize Macro Bodies

Forbidden:

```c
#define MAX_SPEED 100 + 20
```

Required:

```c
#define MAX_SPEED (120U)
```

## Header Rules

### Include Guard

Every header must use either:

```c
#ifndef MOTOR_H
#define MOTOR_H

// ...

#endif
```

or:

```c
#pragma once
```

### Minimize Header Dependencies

Prefer forward declarations over `#include` in headers:

```c
struct motor;  // forward declaration
```

Rather than:

```c
#include "a.h"
#include "b.h"
#include "c.h"
#include "d.h"
```

## Memory Rules

### No Dynamic Allocation

Runtime heap allocation is forbidden:

```c
malloc()
calloc()
realloc()
free()
```

Use static allocation instead:

```c
static uint8_t rx_buffer[256];
```

This prevents heap fragmentation and unpredictable allocation latency — critical for real-time embedded systems.

## Concurrency Rules

### Critical Sections

Shared resources accessed from multiple threads or ISRs must be protected:

```c
osMutexAcquire(mutex_id, timeout);
// ... access shared resource ...
osMutexRelease(mutex_id);
```

or:

```c
taskENTER_CRITICAL();
// ... access shared resource ...
taskEXIT_CRITICAL();
```

## Error Handling Rules

### Unified Error Codes

Use a single project-wide error enum rather than magic return values:

```c
typedef enum {
    ERR_OK = 0,
    ERR_TIMEOUT,
    ERR_BUSY,
    ERR_INVALID_PARAM,
    ERR_NOT_INITIALIZED,
    ERR_OVERFLOW,
    ERR_UNDERFLOW,
} error_t;
```

Forbidden:

```c
return -1;
return -2;
return -3;
```

## Documentation Rules

All public interfaces must carry a Doxygen-style doc block:

```c
/**
 * @brief Set motor speed
 *
 * @param rpm Target speed in revolutions per minute
 *
 * @return error_t ERR_OK on success
 */
error_t motor_set_speed(int32_t rpm);
```

## Validation Checklist

After generating or reviewing code, verify:

- [ ] `<stdint.h>` types used — no bare `int`, `short`, `long`, `unsigned`
- [ ] `<stdbool.h>` used for booleans
- [ ] Internal functions declared `static`
- [ ] No unchecked pointer dereference
- [ ] No ignored return values
- [ ] No runtime `malloc`/`free`
- [ ] No magic numbers — all constants are named macros
- [ ] Macro bodies parenthesized
- [ ] Global variables minimized; cross-module access via functions
- [ ] `const` used wherever data is immutable
- [ ] `volatile` used only for ISR/DMA/register variables
- [ ] Naming follows `module_action`, `typename_t`, `MACRO_NAME` conventions
- [ ] Public interfaces have Doxygen doc blocks
- [ ] No HAL calls in `app/` or `module/`

## Auto Fix Strategy

When violations are found:

1. **Report** the violation with file and line
2. **Explain** the embedded impact (safety, determinism, or portability risk)
3. **Provide** corrected code inline
4. **Refactor** automatically if the fix is mechanical

Priority order for fixes:

```
Safety > Determinism > Maintainability > Readability
```
