# Project Context

This repository contains a reusable embedded firmware engineering framework for AI-assisted development.

The framework targets resource-constrained MCU systems including:

- STM32
- GD32
- ESP32
- RP2040
- NXP LPC
- Nordic nRF

The objective is to generate **production-grade firmware architecture** rather than demonstration code.

Generated code should prioritize:

- maintainability
- portability
- testability
- determinism
- real-time behavior

over rapid prototyping.

## Intended Audience

Target users include:

- embedded firmware engineers
- robotics developers
- flight-control engineers
- industrial automation developers
- RTOS application developers

Assume readers have basic C knowledge and MCU experience.

## Preferred Technology Stack

**Programming Language:** C99

**RTOS:** FreeRTOS, CMSIS-RTOS2

**Build Systems:** CMake, Make

**MCU Platforms:** STM32, GD32, ESP32

**Toolchains:** arm-none-eabi-gcc, clang

**IDE:** VSCode, CLion

## Architecture Philosophy

The preferred architecture is:

```text
APP → MODULE → INTERFACE → BSP → PLATFORM
```

Business logic must remain independent from STM32 HAL, MCU registers, and RTOS internals. Hardware access should be abstracted through stable interface contracts. Dependencies always point downward; circular dependencies are forbidden.

## Real-Time Design Philosophy

The firmware is expected to run in real-time environments. Priorities:

1. Deterministic execution
2. Low latency
3. Predictable scheduling
4. ISR minimization
5. Event-driven design

Control loops must run independently from communication timing.

## Driver Philosophy

Drivers should be reusable, hardware-independent, mockable, and testable. Device drivers depend on bus interfaces rather than MCU HAL APIs. Business logic must not access drivers directly.

## Documentation Philosophy

Documentation is considered part of the deliverable. Generated modules should include purpose, responsibilities, dependencies, and public APIs. Major architecture decisions should be documented.

## Agent Expectations

When generating firmware:

- prefer clarity over cleverness
- prefer maintainability over brevity
- prefer explicit interfaces over hidden coupling
- avoid shortcuts that violate architecture rules
- explain architectural decisions when introducing new modules

Do not generate code that bypasses defined abstractions merely for convenience.
