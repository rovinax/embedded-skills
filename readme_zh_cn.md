# Embedded Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## 项目简介

**Embedded Skills** 是一个面向嵌入式固件开发的开源 AI 技能仓库，旨在帮助开发者在常见嵌入式开发场景中编写高可用、高移植性、更加规范化的嵌入式软件。

本仓库提供了一套专为 AI 编程助手（如 Claude Code、GitHub Copilot 等）设计的技能（Skills）指令集，覆盖嵌入式开发的多个核心领域——从软件架构设计、编码风格规范，到驱动开发、中断服务程序设计和 RTOS 任务编排。每个技能都封装了经过工程验证的最佳实践和评审规范，通过 AI 辅助的方式将其应用到日常开发流程中。

无论你是嵌入式领域的新手还是资深工程师，Embedded Skills 都能帮助你在项目起步阶段快速建立规范的工程基线，并在开发过程中持续提供架构指导和代码评审支持。

## 特性

- **领域专精** — 覆盖嵌入式固件开发的六大核心领域：架构、编码风格、文档、驱动、中断和 RTOS
- **即插即用** — 将技能文件放置到 AI 助手的工作目录即可生效，无需额外配置
- **工程验证** — 每条规则和指南均源自真实嵌入式项目的经验沉淀
- **AI 原生** — 专为 AI 编程助手设计，指令清晰、边界明确、可执行性强
- **开源免费** — MIT 协议，可自由使用、修改和分发

## 技能列表

本仓库包含以下技能，按嵌入式开发的核心关注点组织：

| 名称 | 描述 | 文档 |
| --- | --- | --- |
| `embedded-architecture` | **嵌入式软件架构设计** — 项目目录结构规划、模块分层设计、接口定义规范，构建可移植、可维护的固件架构 | [SKILL.md](skills/embedded-architecture/SKILL.md) |
| `embedded-cstyle` | **嵌入式 C 编码风格与安全审查** — 编码规范、静态分析规则、防御式编程实践，提升代码的安全性与可读性 | [SKILL.md](skills/embedded-cstyle/SKILL.md) |
| `embedded-documentation` | **固件文档与 API 文档生成** — 代码注释规范、API 文档自动化、设计文档模板，降低项目维护成本 | [SKILL.md](skills/embedded-documentation/SKILL.md) |
| `embedded-driver-design` | **外设驱动与硬件抽象设计** — 外设驱动框架设计、HAL 抽象层规划、寄存器操作封装，提高驱动代码的可移植性 | [SKILL.md](skills/embedded-driver-design/SKILL.md) |
| `embedded-isr-design` | **中断服务程序设计评审** — ISR 编写规范、中断优先级规划、临界区保护策略、中断延迟优化 | [SKILL.md](skills/embedded-isr-design/SKILL.md) |
| `embedded-rtos-design` | **RTOS 任务架构与调度评审** — 任务划分原则、优先级分配策略、同步与通信机制、堆栈使用分析 | [SKILL.md](skills/embedded-rtos-design/SKILL.md) |

> 每个技能均包含独立的使用说明和评审清单，详见对应目录下的 `SKILL.md` 文件。

## 安装与使用

### 前置条件

- 支持技能系统（Skills）的 AI 编程助手，如 Claude Code、GitHub Copilot CLI 等
- 已初始化的嵌入式项目目录

### 安装方式

将需要的技能目录复制到 AI 助手的工作目录中的技能文件夹：

```bash
# 复制全部技能
cp -r skills/* /path/to/agent/.skills/

# 或按需选择特定技能
cp -r skills/embedded-architecture /path/to/agent/.skills/
cp -r skills/embedded-cstyle /path/to/agent/.skills/
```

### 使用流程

1. 根据项目需求选择适用的技能（如新项目建议启用 `embedded-architecture`）
2. 安装技能到 AI 助手工作目录
3. 在对话中向 AI 助手提出嵌入式开发相关需求
4. AI 助手会根据技能指令自动应用工程规范

## 项目背景

嵌入式固件开发长期面临几个痛点：

- **规范缺失** — 小团队项目往往缺少明确的架构和编码规范，导致后期维护困难
- **经验分散** — 优秀的工程实践散落在各个项目和工程师头脑中，难以系统化复用
- **AI 辅助无序** — 使用 AI 编程助手时缺少领域引导，生成代码质量参差不齐

Embedded Skills 正是为解决这些问题而生。它将嵌入式领域的工程经验封装成 AI 可理解的技能指令，让 AI 助手成为真正懂嵌入式开发的编程伙伴。

## 贡献指南

欢迎贡献新的技能或改进已有技能。请遵循以下规则：

1. 每个技能独立存放在 `skills/` 目录下
2. 必须包含有效的 `SKILL.md` 文件
3. 必须在 `.claude-plugin/plugin.json` 中注册
4. 必须在本 README 的技能列表中列出

开发准则：

- 保持职责单一聚焦，避免技能间功能重叠
- 优先使用可复用的工程规则而非项目特定的逻辑
- 必要时包含实用示例
- 保持与现有技能结构和命名风格一致

## 许可证

本项目基于 [MIT 许可证](LICENSE) 开源。

## 作者

- **rovinax** - [GitHub](https://github.com/rovinax)

---

**Embedded Skills** — 让 AI 更懂嵌入式开发。
