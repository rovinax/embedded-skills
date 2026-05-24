# Embedded Firmware Skills Repository

This repository contains reusable skills for embedded firmware engineering.

Skills are stored directly under `skills/`:

```text
skills/
├── embedded-architecture
├── embedded-cstyle
├── embedded-documentation
├── embedded-driver-design
├── embedded-isr-design
└── embedded-rtos-design
```

## Project Files

- [CONTEXT.md](CONTEXT.md) — Project philosophy, technology stack, design principles
- [CLAUDE.md](CLAUDE.md) — Repository rules and development guidelines
- Each skill has its own `SKILL.md` with detailed rules and validation checklists

## Skill Responsibilities

- `embedded-architecture`
  - Firmware architecture — project structure, layer separation, dependency direction, platform isolation

- `embedded-cstyle`
  - C coding standards — naming, types, functions, variables, macros, headers, memory, error handling

- `embedded-documentation`
  - Documentation generation — 5-level doc framework: project, module, interface, RTOS/ISR, control

- `embedded-driver-design`
  - Driver layered design — bus abstraction, device context, register encapsulation, mock testing

- `embedded-isr-design`
  - Interrupt design — Fast In Fast Out, event capture, FromISR API, deferred processing

- `embedded-rtos-design`
  - RTOS system architecture — task decomposition, scheduling, priorities, inter-task communication, watchdog

## Repository Rules

Every skill must:

1. Reside directly under `skills/`
2. Contain a valid `SKILL.md`
3. Be registered in `.claude-plugin/plugin.json`
4. Be listed in this document

## Skill Development Guidelines

When creating or modifying skills:

- Keep responsibilities focused and single-purpose
- Avoid overlapping functionality between skills
- Prefer reusable engineering rules over project-specific logic
- Include practical examples when useful
- Maintain consistency with existing skill structure and naming

The goal is to build a maintainable, extensible, and professional embedded firmware skill library for AI-assisted development.
