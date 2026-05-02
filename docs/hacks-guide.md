# Hack 功能详解

本文档详细描述 Timelapse 中所有已实现和待实现的 Hack 功能，包括实现原理、代码位置和开关逻辑。

---

## 已实现的 Hack 功能

### 1. 上帝模式 (Godmode)

#### 1.1 全能上帝模式 (Full Godmode)
- **代码洞穴**：`StatHook`（位于 `Hooks.h`）
- **Hack 地址**：`0x009581D5`
- **实现方式**：直接修改函数跳转指令，跳过伤害判定逻辑
- **开关逻辑**：
  - **开启**：`Jump(fullGodmodeAddr, StatHook, NOPS_FULLGODMODE)` — 在目标地址写入 JMP 到 StatHook 代码洞穴
  - **关闭**：`WriteMemory(fullGodmodeAddr, NOPS_FULLGODMODE, ...)` — 用 NOP 填充，跳过伤害判定

```cpp
// 开启
Void ToggleFullGodMode(bool enable) {
    if (enable)
        Jump(fullGodmodeAddr, StatHook, NOPS_FULLGODMODE);
    else
        WriteMemory(fullGodmodeAddr, NOPS_FULLGODMODE, ...);
}
```

#### 1.2 闪避上帝模式 (Miss Godmode)
- **代码洞穴**：`MissGodmodeHook`（位于 `Hooks.h`）
- **拦截函数**：`CUserLocal::SetDamaged`（`0x009582E9`）
- **实现方式**：维护一个计数器 `missCount`，当计数器 > 0 时将伤害设为 0（闪避），计数器归零时让伤害通过并重置
- **关键变量**：
  - `missCount`：初始值由用户设置，每受一次攻击递减
  - `Assembly::curHP`：由 `StatHook` 捕获的当前 HP

```cpp
// MissGodmodeHook 代码洞穴逻辑
CodeCave(MissGodmodeHook) {
    _asm {
        // ... 保存寄存器等 ...
        cmp Assembly::missCount, 0
        jle letDamageThrough
        dec Assembly::missCount
        mov Assembly::curHP, 0    // 将本次伤害设为 0
    letDamageThrough:
        // ... 恢复原始代码 ...
    }
}
EndCodeCave
```

---

### 2. 无呼吸 (NoBreath)
- **Hack 地址**：`0x00452316`
- **实现方式**：NOP 掉呼吸动画延迟，允许玩家在攻击后立即执行其他动作（如换频道）
- **开关逻辑**：
  - **开启**：`WriteMemory(noBreathAddr, NOPS_NOBREATH, ...)`
  - **关闭**：恢复原始字节

---

### 3. 无限攻击 (Unlimited Attack)
- **Hack 地址**：`0x009586E0`
- **实现方式**：修改攻击计数器重置逻辑，使攻击计数始终不超过 99（`attackCount < 100`）
- **注意**：需配合 `PointerFuncs::getAttackCount()` 来监控当前攻击计数

---

### 4. Tubi（加速物品拾取）
- **Hack 地址**：`0x00485C01`
- **实现方式**：NOP 掉物品拾取间的延迟检查
- **效果**：允许快速拾取多个物品，无需等待

---

### 5. 瞬移拾取

#### 5.1 InstantDrop
- **实现方式**：修改物品掉落的初始速度，使物品直接落地而不在空中漂浮

#### 5.2 InstantLoot
- **实现方式**：修改物品拾取动画延迟，瞬间完成拾取

---

### 6. 传送类 Hack

#### 6.1 鼠标飞行 (Mouse Fly)
- **代码洞穴**：`MouseFlyXHook` / `MouseFlyYHook`
- **拦截函数**：`CVecCtrl::raw__GetSnapshot`
- **实现方式**：当玩家的 `pID`（本地玩家 ID）与快照中的匹配时，将快照坐标替换为鼠标所指位置的坐标
- **条件**：需要知道鼠标所指位置的世界坐标（通常通过读取鼠标位置并转换）

#### 6.2 点击传送 (Click Teleport)
- **代码洞穴**：`ClickTeleportXHook` / `ClickTeleportYHook`
- **实现方式**：与鼠标飞行类似，但仅在 `MouseAnimation == 0x0C`（点击状态）时触发

#### 6.3 鼠标移动传送 (Mouse Teleport)
- **代码洞穴**：`MouseTeleportXHook` / `MouseTeleportYHook`
- **实现方式**：与鼠标飞行类似，但仅在 `MouseAnimation == 0x00`（非动画状态）时触发

---

### 7. 吸附类 Hack (Vacs)

#### 7.1 全图攻击 (FMA - Full Map Attack)
- **实现方式**：修改攻击范围判定，使攻击覆盖整个地图
- **稳定性问题**：在大型地图或飞行/跳跃怪物地图上不稳定

#### 7.2 DupeEx 吸附
- **代码洞穴**：`DupeExHook`
- **实现方式**：追踪玩家的立足点（foothold），将所有怪物传送到玩家所在位置
- **关键数据**：需要读取 `SpawnControlData` 中的出生点坐标

#### 7.3 怪物冻结 (Mob Freeze)
- **代码洞穴**：`MobFreezeHook`
- **拦截函数**：`CVecCtrlMob::WorkUpdateActive`（`0x009BCA92`）
- **实现方式**：将怪物的移动状态字节设为 `0x06`，使其冻结不动

#### 7.4 怪物自动仇恨 (Mob Auto Aggro)
- **代码洞穴**：`MobAutoAggroHook`
- **拦截函数**：同上（`0x009BCAF7`）
- **实现方式**：在 `WorkUpdateActive` 执行后，将怪物的仇恨值设为本地玩家的 ID

---

### 8. 出生点控制 (Spawn Control)
- **代码洞穴**：`SpawnPointHook`
- **拦截函数**：`CVecCtrl::SetActive`（`0x009B12A8`）
- **实现方式**：根据当前地图 ID 查找 `spawnControl` 列表中的出生点数据，修改怪物的生成坐标
- **数据结构**：
```cpp
struct SpawnControlData {
    UINT mapID;      // 地图 ID
    INT spawnX;      // 出生 X 坐标
    INT spawnY;      // 出生 Y 坐标
};
```

---

### 9. 过滤器

#### 9.1 物品过滤 (Item Filter)
- **代码洞穴**：`ItemFilterHook`
- **拦截函数**：`CDropPool::OnDropEnterField`（`0x005059CC`）
- **实现方式**：当物品进入地图时，根据 `itemList` 白名单/黑名单决定是否过滤该物品
- **注意**：并非所有物品 ID 都是游戏内物品，部分是道具/任务物品

#### 9.2 怪物过滤 (Mob Filter)
- **代码洞穴**：`MobFilter1Hook` / `MobFilter2Hook`
- **拦截函数**：`CMobPool::SetLocalMob` / `OnMobEnterField`
- **实现方式**：根据 `mobList` 决定是否过滤特定类型的怪物

---

### 10. 指针禁用 (Disable Pointers)
- **实现方式**：勾选时移除所有代码洞穴钩子，恢复原始指令，指针显示函数返回默认值
- **代码位置**：`MainForm.h` 中的 `cbDisablePointers_CheckedChanged`

---

## Hack 开关机制总结

所有 Hack 的开关逻辑遵循统一模式：

1. **启用**：
   - **代码洞穴型**：使用 `Jump()` 写入 JMP 指令到代码洞穴
   - **字节补丁型**：使用 `WriteMemory()` 写入修改后的字节

2. **禁用**：
   - **代码洞穴型**：使用 `WriteMemory()` 写入原始指令字节
   - **字节补丁型**：使用 `WriteMemory()` 写入原始字节

3. **全局禁用**（`cbDisablePointers`）：
   - 移除所有已安装的代码洞穴钩子
   - 所有 Hack 地址恢复原始字节

---

## 待实现的 Hack 功能

### 11. BYOR（Bring Your Own Rope）

- **Hack 地址**：`bringYourOwnRopeAddr`（`0x00A45B03`）— 代码洞穴型
- **实现方式**：钩取绳索/梯子相关函数，将玩家传送到自定义的绳索位置。当玩家按下上键时，将角色坐标修改为预设的绳索坐标，使玩家可以直接"爬"到指定绳索上
- **关键数据**：需要读取玩家当前坐标（`OFS_CharX` / `OFS_CharY`）和鼠标坐标（`OFS_MouseX` / `OFS_MouseY`）来确定目标绳索位置
- **开关逻辑**：
  - **开启**：`Jump(bringYourOwnRopeAddr, BYORHook, NOPS_BYOR)` — 在目标地址写入 JMP 到 BYORHook 代码洞穴
  - **关闭**：`WriteMemory(bringYourOwnRopeAddr, NOPS_BYOR, ...)` — 恢复原始指令字节

```cpp
// BYORHook 代码洞穴逻辑（待实现）
CodeCave(BYORHook) {
    _asm {
        // ... 保存寄存器 ...
        // 读取鼠标位置对应的绳索坐标
        // 将玩家坐标设置为绳索位置
        // ... 恢复原始代码 ...
    }
}
EndCodeCave
```

- **实现注意事项**：需要在 `Hooks.h` 中添加 `BYORHook` 代码洞穴，在 `Assembly` 命名空间中添加绳索坐标变量，在 `MainForm` 中添加复选框和坐标输入控件

---

### 12. ItsRainingMobs（怪物雨）

- **Hack 地址**：`itsRainingMobsAddr`（`0x009B1E8E`）— 字节补丁型
- **实现方式**：修改怪物生成逻辑中的条件分支。原始字节为 `0xF1`，将其修改为 `0xF2`，改变怪物的生成行为使其从空中落下
- **开关逻辑**：
  - **开启**：`WriteMemory(itsRainingMobsAddr, 1, 0xF2)` — 将 F1 改为 F2
  - **关闭**：`WriteMemory(itsRainingMobsAddr, 1, 0xF1)` — 恢复原始字节

```cpp
// 开关逻辑
Void ToggleItsRainingMobs(bool enable) {
    if (enable)
        WriteMemory(itsRainingMobsAddr, 1, 0xF2); // F1 -> F2
    else
        WriteMemory(itsRainingMobsAddr, 1, 0xF1); // 恢复原始
}
```

- **实现注意事项**：地址注释中提到"bugged disassembly??"，说明该地址的反汇编分析可能不准确，实际修改效果需在游戏中验证。此 Hack 难度较高，可能需要额外的代码洞穴来确保稳定性

---

### 13. MSCRC Bypass（绕过 CRC 检测）

- **Hack 地址**：
  - `MSCRCBypassAddr1`（`0x004A27E7`）— 代码洞穴型
  - `MSCRCBypassAddr2`（`0x004A27EC`）— 代码洞穴型
- **实现方式**：MapleStory 使用 CRC（循环冗余校验）来检测游戏代码段是否被修改。Bypass 的原理是拦截 CRC 校验函数，在校验时返回原始代码内容而非被修改后的内容。通常使用"代码洞穴 + 原始代码副本"的方式实现：
  1. 在进程内存中分配一块新区域，存放原始代码段的副本
  2. 拦截 CRC 校验函数，使其读取原始副本而非被修改的代码段
  3. CRC 校验函数校验原始副本，结果与预期一致，从而绕过检测
- **开关逻辑**：
  - **开启**：在 `MSCRCBypassAddr1` 和 `MSCRCBypassAddr2` 处写入 JMP 到 CRC Bypass 代码洞穴
  - **关闭**：恢复两个地址的原始指令字节

```cpp
// MSCRCBypassHook 代码洞穴逻辑（待实现）
CodeCave(MSCRCBypassHook) {
    _asm {
        // ... 保存寄存器 ...
        // 将 CRC 校验的目标地址重定向到原始代码副本
        // 使校验函数读取未修改的原始代码
        // ... 恢复原始代码并返回 ...
    }
}
EndCodeCave
```

- **实现注意事项**：
  - 这是高优先级功能，因为其他 Hack 修改代码段后会被 CRC 检测到
  - 需要在 DLL 注入后尽早启用，建议在 `DllMain` 的 `DLL_PROCESS_ATTACH` 中执行
  - 两个地址（`0x004A27E7` 和 `0x004A27EC`）相距仅 5 字节，可能需要合并处理
  - 必须在 `Hooks.h` 的 `#pragma unmanaged` 区域中实现代码洞穴

---

### 14. MP/HP 再生 Hack

- **Hack 地址**：`mpRegenTickTimeAddr`（`0x00A031F5`）— 字节补丁型
- **实现方式**：修改 MP/HP 自然恢复的时间间隔检查。原始代码为 `cmp ebx, 0x00002710`（比较是否达到 10000ms 的恢复间隔），将其修改为 `cmp ebx, [BYTE_VALUE]`，使用一个自定义的较小值，从而加速 HP/MP 的自然恢复
- **开关逻辑**：
  - **开启**：`WriteMemory(mpRegenTickTimeAddr, ...)` — 将比较值替换为用户自定义的较小间隔（如 100ms）
  - **关闭**：`WriteMemory(mpRegenTickTimeAddr, ...)` — 恢复原始的 `0x00002710`（10000ms）

```cpp
// 开关逻辑
Void ToggleMPRegenHack(bool enable) {
    if (enable) {
        // 将 cmp ebx, 0x00002710 改为 cmp ebx, [自定义值]
        // 例如 100ms 间隔：0x00000064
        WriteMemory(mpRegenTickTimeAddr, 4, 0x64, 0x00, 0x00, 0x00);
    }
    else {
        // 恢复原始：0x00002710 (10000ms)
        WriteMemory(mpRegenTickTimeAddr, 4, 0x10, 0x27, 0x00, 0x00);
    }
}
```

- **实现注意事项**：
  - 目前只有 MP 再生地址，HP 再生可能使用相同的机制或需要查找独立地址
  - 需要添加 UI 控件让用户自定义再生间隔时间
  - 可配合已有的 `PointerFuncs::getCharHP()` / `getCharMP()` 来监控效果

---

### 15. AttackUnrandomizer（攻击伤害去随机化）

- **Hack 地址**：`attackUnrandommizerAddr`（`0x0076609C`）— 代码洞穴型
- **实现方式**：钩取攻击伤害计算中的随机数生成函数，将随机伤害值替换为固定值（通常为最大伤害），使每次攻击都造成相同（最大）伤害
- **开关逻辑**：
  - **开启**：`Jump(attackUnrandommizerAddr, AttackUnrandomizerHook, NOPS_ATTACKUNRANDOM)` — 在目标地址写入 JMP 到代码洞穴
  - **关闭**：`WriteMemory(attackUnrandommizerAddr, NOPS_ATTACKUNRANDOM, ...)` — 恢复原始指令字节

```cpp
// AttackUnrandomizerHook 代码洞穴逻辑（待实现）
CodeCave(AttackUnrandomizerHook) {
    _asm {
        // ... 保存寄存器 ...
        // 将随机数种子替换为固定值
        // 或将伤害计算结果覆盖为最大值
        // ... 恢复原始代码 ...
    }
}
EndCodeCave
```

- **实现注意事项**：
  - 注意地址变量名拼写为 `attackUnrandommizerAddr`（双 m），代码中需保持一致
  - 去随机化效果是客户端可见的，实际伤害仍由服务器计算
  - 可扩展为支持自定义伤害值而不仅是最大伤害

---

### 16. 角色速度编辑

- **相关地址**：
  - `speedWalkAddr`（`0x009B268D`）— 字节补丁型（`je` → `6x nop`）
  - `walkingFrictionAddr`（`0x009B4365`）— 字节补丁型（`je` → `jne`）
  - `gravity`（`0x00AF0DE0`）— double 类型（重力值）
  - `slideRightAddr`（`0x009B2C0A`）— 字节补丁型（`jna` → `jne`）
- **实现方式**：通过多个地址的组合修改，实现角色行走速度、跳跃高度、摩擦力等物理参数的调整：
  - **加速行走**：在 `speedWalkAddr` 处 NOP 掉速度限制条件跳转（`je` → `6x nop`），允许角色在任何状态下以最高速度行走
  - **无摩擦行走**：在 `walkingFrictionAddr` 处修改条件跳转（`je` → `jne`），移除行走时的摩擦力减速效果
  - **重力修改**：直接修改 `gravity` 地址处的 double 值，降低重力可实现更高跳跃，增大重力则下落更快
  - **右滑行**：在 `slideRightAddr` 处修改条件跳转（`jna` → `jne`），使角色可以沿墙壁向右滑行
- **开关逻辑**：

```cpp
// 加速行走
Void ToggleSpeedWalk(bool enable) {
    if (enable)
        WriteMemory(speedWalkAddr, 6, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90); // 6x nop
    else
        WriteMemory(speedWalkAddr, 6, 0x0F, 0x84, 0x3A, 0x01, 0x00, 0x00); // je 原始字节
}

// 重力修改
Void SetGravity(double value) {
    *(double*)gravity = value; // 直接写入 double 值
}
```

- **实现注意事项**：
  - 重力值为 double 类型（8 字节），原始值约为 `0.0` 或默认重力常数，需要先读取原始值再修改
  - 所有物理参数修改均为客户端效果，不影响服务器判定
  - 建议在 UI 中添加滑块控件（TrackBar）来直观调整各参数

---

### 17. Auto Assign AP（自动分配能力点）

- **实现方式**：基于现有的 `PointerFuncs` 读写函数和 ZtlSecureFuse 编码，在每级升级时自动将能力点（AP）分配到指定属性。已存在 UI 框架（`cbAP` 复选框、`tbAPLevel`/`tbAPSTR`/`tbAPDEX`/`tbAPINT`/`tbAPLUK`/`tbAPHP`/`tbAPMP` 等输入框），但功能尚未启用
- **关键数据**：
  - 角色等级：`PointerFuncs::getCharLevel()`
  - 能力点值：`PointerFuncs::setCharSTR()` / `setCharDEX()` / `setCharINT()` / `setCharLUK()`
  - HP/MP 分配：`PointerFuncs::setCharHP()` / `setCharMP()`
- **实现逻辑**：

```cpp
// 在 GUITimer_Tick 中检查（伪代码）
Void AutoAssignAP() {
    if (!cbAP->Checked) return;

    UINT8 currentLevel = PointerFuncs::getCharLevel();
    UINT8 targetLevel = Convert::ToByte(tbAPLevel->Text);
    if (currentLevel > targetLevel) return;

    // 获取用户设定的各属性分配点数
    SHORT strAdd = Convert::ToInt16(tbAPSTR->Text);
    SHORT dexAdd = Convert::ToInt16(tbAPDEX->Text);
    SHORT intAdd = Convert::ToInt16(tbAPINT->Text);
    SHORT lukAdd = Convert::ToInt16(tbAPLUK->Text);

    // 分配能力点，直到 AP < 5
    SHORT currentSTR = PointerFuncs::getCharSTR();
    PointerFuncs::setCharSTR(currentSTR + strAdd);
    // ... 同理分配 DEX, INT, LUK, HP, MP ...
}
```

- **实现注意事项**：
  - 当前 `cbAP` 复选框的 `Enabled` 属性为 `false`，功能实现后需改为 `true`
  - **必须确保封包 Auto AP 未启用**，否则两者会冲突
  - HP/MP 的 AP 分配与 STR/DEX/INT/LUK 不同，需要发送特定的升级封包才能正确增加
  - 建议使用 `writeShortValueZtlSecureFuse` 来写入属性值，确保与游戏的 ZtlSecureFuse 编码兼容

---

## 待实现功能总览

| # | 功能 | 优先级 | 实现难度 | Hack 类型 | 地址 |
|---|------|--------|----------|-----------|------|
| 11 | BYOR (Bring Your Own Rope) | 高 | 中 | 代码洞穴 | `0x00A45B03` |
| 12 | ItsRainingMobs | 中 | 高 | 字节补丁 | `0x009B1E8E` |
| 13 | MSCRC Bypass | 高 | 高 | 代码洞穴 | `0x004A27E7` / `0x004A27EC` |
| 14 | MP/HP 再生 Hack | 中 | 低 | 字节补丁 | `0x00A031F5` |
| 15 | AttackUnrandomizer | 低 | 中 | 代码洞穴 | `0x0076609C` |
| 16 | 角色速度编辑 | 中 | 低 | 字节补丁 + 直接写入 | `0x009B268D` / `0x009B4365` / `0x00AF0DE0` |
| 17 | Auto Assign AP | 低 | 低 | 指针写入 | `CharacterStatBase` + 偏移 |
