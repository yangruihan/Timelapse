# 已知问题与 TODO

本文档汇总了 Timelapse 项目中所有已知的 Bug、待实现功能和技术债务。

---

## 🐛 已知 Bug

### 严重 Bug

| # | Bug | 影响 | 文件 | 详细说明 |
|---|-----|------|------|----------|
| 1 | 热键队列无响应 | 高 | `Macro.h` | 攻击/拾取宏高频触发时会阻塞所有其他按键输入。`SendKeyMacroToQueue` 和 `MacroQueueWorker` 存在竞争条件，优先级队列的 pop 操作阻塞了 UI 线程 |
| 2 | Auto CS 功能调用失败 | 高 | `MainForm.h` | 当 `MigrateToCashShop` 触发时弹出蓝色异常框，Auto CC/CS 无法按预期工作 |
| 3 | ClickTeleport / MouseTeleport / TeleportLoop 延迟为空 | 高 | `MainForm.h` | 对应的延迟文本框为空时触发空引用异常 |
| 4 | FMA 稳定性问题 | 中 | `Hooks.h` | 在大型地图或飞行/跳跃怪物地图上不稳定 |
| 5 | Kami 稳定性问题 | 中 | `Hooks.h` | 同上，定时 Auto CC 可提升稳定性 |

### 一般 Bug

| # | Bug | 影响 | 文件 | 详细说明 |
|---|-----|------|------|----------|
| 6 | AutoCC/CS 空值修复 | 中 | `MainForm.h` | `tbCCCSTime`、`tbCCCSPeople` 等字段可能为空，需添加错误提示而非直接崩溃 |
| 7 | 窗体关闭时布尔值重置 | 低 | `MainForm.h` | 关闭窗体时某些布尔值可能无法正常重置（高延迟线程退出问题） |
| 8 | 频道从 0 开始 | 低 | `MainForm.h` | 代码中频道 0 不存在（应为 1-19），需确保函数反映这一点 |
| 9 | AutoCC/CS SendPacket | 低 | `Packet.cpp` | 当前直接调用 `Send`，应改用 packet write 方法 |
| 10 | ListView 美化 | 低 | `MainForm.h` | 需适配垂直滚动条，统一增删查代码风格 |
| 11 | 文本框命名 | 低 | `MainForm.h` | 如 `tbHPMob` → `tbHPMobCount`，"interval" → "delay" 等 |

### 代码 Bug

| # | Bug | 影响 | 文件 | 详细说明 |
|---|-----|------|------|----------|
| 12 | `HelperFunctions.h` 引用不存在文件 | 低 | `HelperFunctions.h` | 引用了不存在的 `"Functions.h"`，且与 `MapleFunctions.h` 代码重复 |
| 13 | `Assembly.h` 冗余文件 | 低 | `Assembly.h` | 几乎是 `Hooks.h` 的副本（旧版本），不应使用 |

---

## ✅ 已实现功能

### Hacks 标签
- [x] Full Godmode（全能上帝模式）
- [x] Miss Godmode（闪避上帝模式）
- [x] NoBreath（无呼吸）
- [x] Unlimited Attack（无限攻击）
- [x] Tubi（加速拾取）
- [x] InstantDrop（瞬移掉落）
- [x] InstantLoot（瞬移拾取）
- [x] Mouse Fly（鼠标飞行）
- [x] Click Teleport（点击传送）
- [x] Mouse Teleport（鼠标移动传送）

### Vacs 标签
- [x] FMA（全图攻击）
- [x] Mob Freeze（怪物冻结）
- [x] Mob Auto Aggro（怪物自动仇恨）
- [x] Spawn Control（出生点控制）

### Filters 标签
- [x] Item Filter（物品过滤）
- [x] Mob Filter（怪物过滤）

### Packets 标签
- [x] SendPacket（发送封包）
- [x] RecvPacket（接收封包）
- [x] SendPacketLogHook（封包日志记录）

### Bots 标签
- [x] Auto Attack（自动攻击）
- [x] Auto Loot（自动拾取）
- [x] Auto HP Pot（自动 HP 药水）
- [x] Auto MP Pot（自动 MP 药水）
- [x] Auto CC/CS（自动换频道/换商店）
- [x] Macro System（宏系统：优先级队列 + 定时发送）

### Main 标签
- [x] Pointer Display（指针显示：HP/MP/EXP/Level/Job/Map 等）
- [x] Teleport（传送功能）
- [x] Settings Save/Load（设置保存/加载）

---

## ❌ 待实现功能（TODO）

### 工具栏 (ToolBar)

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T1 | 保存/加载机制 | 高 | 低 | 添加 itemFilter 和 mapRusher 预定义路线的保存 |
| T2 | 全选启用/禁用 | 中 | 低 | 批量启用或禁用所有 hack |
| T3 | 启动 MS 客户端 | 中 | 中 | 从训练器内直接启动 MapleStory |
| T4 | 注入例程 | 高 | 中 | 部署训练器的注入流程 |
| T5 | 调整大小并嵌入 MS 客户端 | 中 | 高 | 将 MapleStory 窗口嵌入训练器窗体 |
| T6 | 隐藏/显示 MS 窗口 | 低 | 低 | 旧训练器中已实现，需移植 |

### 主标签 (Main Tab)

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T7 | AutoLogin 例程 | 高 | 高 | 通过封包或钩取客户端函数实现 |
| T8 | Pin/Pic 绕过 | 高 | 高 | - |
| T9 | 跳过 Logo | 低 | 低 | CT 文件中已找到，需编码实现 |
| T10 | 禁用指针显示 | 低 | 低 | - |

### Bots 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T11 | 自动转向 (Auto Turn) | 高 | 低 | 需新增 |
| T12 | 控制台输出 | 低 | 低 | 各种日志流的输出控制台 |
| T13 | CC/CS 后自动面朝左/右选项 | 中 | 低 | 需新增 |
| T14 | 线程化 Auto CC/CS | 高 | 中 | 当前使用 Timer 导致训练器卡顿，应改为线程 |
| T15 | 频道选择 | 中 | 低 | 添加选择随机 CC 频道和禁止进入频道的选项 |
| T16 | NoBreath 与 Auto CC 关系 | 中 | 研究 | 无呼吸状态下发送 CC 封包是否有效？ |

### 高级 Bot

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T17 | 平地 Bot (Flat Bot) | 中 | 中 | 需新增 |
| T18 | 爬梯/绳子 | 中 | 高 | `CVecCtrl::GetLadderOrRope` / `IsOnLadder` / `IsOnRope` |
| T19 | 查找最近/最密集怪物 | 中 | 中 | 螺旋算法（从内到外的环形搜索） |
| T20 | 基于等级的 Map Rush | 中 | 中 | - |
| T21 | 左/右行走 | 中 | 低 | - |
| T22 | 鼠标模拟自动点击 | 中 | 中 | - |

### Hacks 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T23 | BYOR (Bring Your Own Rope) | 中 | 低 | - |
| T24 | ItsRainingMobs | 低 | 高 | - |
| T25 | MSCRC Bypass | 高 | 高 | - |
| T26 | MP/HP 再生 Hack | 中 | 低 | - |
| T27 | AttackUnrandomizer | 低 | 中 | - |
| T28 | 角色行走/跳跃速度编辑 | 中 | 低 | 双倍速度、伤害等物理参数修改 |
| T29 | Auto Assign AP | 低 | 低 | 需确保封包 Auto AP 未启用 |

### Vacs 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T30 | DupeEx/MMC/Wall Vac | 中 | 中 | 旧训练器中已编码，需移植 |
| T31 | Vami, Portal Kami, dEMi | 低 | 高 | - |
| T32 | CSEAX Vac, Mouse CSEAX | 中 | 中 | - |
| T33 | Force Vac, PVac, Vac 左/右 | 低 | 中 | - |
| T34 | Pet Item Vac | 低 | 低 | - |

### Filters 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T35 | 非游戏物品过滤 | 中 | 低 | 并非所有物品都是游戏内物品，可通过字符串搜索发现 |

### Packets 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T36 | 封锁所有封包 | 中 | 中 | - |
| T37 | 记录发送/接收封包 | 中 | 中 | - |
| T38 | 多封包发送/接收 | 低 | 低 | - |
| T39 | 查找/编码定义封包 | 低 | 中 | 包括 Auto AP、Auto-Sell、跨大陆封包 |

### Map Rusher 标签

| # | 功能 | 优先级 | 难度 | 说明 |
|---|------|--------|------|------|
| T40 | Map Rush 核心功能 | 高 | 高 | 主要的地图传送逻辑 |
| T41 | 跨大陆 Map Rush | 中 | 高 | - |
| T42 | Auto-Rushback | 中 | 中 | 自动返回 |
| T43 | 封包和传送双模式 | 中 | 高 | 支持基于封包和基于传送的两种 Map Rush |
| T44 | WZ XML 解析器重写 | 高 | 高 | 需用 C++ 重写，因很多地图缺少传送门数据（如 Kerning Square） |

---

## 🔧 整体改进

| # | 改进 | 优先级 | 说明 |
|---|------|--------|------|
| I1 | GUI 整体重构 | 高 | 当前界面需要全面改进 |
| I2 | 多实例注入 | 中 | 确保训练器可注入多个实例且互不干扰 |
| I3 | 内存管理重写 | 高 | 重写为类型泛型、操作更安全的内存管理器 |
| I4 | C 风格转换 → C++ 风格转换 | 低 | 使用 `static_cast`/`reinterpret_cast` 等 |
| I5 | 自动拾取检测背包满 | 中 | 检查背包是否已满，并添加自动出售垃圾和购买药水选项 |
| I6 | 操作日志/标签 | 中 | 使标签可见一段时间（或记录日志）显示训练器中所有操作 |
| I7 | 全面工具提示 (Tooltips) | 中 | 为所有控件添加说明 |
| I8 | 统计日志 | 低 | 记录拾取金币/击杀怪物/升级数量 |
| I9 | Auto CC/CS 倒计时 | 中 | 在训练器底部显示倒计时（替换无用的时钟标签） |

---

## 🏗️ 技术债务

| # | 项目 | 说明 |
|---|------|------|
| D1 | `HelperFunctions.h` | 引用不存在的 `"Functions.h"`，且与 `MapleFunctions.h` 代码重复 → 应删除 |
| D2 | `Assembly.h` | 近似 `Hooks.h` 的旧版本副本 → 应删除 |
| D3 | 动态出生点数据 | 需添加到 CodeCave，检查 `Structs.h`/`SpawnControlStruct` 是否必要 |
| D4 | 地图加载检测 | 需要指针判断地图是否加载完成（防止重复点击导致 Map Rush/Auto CC 多次执行） |
| D5 | HP/MP/EXP/Map Name 指针 | 需找更好的函数做 Code Cave |
| D6 | 指针函数返回值 | 所有 `ReadPointer`/`ReadMultiPointer` 应返回整数值，由调用者转换 |
| D7 | 禁用文本框联动 | 勾选某项时禁用相关文本框，移除所有 `textChanged` 事件 |
| D8 | 新数字文本框按键事件 | 确认 `MainForm.h` 中的 `keypress` 事件已定义 |
| D9 | `char` vs `unsigned char` | 检查字符数组用文本，无符号字符用于内存 |
| D10 | 频道从 1 开始 | 代码中频道 0 不存在，需确保函数反映这一点 |
