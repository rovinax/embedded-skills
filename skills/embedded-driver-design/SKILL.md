---
name: embedded-driver-design
description: >
  Use when developing or reviewing peripheral drivers for embedded firmware (STM32, GD32,
  ESP32, FreeRTOS, drone flight controllers, robotics). Enforces layered driver architecture
  with bus abstraction, device context objects, register encapsulation, and strict separation
  between bus drivers, device drivers, BSP, and business modules. Prevents HAL/register leakage
  into device drivers and keeps control algorithms (PID, Kalman, attitude estimation) out of
  driver code.
---

# Embedded Driver Design

## Purpose

Design and review embedded drivers using a layered architecture.

The objective is to ensure:

- portability across MCU platforms
- testability via mock bus interfaces
- maintainability through stable interface contracts
- hardware abstraction that hides MCU details
- bus independence — device drivers talk to an interface, not a peripheral
- platform independence — drivers compile on STM32, GD32, or PC test harness unchanged

Drivers must hide hardware details behind stable interfaces. Business logic shall never access
MCU peripherals directly.

## Layer Architecture

The complete driver stack:

```text
APP
 ↓
MODULE  (business logic — estimator, controller, navigation)
 ↓
INTERFACE  (contracts only — uart_if.h, spi_if.h)
 ↓
DEVICE DRIVER  (icm42688, w25q64, bmp280)
 ↓
BUS DRIVER  (spi, i2c, uart, can)
 ↓
HAL / BSP  (vendor HAL -> platform abstraction)
 ↓
STM32 HAL / CMSIS
```

The forbidden pattern — IMU driver calling HAL directly:

```text
APP
 ↓
ICM42688  ← contains HAL_SPI_Transmit()
 ↓
HAL_SPI_Transmit()
```

This couples the device driver to one MCU family and makes porting, testing, and reuse impossible.

## Driver Classification Rules

Before generating driver code, classify the driver into exactly one of these categories.

### Bus Driver — `driver/bus/`

Responsible for bus protocol transaction on a specific MCU.

Examples: `SPI`, `I2C`, `UART`, `CAN`, `USB`, `ETH`

Directory: `driver/bus/spi/`, `driver/bus/i2c/`

### Device Driver — `driver/device/`

Responsible for a specific peripheral chip.

Examples: `ICM42688`, `MPU6050`, `W25Q64`, `BMP280`, `SX1278`

Directory: `driver/device/icm42688/`

### BSP Driver — `bsp/`

Board-level resources that map to specific hardware.

Examples: `LED`, `Button`, `Buzzer`, `OLED`

Directory: `bsp/` (not `driver/`)

Many AI tools incorrectly place board-level items like LED or Button into `driver/`. BSP items
belong under `bsp/`, not `driver/`.

### Driver Layer Responsibility Boundaries

Device drivers must **never** contain:

- PID controller logic
- Kalman filter implementation
- Attitude estimation algorithms
- Navigation algorithms
- Application state machines

Mixing driver code with control algorithms is a common architecture error in AI-generated
flight controller and robotics firmware. The IMU driver reads and converts sensor data.
The attitude estimator processes that data. Keep these in separate layers:

```text
driver/device/imu/          ← register reads, data conversion only
module/estimator/           ← Kalman, attitude fusion
module/controller/          ← PID, control law
```

## Required Directory Structure

```text
driver/
├── bus/
│   ├── spi/
│   ├── i2c/
│   ├── uart/
│   └── can/
├── device/
│   ├── icm42688/
│   │   ├── icm42688.c
│   │   ├── icm42688.h
│   │   └── icm42688_reg.h
│   ├── w25qxx/
│   │   ├── w25qxx.c
│   │   ├── w25qxx.h
│   │   └── w25qxx_reg.h
│   └── bmp280/
└── common/
    ├── error.h
    └── types.h
```

## Bus Abstraction Rules

**Every bus must expose an abstract interface.** Device drivers depend on these interfaces
only — never on vendor HAL types.

SPI bus interface:

```c
typedef struct {
    int32_t (*transfer)(const uint8_t *tx, uint8_t *rx, uint32_t len);
    int32_t (*transfer_async)(const uint8_t *tx, uint8_t *rx, uint32_t len,
                              void (*callback)(int32_t status));
} spi_if_t;
```

I2C bus interface:

```c
typedef struct {
    int32_t (*write)(uint8_t addr, const uint8_t *data, uint32_t len);
    int32_t (*read)(uint8_t addr, uint8_t *data, uint32_t len);
    int32_t (*write_read)(uint8_t addr, const uint8_t *cmd, uint32_t cmd_len,
                          uint8_t *rx, uint32_t rx_len);
} i2c_if_t;
```

A device driver may only access the bus through `spi_if_t` or `i2c_if_t`, never through
`SPI_HandleTypeDef` or `HAL_SPI_Transmit()` directly.

Forbidden in any device driver:

```c
SPI_HandleTypeDef hspi1;       // forbidden — MCU-specific type
HAL_SPI_Transmit(&hspi1, ...); // forbidden — vendor HAL API
```

Required — always go through the bus interface:

```c
spi->transfer(tx, rx, len);    // correct
```

## Device Context Rules

Driver state must live in a device context struct, not in global variables.

Forbidden — global device state:

```c
static float gyro_x;
static float gyro_y;
static uint8_t initialized;  // forbidden
```

Required — device context:

```c
typedef struct {
    spi_if_t      *spi;
    float          gyro[3];
    float          accel[3];
    uint8_t        initialized;
    uint32_t       timeout_ms;
} icm42688_t;
```

All state belongs to the device object. This enables multiple instances of the same device
on different bus channels.

## Initialization Rules

Unified init interface — all device drivers must follow this signature:

```c
error_t xxx_init(xxx_t *dev);
```

Example:

```c
error_t icm42688_init(icm42688_t *imu);
```

Forbidden:

```c
void imu_init(void);  // hidden global state — cannot determine which instance
```

## Register Encapsulation Rules

Register addresses and bit-field definitions must be centralized in a `_reg.h` header.

Forbidden — magic register addresses scattered in driver code:

```c
spi_write(0x1A, 0x80);   // what is 0x1A? what is 0x80?
```

Required — named register constants:

```c
// icm42688_reg.h
#define ICM42688_REG_PWR_MGMT0    (0x4EU)
#define ICM42688_BIT_TEMP_EN      (0x01U << 4)

// icm42688.c
ret = icm42688_write_reg(imu, ICM42688_REG_PWR_MGMT0, ICM42688_BIT_TEMP_EN);
```

## Synchronization and DMA Rules

Device drivers must not be aware of DMA streams, DMA channels, or IRQ numbers. These belong
to the bus driver layer.

The bus interface exposes both synchronous and asynchronous transfer functions:

```c
spi->transfer(tx, rx, len);           // synchronous — blocks until done
spi->transfer_async(tx, rx, len, cb); // asynchronous — callback on completion
```

The device driver chooses the mode but does not configure DMA hardware directly.

Forbidden in device drivers:

```c
DMA_HandleTypeDef hdma_spi1_tx;  // forbidden — DMA is a bus-layer concern
__HAL_DMA_ENABLE(...);           // forbidden
```

## RTOS Isolation Rules

Device drivers must be RTOS-agnostic. They must not call:

```c
osDelay();          // forbidden — introduces RTOS dependency
vTaskDelay();       // forbidden
osMutexAcquire();   // forbidden
```

Device drivers must not busy-wait:

```c
while (!(reg & BIT_READY));  // forbidden — spin-wait blocks the entire system
```

Instead, device drivers should support timeout, callback, or event mechanisms:

```c
ret = spi->transfer(tx, rx, len);
if (ret == ERR_TIMEOUT) {
    // handle timeout — caller decides retry or recovery
}
```

RTOS concerns (blocking, yielding, mutex protection) are handled by the layer *above* the
device driver — the module or app layer coordinates access if multiple tasks share a device.

## Error Handling Rules

All drivers must use a unified error code type:

```c
typedef enum {
    ERR_OK = 0,
    ERR_TIMEOUT,
    ERR_BUSY,
    ERR_INVALID_PARAM,
    ERR_NOT_READY,
    ERR_COMMUNICATION,
} error_t;
```

Forbidden:

```c
return -1;  // meaningless — what failed?
return -2;
return -3;
```

## Mock Testing Rules

Every bus interface must be mockable. The bus abstraction (`spi_if_t`, `i2c_if_t`) design
enables this naturally — the same interface can point to real hardware or a test stub:

```c
// real hardware
spi_if_t spi = { .transfer = stm32_spi_transfer };

// test mock
spi_if_t mock_spi = { .transfer = mock_spi_transfer };
```

This allows device drivers to be tested on a PC without any MCU hardware:

- compile for host (x86/ARM64)
- inject `mock_spi` that returns known patterns
- verify register sequences and data conversion logic

## Validation Checklist

After generating driver code, verify:

- [ ] Device driver contains no HAL API calls
- [ ] Device driver contains no MCU-specific types
- [ ] Device driver depends on bus interface, not MCU bus peripheral
- [ ] No magic register addresses — all in `_reg.h`
- [ ] Register bit definitions are named constants
- [ ] Device state is in a context struct, not globals
- [ ] No global `static` device state
- [ ] Init interface follows `error_t xxx_init(xxx_t *dev)` pattern
- [ ] Error codes are typed (`error_t`), not magic `-1`/`-2`
- [ ] No DMA configuration inside device driver
- [ ] No IRQ numbers inside device driver
- [ ] No RTOS calls (`osDelay`, `vTaskDelay`, mutex) in driver code
- [ ] No busy-wait loops
- [ ] Bus interface supports mock injection for testing
- [ ] Driver contains no PID, Kalman, or control algorithm logic

## Auto Fix Strategy

When violations are found:

1. **Identify** HAL or register leakage into the device driver
2. **Extract** the bus operations into a `spi_if_t` / `i2c_if_t` interface
3. **Move** register constants into a dedicated `_reg.h` header
4. **Encapsulate** global device state into a context struct
5. **Replace** magic error codes with `error_t` enum values
6. **Remove** DMA/IRQ/RTOS dependencies from the device layer
7. **Separate** control algorithms into `module/` — drivers read data, modules process it
