# Embedded Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A collection of **AI assistant skills for embedded firmware engineering**, designed for STM32, GD32, ESP32, RP2040, NXP, Nordic, and similar MCU platforms.

These skills guide AI coding assistants (Claude Code, GitHub Copilot, etc.) to produce production-grade embedded firmware for RTOS-based systems, drone flight controllers, robotics, and industrial control applications.

## Skills

| Skill | Description |
| --- | --- |
| [embedded-architecture](skills/embedded-architecture/SKILL.md) | Firmware architecture — project structure, layer separation, dependency rules, platform isolation |
| [embedded-cstyle](skills/embedded-cstyle/SKILL.md) | C coding standards — naming, types, functions, variables, macros, headers, memory, error handling |
| [embedded-driver-design](skills/embedded-driver-design/SKILL.md) | Driver layered design — bus abstraction, device context, register encapsulation, mock testing |
| [embedded-rtos-design](skills/embedded-rtos-design/SKILL.md) | RTOS system architecture — task decomposition, scheduling, priorities, inter-task communication, watchdog |
| [embedded-isr-design](skills/embedded-isr-design/SKILL.md) | Interrupt design — Fast In Fast Out, event capture, deferred processing, FromISR API, priority hierarchy |
| [embedded-documentation](skills/embedded-documentation/SKILL.md) | Documentation generation — 5-level documentation framework covering project/module/interface/RTOS/control docs |

## Architecture

The skills enforce a consistent layered firmware architecture:

```text
APP → MODULE → INTERFACE → BSP → PLATFORM
```

Dependencies always point downward. Business logic must never depend on MCU vendor libraries.

## Quick Start

```bash
# Copy all skills to your AI assistant
cp -r skills/* /path/to/agent/.skills/
```

See [CONTEXT.md](CONTEXT.md) for project philosophy and design principles.

## License

MIT — see [LICENSE](LICENSE).

## Author

- **rovinax** - [GitHub](https://github.com/rovinax)
