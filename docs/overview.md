# 项目概述

## 项目名称

**Timelapse** — MapleStory v83 训练器 (Trainer)

## 项目简介

Timelapse 是一个针对 MapleStory v83 客户端的 **C++/CLI DLL 训练器**，通过注入到 MapleStory.exe 进程中运行。它提供了一个 WinForms GUI，用于内存修改、代码洞穴钩子、封包发送以及自动化 Bot 功能。

## 核心定位

| 特性 | 描述 |
|------|------|
| 目标进程 | MapleStory v83（32位 x86 Windows 进程） |
| 输出类型 | DLL（动态链接库） |
| 注入方式 | 外部注入器（如 Extreme Injector v3.7.2，推荐使用 Stealth Inject 选项） |
| 运行方式 | 注入后自动创建 WinForms 窗口作为训练器界面 |

## 功能概览

训练器提供以下核心功能模块：

- **Hacks（黑客功能）**：上帝模式（Godmode）、无呼吸（NoBreath）、无限攻击（UnlimitedAttack）、瞬移拾取（InstantDrop/Loot）等
- **Vacs（吸附功能）**：全图攻击（FMA）、DupeEx、怪物冻结（MobFreeze）、怪物自动仇恨（MobAutoAggro）等
- **Bots（自动化）**：自动攻击、自动拾取、自动吃药（HP/MP Pot）、自动换频道（AutoCC/CS）
- **Filters（过滤器）**：物品过滤（白名单/黑名单）、怪物过滤
- **Packets（封包）**：封包发送、封包接收、封包记录
- **Map Rusher（地图传送）**：快速传送至指定地图
- **Macro System（宏系统）**：优先级队列驱动的按键宏，支持自动吃药和自动攻击

## 技术栈

- **语言**：C++/CLI（`/clr` 启用），混合原生 C++ 与托管代码
- **UI 框架**：Windows Forms（WinForms）
- **钩子库**：Microsoft Detours 3.0（包含在仓库中）
- **构建工具**：Visual Studio 2022（Platform Toolset v143）
- **C++ 标准**：C++20（`stdcpp20`，C++/CLI 支持的最高版本）
- **Windows SDK**：10.0.22621.0

## 许可证

请参阅仓库根目录的 [LICENSE](../LICENSE) 文件。
