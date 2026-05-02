# 核心模块详解

本文档详细描述 Timelapse 项目中各核心模块的功能和实现细节。

## 模块总览

| 文件 | 功能定位 | 代码量占比 |
|------|----------|-----------|
| `MainForm.h/cpp` | WinForms UI 定义及所有事件处理程序 | ~70% |
| `Addresses.h` | MapleStory v83 所有硬编码内存地址 | 核心数据 |
| `Hooks.h` | Microsoft Detours 钩子、汇编代码洞穴、`Assembly` 命名空间共享状态 | 核心逻辑 |
| `Memory.h` | 低级内存工具：指针读写、内存保护、ZtlSecureFuse | 基础设施 |
| `MapleFunctions.h` | 游戏辅助函数：传送、窗口操作、指针读取、状态判断 | 业务逻辑 |
| `Macro.h` | 宏系统：优先级队列、定时按键发送 | 自动化核心 |
| `Packet.h/cpp` | 封包发送/接收、封包构建器 | 网络交互 |
| `Keyboard.h/cpp` | 键盘模拟：基于 `SendInput` | 输入模拟 |
| `Mouse.h/cpp` | 鼠标模拟：基于 `SendInput` | 输入模拟 |
| `Structs.h` | 数据结构：封包结构、出生点数据、传送门数据、地图数据 | 数据模型 |
| `Settings.h/cpp` | XML 序列化保存/加载窗体控件状态 | 配置管理 |
| `Log.h/cpp` | 日志输出到文件和控制台 | 调试/审计 |

---

## 1. Addresses.h — 内存地址定义

### 功能
定义了 MapleStory v83 进程虚拟地址空间中的所有关键地址，包括：

### Hack 地址
用于直接修改游戏逻辑的内存地址。每个地址对应一种 Hack 功能的开关位置。

**示例**：
```cpp
ULONG fullGodmodeAddr = 0x009581D5;     // 全能上帝模式
ULONG missGodmodeAddr = 0x009582E9;      // 闪避上帝模式
ULONG noBreathAddr = 0x00452316;          // 无呼吸
ULONG tubiAddr = 0x00485C01;              // Tubi（加速物品拾取）
```

### 代码洞穴地址
用于钩取游戏函数的关键拦截点。代码洞穴通过 `Jump()` 函数写入 JMP 指令跳转到这些位置。

**示例**：
```cpp
ULONG statHookAddr = 0x008D8581;          // 拦截 CUIStatusBar::SetNumberValue
ULONG mapNameHookAddr = 0x005CFA48;       // 拦截 CItemInfo::GetMapString
ULONG itemVacAddr = 0x005047AA;           // 拦截 CDropPool::TryPickUpDrop
```

### 指针基址与偏移
多层指针解引用的静态基址及对应的偏移量，用于读取游戏运行时数据。

**示例**：
```cpp
// UI 信息基址
ULONG UIInfoBase = 0xBEC208;
ULONG OFS_HP = 0xD18;        // HP 偏移
ULONG OFS_MP = OFS_HP + 4;   // MP 偏移

// 角色状态基址
ULONG CharacterStatBase = 0xBF3CD8; // GW_CharacterStat
ULONG OFS_Level = 0x33;       // 等级偏移
ULONG OFS_JobID = 0x39;       // 职业ID偏移
ULONG OFS_Mesos = 0xA5;       // 金币偏移
```

---

## 2. Memory.h — 内存操作工具

### 功能
提供所有低级内存读写操作，是训练器的基础设施层。

### 核心函数

| 函数 | 用途 |
|------|------|
| `MakePageWritable(ulAddress, ulSize)` | 将目标内存页的属性修改为 `PAGE_EXECUTE_READWRITE`，以便写入代码洞穴 |
| `Jump(ulAddress, Function, Nops)` | 在目标地址写入 `JMP` 指令跳转到代码洞穴函数，并用 `NOP` 填充空隙 |
| `ReadPointer(ulBase, iOffset)` | 通过基址+偏移读取指针指向的 `ULONG` 值，带 SEH 异常保护 |
| `ReadPointerSigned(ulBase, iOffset)` | 同上，返回有符号 `LONG` |
| `ReadPointerDouble(ulBase, iOffset)` | 同上，返回 `double` |
| `ReadPointerString(ulBase, iOffset)` | 同上，返回 `LPCSTR` |
| `ReadMultiPointerSigned(ulBase, level, ...)` | 多层指针解引用，可变参数指定每层的偏移 |
| `ReadMultiPointerString(ulBase, level, ...)` | 同上，返回字符串 |
| `WriteMemory(ulAddress, ulAmount, ...)` | 向目标地址写入指定数量的字节 |
| `WritePointer(ulBase, iOffset, iValue)` | 通过基址+偏移写入一个整数值 |

### ZtlSecureFuse 读写
针对 Nexon ZtlSecureFuse 编码的专用读写函数：

```cpp
// 读取编码值：读取两个相邻字节并 XOR 解码
static UINT8 readCharValueZtlSecureFuse(int at);
static INT16 readShortValueZtlSecureFuse(int a1);
inline unsigned int readLongValueZtlSecureFuse(ULONG *a1);

// 写入编码值：读取当前编码对，计算 XOR 后写入
static bool writeCharValueZtlSecureFuse(int at, UINT8 value);
static bool writeShortValueZtlSecureFuse(int a1, INT16 value);
inline bool writeLongValueZtlSecureFuse(ULONG *a1, unsigned int value);
```

**ZtlSecureFuse 原理**：数据存储为相邻两个字节，读取时将两字节 XOR 得到实际值。写入时先读取当前位置的字节对，计算出需要写入的目标值。

---

## 3. Hooks.h — 钩子与代码洞穴

### 功能
定义了所有 Microsoft Detours 函数钩子和汇编代码洞穴，是训练器核心逻辑的实现层。

### SetHook 函数
包装 Microsoft Detours API，用于钩取游戏函数：

```cpp
bool SetHook(bool enable, void** function, void* redirection) {
    DetourTransactionBegin();
    DetourUpdateThread(GetCurrentThread());
    (enable ? DetourAttach : DetourDetach)(function, redirection);
    DetourTransactionCommit();
}
```

### CodeCave 宏
定义裸函数（naked function）包含内联汇编：

```cpp
#define CodeCave(name) static void __declspec(naked) ##name() { _asm
#define EndCodeCave }
```

### Assembly 命名空间
代码洞穴与 UI 之间的共享状态：

```cpp
namespace Assembly {
    // 钩子捕获的数据
    ULONG curHP = 0, maxHP = 0, curMP = 0, maxMP = 0;
    ULONG curEXP = 0, maxEXP = 0, mapNameAddr = 0x0;
    double hpPercent = 0.00, mpPercent = 0.00, expPercent = 0.00;

    // 过滤器列表
    static std::vector<ULONG> *itemList = new std::vector<ULONG>();
    static std::vector<ULONG> *mobList = new std::vector<ULONG>();
    static std::vector<SpawnControlData*> *spawnControl = new std::vector<SpawnControlData*>();

    // 物品/怪物查找函数
    static String^ findItemNameFromID(int itemID);
    static String^ findMobNameFromID(int mobID);
    static SpawnControlData* __stdcall getSpawnControlStruct();

    // 过滤判断逻辑
    inline void __stdcall itemLog();
    inline bool __stdcall shouldItemBeFiltered();
    inline void __stdcall mobLog();
    inline bool __stdcall shouldMobBeFiltered();
}
```

### 已实现的代码洞穴

| 代码洞穴 | 拦截位置 | 功能描述 |
|---------|---------|---------|
| `StatHook` | `CUIStatusBar::SetNumberValue` (0x008D8581) | 从函数参数中捕获 curHP/maxHP/curMP/maxMP/curEXP/maxEXP，写入 `Assembly` 命名空间变量 |
| `MapNameHook` | `CItemInfo::GetMapString` (0x005CFA48) | 将 `ecx` 寄存器中的地图名称地址保存到 `mapNameAddr` |
| `ItemVacHook` | `CDropPool::TryPickUpDrop` (0x005047AA) | 重定向物品拾取坐标到指定位置 |
| `MouseFlyXHook` / `MouseFlyYHook` | `CVecCtrl::raw__GetSnapshot` | 鼠标飞行：将玩家传送至鼠标所指位置（当 pID 匹配时） |
| `ClickTeleportXHook` / `ClickTeleportYHook` | 同上 | 点击传送：同鼠标飞行，但仅在 `MouseAnimation == 0x0C` 时触发 |
| `MouseTeleportXHook` / `MouseTeleportYHook` | 同上 | 鼠标移动传送：同上，但在 `MouseAnimation == 0x00` 时触发 |
| `MobFreezeHook` | `CVecCtrlMob::WorkUpdateActive` (0x009BCA92) | 将怪物的某个状态值设为 0x06，使其冻结 |
| `MobAutoAggroHook` | 同上 (0x009BCAF7) | 在 `WorkUpdateActive` 调用后，设置怪物的仇恨值为本地玩家的 ID |
| `SpawnPointHook` | `CVecCtrl::SetActive` (0x009B12A8) | 根据当前地图 ID 查找出生点控制数据，修改生成坐标 |
| `ItemFilterHook` | `CDropPool::OnDropEnterField` (0x005059CC) | 判断物品是否应被过滤（白名单/黑名单机制） |
| `MobFilter1Hook` / `MobFilter2Hook` | `CMobPool::SetLocalMob` / `OnMobEnterField` | 判断怪物是否应被过滤 |
| `MissGodmodeHook` | `CUserLocal::SetDamaged` (0x009582E9) | 闪避上帝模式：维护一个计数器，当计数器 > 0 时将伤害设为 0（闪避），计数器归零时让伤害通过并重置 |
| `DupeXHook` | — | DupeEx 吸附：追踪玩家立足点，将怪物吸附到该位置 |
| `SendPacketLogHook` | 封包发送函数 | 记录所有发出的封包到 `sendPacketLogQueue` |

---

## 4. MapleFunctions.h — 游戏辅助函数

### 功能
提供游戏相关的辅助功能，包括窗口操作、传送、指针读取和状态判断。

### PointerFuncs 命名空间
所有指针读取和格式化函数，供 UI 标签绑定调用：

```cpp
namespace PointerFuncs {
    static String^ getCharHP();       // 读取 HP 百分比
    static String^ getCharMP();       // 读取 MP 百分比
    static String^ getCharEXP();      // 读取 EXP 百分比
    static String^ getCharLevel();    // 读取等级
    static String^ getCharJob();      // 读取职业
    static String^ getCharName();     // 读取角色名
    static String^ getWorld();        // 读取世界名
    static String^ getChannel();      // 读取频道号
    static String^ getMapName();      // 读取地图名
    static String^ getMapID();        // 读取地图 ID
    static String^ getCharPos();      // 读取角色坐标
    static String^ getCharPosX();     // 读取角色 X 坐标
    static String^ getCharPosY();     // 读取角色 Y 坐标
    static String^ getMousePos();     // 读取鼠标坐标
    static String^ getAttackCount();  // 读取攻击计数
    static String^ getBreathCount();  // 读取呼吸计数
    static String^ getPeopleCount();  // 读取同图玩家数
    static String^ getMobCount();     // 读取怪物数
    static String^ getItemCount();    // 读取地面物品数
    // ... 更多
}
```

这些函数内部会判断 `isHooksEnabled` 标志：
- **启用时**：通过 `Jump()` 安装代码洞穴钩子，从钩子捕获的数据中获取实时值
- **禁用时**：恢复原始指令字节，指针返回默认值

### HelperFuncs 命名空间
游戏状态判断辅助函数：

```cpp
namespace HelperFuncs {
    static void SetMapleWindowToForeground();  // 将 MS 窗口置于前台
    static RECT GetMapleWindowRect();          // 获取 MS 窗口矩形
    static bool IsInGame();                    // 检查是否在游戏中（mapID != 0）
    static bool ValidToAttack();               // 检查是否可以攻击（attackCount < 99 且在游戏中）
    static bool ValidToLoot();                 // 检查是否可以拾取（peopleCount <= 0 且在游戏中）
    static bool IsInventoryFull();             // 检查背包是否已满（itemCount > 50）
}
```

### Teleport 函数
通过写入指针实现玩家位置传送：

```cpp
static void Teleport(int X, int Y) {
    WritePointer(UserLocalBase, OFS_TeleX, X);
    WritePointer(UserLocalBase, OFS_TeleY, Y);
    WritePointer(UserLocalBase, OFS_Tele, 1);
}

static void Teleport(POINT point) {
    if (isPosValid(point.x, point.y)) {
        WritePointer(UserLocalBase, OFS_TeleX, point.x);
        WritePointer(UserLocalBase, OFS_TeleY, point.y);
        WritePointer(UserLocalBase, OFS_Tele, 1);
        WritePointer(UserLocalBase, OFS_Tele + 4, 1);
    }
}
```

---

## 5. Macro.h — 宏系统

### 功能
基于优先级队列的定时按键发送系统，用于自动吃药和自动攻击。

### KeyMacro 结构
```cpp
ref struct KeyMacro {
    int keyCode;           // 虚拟键码
    MacroType macroType;   // 宏类型（攻击/拾取/Buff/HP药/MP药）

    // 模拟按键（不释放）
    static void SendKey(int Key);
    // 快速连续发送按键
    static void SpamSendKey(int Key, int times);
    // 模拟完整的按下+释放
    static void PressKey(int Key);
};
```

### MacroType 枚举与优先级
```cpp
enum class MacroType {
    LOOTMACRO = 1,       // 优先级最低
    ATTACKMACRO = 2,
    BUFFMACRO = 3,       // 优先级最高
    MPPOTMACRO = 4,
    HPPOTMACRO = 5
};
```

优先级队列按 `MacroType` 数值排序，数值越大优先级越高，Buff 宏最先被执行。

### ManagedPriorityQueue\<T\>
自定义托管二叉小顶堆，替代不兼容 v143 的 `cliext::priority_queue`：

```cpp
generic<typename T>
ref class ManagedPriorityQueue {
private:
    List<T>^ heap;
    Func<T, T, bool>^ compareFunc;
    void SiftUp(int i);
    void SiftDown(int i);
public:
    ManagedPriorityQueue(Func<T, T, bool>^ cmp);
    void push(T item);
    T top();
    void pop();
    bool empty();
    int size();
};
```

### PriorityQueue 工作线程
```cpp
static void MacroQueueWorker() {
    while (!closeMacroQueue) {
        if (macroQueue->empty()) continue;

        KeyMacro^ key = macroQueue->top();
        macroQueue->pop();

        switch (key->macroType) {
            case MacroType::BUFFMACRO:
                if (HelperFuncs::IsInGame())
                    KeyMacro::PressKey(key->keyCode);
                break;
            case MacroType::HPPOTMACRO:
                if (MacrosEnabled::bMacroHP && HelperFuncs::IsInGame()) {
                    // 仅当当前 HP 低于阈值时吃药
                    if (curHP < hpCntDrinkLimit)
                        KeyMacro::PressKey(key->keyCode);
                }
                break;
            case MacroType::ATTACKMACRO:
                if (MacrosEnabled::bMacroAttack && HelperFuncs::ValidToAttack())
                    KeyMacro::SpamSendKey(key->keyCode, 2);
                break;
            case MacroType::LOOTMACRO:
                if (MacrosEnabled::bMacroLoot && HelperFuncs::ValidToLoot())
                    KeyMacro::SpamSendKey(key->keyCode, 2);
                break;
        }
    }
}
```

### Macro 类
```cpp
ref class Macro {
    Threading::Timer^ timer;
    int keyCode, delay;
    MacroType macroType;

    Macro(int keyCode, int delay, MacroType macro) {
        this->keyCode = keyCode;
        this->delay = delay;
        this->macroType = macro;
        timer = gcnew Threading::Timer(
            gcnew TimerCallback(this, &Macro::TimerElapsed),
            nullptr, Timeout::Infinite, delay);
    }

    void Toggle(bool enable) {
        if (enable) timer->Change(delay, delay);
        else timer->Change(Timeout::Infinite, Timeout::Infinite);
    }
};
```

---

## 6. Packet.h/cpp — 封包系统

### 封包结构
```cpp
struct COutPacket {
    int Loopback;
    union {
        PUCHAR Data;
        PVOID Unk;
        PUSHORT Header;
    };
    ULONG Size;
    UINT Offset;
    int EncryptedByShanda;
};

struct CInPacket {
    bool fLoopback;
    int iState;
    union {
        PVOID lpvData;
        struct { ULONG dw; USHORT wHeader; } *pHeader;
        struct { ULONG dw; PUCHAR Data; } *pData;
    };
    ULONG Size;
    USHORT usRawSeq, usDataLen, usUnknown;
    UINT uOffset;
    PVOID lpv;
};
```

### 封包构建器
辅助函数用于构建十六进制格式的封包字符串：

```cpp
void writeByte(String^ %packet, BYTE byte);
void writeBytes(String^ %packet, array<BYTE>^ bytes);
void writeString(String^ %packet, String^ str);  // 长度前缀 + 空字节 + UTF8 数据
void writeInt(String^ %packet, int num);          // 小端序 4 字节
void writeShort(String^ %packet, short num);      // 小端序 2 字节
void writeUnsignedShort(String^ %packet, USHORT num);
```

### 封包发送与接收
```cpp
bool SendPacket(String^ packetStr) {
    // 1. 清理空白字符
    // 2. 替换通配符 "*" 为随机十六进制字符
    // 3. 验证封包字符串合法性
    // 4. 将十六进制字符串转换为字节数组
    // 5. 填充 COutPacket 结构
    // 6. 调用游戏客户端的封包发送函数
}

bool RecvPacket(String^ packetStr) {
    // 类似 SendPacket，但填充 CInPacket 结构并调用接收函数
}
```

### 封包地址
```cpp
ULONG clientSocketAddr = 0x00BE7914;  // 客户端 Socket 指针
ULONG COutPacketAddr = 0x0049637B;    // 发送封包函数地址
ULONG CInPacketAddr = 0x004965F1;     // 接收封包函数地址
```

---

## 7. Structs.h — 数据结构

```cpp
struct SpawnControlData {
    UINT mapID;
    INT spawnX;
    INT spawnY;
};

ref struct PortalData {
    String^ portalName;
    int portalType;
    int xPos, yPos, toMapID;
};

ref struct MapData {
    int mapID;
    String^ islandName;
    String^ streetName;
    String^ mapName;
    Collections::Generic::List<PortalData^>^ portals;
};

ref struct MapPath {
    int mapID;
    PortalData^ portal;
};
```

---

## 8. Settings.h/cpp — 设置管理

通过 XML 序列化保存和恢复窗体控件状态：

```cpp
public ref class Settings {
public:
    static Object^ Deserialize(String^ Path, XmlSerializer^ Deserializer);
    static void Deserialize(Control^ c, String^ Path);
    static void Serialize(String^ Path, XmlSerializer^ Serializer, Object^ Collection);
    static void Serialize(Control^ c, String^ Path);
    static bool isExcluded(Control^ ctrl);
    static String^ GetSettingsPath();
private:
    static void AddChildControls(XmlTextWriter^ xmlSerialisedForm, Control^ c);
    static void SetControlProperties(Control^ currentCtrl, XmlNode^ n);
};
```

---

## 9. 其他辅助模块

### Keyboard.h/cpp
基于 `SendInput` 的键盘模拟：

```cpp
namespace KeyboardInput {
    class Keyboard {
    public:
        static void pressKey(Key key);
        static void releaseKey(Key key);
        static void spamKey(Key key, int times);
        static void holdKeyDown(Key key, int time);
    };
}
```

### Mouse.h/cpp
基于 `SendInput` 的鼠标模拟：

```cpp
namespace MouseInput {
    class Mouse {
    public:
        static void moveTo(int x, int y, bool leftDown, bool rightDown);
        static void leftClick();
        static void doubleLeftClick();
        static void leftDragClickTo(int toX, int toY);
        // ... 更多
    };
}
```

### Log.h/cpp
日志记录模块：

```cpp
public ref class Log sealed {
public:
    static void WriteLine(String^ Message);
    static void WriteLine();
    static void WriteLineToConsole(String^ str);
    static String^ GetLogPath();
};
```

### GeneralFunctions.h
字符串转换工具：

```cpp
static std::string ConvertSystemToStdStr(String^ str1);  // String^ → std::string
static String^ ConvertStdToSystemStr(std::string str1);  // std::string → String^
static PUCHAR atohx(PUCHAR szDestination, LPCSTR szSource);  // ASCII 十六进制转字节
```

---

## 遗留/冗余文件

| 文件 | 问题 |
|------|------|
| `HelperFunctions.h` | 引用不存在的 `"Functions.h"`，且与 `MapleFunctions.h` 中的 `HelperFuncs` 代码重复 |
| `Assembly.h` | 近似 `Hooks.h` 的旧版本副本，不应使用 |
