---
name: embedded-documentation
description: >
  Use when creating or reviewing embedded firmware projects (STM32, FreeRTOS, drone, robotics) to
  ensure complete, consistent, and maintainable documentation. Automatically generates and validates
  project documentation: README, architecture docs, module design docs, interface specs, RTOS task
  design, ISR documentation, state machines, hardware pin mapping, and data flow diagrams.
  Triggers on new projects, new modules, new RTOS tasks/ISRs, state machine detection, or any
  undocumented component.
---

# Embedded Documentation

## Purpose

This skill defines documentation standards for embedded firmware projects. The goal is to ensure
every generated project is accompanied by documentation sufficient for another engineer to
understand, build, modify, and maintain the system.

Goals:

- maintainability
- traceability
- onboarding efficiency
- architecture visibility
- design consistency

The most valuable embedded documentation is not Doxygen API comments — it is architecture overviews,
RTOS task design, and hardware resource mapping that let a new engineer understand the whole
system in minutes.

Documentation must evolve together with code. Never leave public modules undocumented.

## Documentation Level Overview

| Level | Content | When Generated |
|-------|---------|----------------|
| L1 — Project | README, Architecture, Build, Hardware, Directory Structure | New project |
| L2 — Module | Per-module design docs | New module created |
| L3 — Interface | Interface contract docs | New interface created |
| L4 — RTOS/ISR | TaskDesign, ISRDesign, EventFlow | `osThreadNew` or `IRQHandler` detected |
| L5 — Control | ControlLoop, StateMachine, Estimator | State machine enum or control structures detected |

## Project-Level Documentation (L1)

When creating a new embedded project, generate:

```text
README.md

docs/
├── Architecture.md
├── Build.md
├── Hardware.md
└── DirectoryStructure.md
```

### README Requirements

Every project README must contain:

```markdown
# Project Overview

What this firmware does and which hardware it targets.

# Features

Key capabilities and supported peripherals.

# Hardware Platform

MCU family, board revision, key pin assignments.

# Build Instructions

Toolchain, build commands, optional features.

# Flash Instructions

Debugger, probe, or bootloader method.

# Project Structure

Overview of the directory layout.

# Software Architecture

Layer diagram and dependency direction.

# Dependencies

Middleware, SDKs, toolchains required.

# License
```

### Architecture Documentation

Must describe the layer model — a ASCII dependency diagram:

```text
app/
 ↓
module/
 ↓
interface/
 ↓
bsp/
 ↓
platform/
```

Explain the role of each layer and the dependency direction constraint.

### Directory Structure Documentation

Auto-generate from the actual project tree:

```text
project/
├── app/              — Application entry, tasks, state machines
├── module/           — Business logic modules
├── interface/        — Hardware abstraction contracts
├── bsp/              — Board-specific driver implementations
├── platform/         — Vendor SDK, HAL, CMSIS, linker scripts
├── middleware/       — Third-party (FreeRTOS, lwIP, FatFS)
├── docs/             — Documentation
├── tools/            — Build and debug scripts
└── tests/            — Unit and integration tests
```

### Hardware Documentation

When the project references GPIO, UART, SPI, I2C, CAN, PWM, or ADC, generate `docs/Hardware.md`
with a resource table:

```markdown
| Peripheral | Interface | Target       | Purpose          |
|------------|-----------|--------------|------------------|
| USART1     | uart_if   | Telemetry    | 115200 baud      |
| SPI1       | spi_if    | ICM-42688    | IMU data @ 1MHz  |
| TIM3       | pwm_if    | Motor driver | 50Hz PWM         |
| ADC1       |           | Battery      | Voltage monitor  |
| PB0        | gpio_if   | LED          | Status indicator |
```

## Module-Level Documentation (L2)

When a new module is created (e.g., `module/motor/`), generate `docs/modules/motor.md`:

```markdown
# Motor Module

## Purpose

Control motor output — converts speed commands into PWM signals and monitors fault status.

## Responsibilities

- speed control loop execution
- PWM generation via pwm_if
- fault detection and safe stop
- current limiting

## Public API

error_t motor_init(void)
error_t motor_set_speed(int32_t rpm)
void    motor_stop(void)
bool    motor_is_faulted(void)

## Dependencies

- pwm_if
- timer_if
- gpio_if

## Data Flow

control_module → motor_module → pwm_if → platform_pwm

## State Machine

STOPPED → STARTING → RUNNING → FAULT → STOPPED
```

## Interface Documentation (L3)

Every interface in `interface/` must document:

```markdown
# UART Interface

## Purpose

Provide hardware-independent UART access for modules.

## Public API

int32_t uart_init(uart_config_t *cfg)
int32_t uart_write(const uint8_t *data, uint32_t len)
int32_t uart_read(uint8_t *data, uint32_t len)

## Ownership

Implemented by BSP layer. Consumed by module/ layer.

## Thread Safety

Not thread-safe. Caller must protect with mutex or critical section.
```

## RTOS Task Documentation (L4)

When the code contains `osThreadNew()` or `xTaskCreate()`, generate `docs/TaskDesign.md`:

```markdown
# Task Design

| Task            | Period | Priority | Stack  | Responsibility          |
|-----------------|--------|----------|--------|-------------------------|
| imu_task        | 1 ms   | High     | 512    | Sensor acquisition      |
| control_task    | 1 ms   | High     | 1024   | PID control loop        |
| telemetry_task  | 20 ms  | Low      | 256    | UART telemetry output   |
| usb_cdc_task    | event  | Medium   | 512    | USB serial handling     |
```

Also document synchronization primitives:

```markdown
### Synchronization

| Resource         | Type         | Purpose                  |
|------------------|--------------|--------------------------|
| imu_data_queue   | MessageQueue | IMU → Control            |
| control_mutex    | Mutex        | Shared setpoint access   |
| telemetry_event  | EventFlags   | Telemetry trigger        |
```

## ISR Documentation (L4)

When the code contains `IRQHandler` functions, generate `docs/ISRDesign.md`:

```markdown
# ISR Design

| Interrupt      | Source   | Priority | Execution Time | Shared Variables        |
|----------------|----------|----------|----------------|-------------------------|
| USART1_IRQn    | RX       | 5        | < 5 µs         | rx_buffer, rx_index     |
| TIM3_IRQn      | Update   | 4        | < 2 µs         | pwm_duty                |

### Deferred Processing

| ISR            | Deferred Mechanism     | Consumer      |
|----------------|------------------------|---------------|
| USART1_IRQn    | osMessageQueuePut      | usb_cdc_task  |
| TIM3_IRQn      | Direct (in-isr update) | —             |
```

## State Machine Documentation (L5)

When the code contains a state enum or `switch(state)`, generate `docs/StateMachine.md`:

```text
INIT
 ↓
IDLE
 ↓
RUNNING
 ↓
FAULT → IDLE
```

Include transition conditions:

```markdown
| Transition | From     | To       | Condition               |
|------------|----------|----------|-------------------------|
| start      | IDLE     | RUNNING  | system_ready == true    |
| fault      | RUNNING  | FAULT    | motor_fault_detected    |
| reset      | FAULT    | IDLE     | fault_ack received      |
| error      | INIT     | IDLE     | init complete           |
```

## Diagram Generation Rules

When producing documentation, prefer ASCII diagrams for architecture and data flow so they
render inline in any Markdown viewer without external tools.

### Architecture Diagram

```text
app/
 ↕
module/
 ↕
interface/
 ↕
bsp/
 ↕
platform/
```

### Data Flow Diagram

```text
imu_sensor
    ↓ (SPI)
imu_module
    ↓ (queue)
estimator
    ↓ (shared data)
controller
    ↓ (pwm_if)
motor_driver
```

### Task Interaction Diagram

```text
imu_task ──queue──→ control_task ──queue──→ telemetry_task
```

## Documentation Validation Checklist

After generating code, verify:

- [ ] README exists and covers all required sections
- [ ] Architecture document describes layer model
- [ ] Build instructions are documented
- [ ] Hardware resources are documented with peripheral table
- [ ] New modules each have a design doc
- [ ] All interfaces document purpose, API, ownership, and thread safety
- [ ] RTOS tasks documented with period/priority/stack/responsibility
- [ ] ISRs documented with source, latency, shared variables, deferred processing
- [ ] State machines documented with transitions and conditions
- [ ] Dependency graph and data flow documented
- [ ] Directory structure documented

## Auto Fix Strategy

When documentation is missing:

1. **Detect** the undocumented component (module, interface, task, ISR, hardware)
2. **Generate** the required document following the templates above
3. **Link** the new document from the README table of contents
4. **Update** architecture references to maintain document graph coherence

Documentation should evolve together with code. Never leave public modules undocumented.
