# 构建与部署

本文档描述如何从源码构建 Timelapse DLL 以及将其部署到 MapleStory v83 客户端。

---

## 1. 环境要求

| 依赖 | 版本 | 说明 |
|------|------|------|
| Visual Studio 2022 | 17.x | 带 C++/CLI 支持的工作负载 |
| Windows SDK | 10.0.22621.0 | 项目属性中指定的版本 |
| Platform Toolset | v143 (MSVC 19.3x) | VS2022 的工具集 |
| C++ 标准 | C++20 (`stdcpp20`) | C++/CLI 支持的最高版本 |
| .NET Framework | 4.7.2 | WinForms 所需的 CLR 版本 |

### Visual Studio 工作负载

确保安装以下工作负载：
- **使用 C++ 的桌面开发** — 包含 MSVC 编译器和 Windows SDK
- **.NET 桌面开发** — 包含 CLR 和 WinForms 支持

### 单个组件

确保勾选：
- **C++/CLI 支持** — 用于 `/clr` 编译
- **Windows 10 SDK (10.0.22621.0)** — 项目属性中指定的版本

---

## 2. 构建步骤

### 2.1 克隆仓库

```bash
git clone <repository-url> Timelapse
cd Timelapse
```

### 2.2 打开项目

使用 Visual Studio 2022 打开 `Timelapse.sln`（或 `Timelapse.vcxproj`）。

### 2.3 配置选择

| 配置 | 说明 |
|------|------|
| **Debug** | 包含调试信息，用于开发调试 |
| **Release** | 优化过的构建，用于实际部署 |

### 2.4 构建项目

在 Visual Studio 中：
1. 选择 **Release** 配置
2. 选择 **x86**（Win32）平台（MapleStory v83 是 32 位进程）
3. 右键项目 → **生成**（Build）

或使用命令行：
```powershell
msbuild Timelapse.vcxproj /p:Configuration=Release /p:Platform=Win32
```

### 2.5 构建输出

构建成功后，DLL 文件位于：
```
Timelapse/Release/Timelapse.dll
```

---

## 3. 部署步骤

### 3.1 准备注入器

推荐使用 **Extreme Injector v3.7.2**（或兼容的 DLL 注入器）。

下载地址：搜索 "Extreme Injector v3.7.2"

### 3.2 注入设置

1. 打开 Extreme Injector
2. 在 **DLL to inject** 中选择 `Timelapse.dll` 的路径
3. 在 **Process Name** 中选择或输入 `MapleStory.exe`
4. **推荐**：勾选 **Stealth Inject** 选项以降低被检测的风险

### 3.3 注入 DLL

1. 确保 MapleStory v83 客户端正在运行
2. 点击 **Inject** 按钮
3. 如果一切正常，Timelapse 的 WinForms 窗口将会出现

### 3.4 使用注意事项

- **注入时机**：在角色登录到游戏世界后注入（而非在登录界面）
- **多次注入**：不要对同一进程多次注入同一 DLL
- **管理员权限**：注入器和 MapleStory 可能需要管理员权限运行
- **杀毒软件**：某些杀毒软件可能阻止 DLL 注入，需添加白名单

---

## 4. 项目结构

```
Timelapse/
├── Timelapse/
│   ├── Timelapse.vcxproj          # 主项目文件
│   ├── MainForm.h                  # WinForms UI 定义
│   ├── MainForm.cpp               # UI 事件处理实现
│   ├── Addresses.h                # 内存地址定义
│   ├── Hooks.h                    # Detours 钩子 & 代码洞穴
│   ├── Memory.h                   # 内存读写工具
│   ├── MapleFunctions.h           # 游戏辅助函数
│   ├── Macro.h                    # 宏系统
│   ├── Packet.h / Packet.cpp     # 封包处理
│   ├── Keyboard.h / Keyboard.cpp # 键盘模拟
│   ├── Mouse.h / Mouse.cpp       # 鼠标模拟
│   ├── Structs.h                  # 数据结构
│   ├── Settings.h / Settings.cpp # 设置序列化
│   ├── Log.h / Log.cpp           # 日志模块
│   ├── GeneralFunctions.h        # 字符串转换工具
│   ├── HelperFunctions.h         # ⚠️ 冗余文件，待删除
│   ├── Assembly.h                # ⚠️ 旧版本副本，待删除
│   ├── detours.h / detours.lib   # Microsoft Detours 库
│   └── Timelapse.rc              # 资源文件（嵌入物品/怪物/地图数据）
├── docs/                          # 文档目录
└── Timelapse.sln                 # 解决方案文件
```

---

## 5. 常见构建问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| `fatal error C1192: #using requires C++/CLI` | 项目未启用 `/clr` | 检查项目属性 → C/C++ → 所有选项 → 公共语言运行时支持 → `/clr` |
| `LNK2019: unresolved external symbol` | 缺少库文件或符号 | 确保 `detours.lib` 已正确链接；检查库路径 |
| `error C7510: 'stdcpp20'` | C++/CLI 不支持 C++20 之后的版本 | 使用 `stdcpp20`（C++/CLI 支持的最高版本） |
| `LNK2022: metadata operation failed` | CLR 元数据冲突 | 确保所有源文件使用相同的 `/clr` 设置 |
| Windows SDK 版本错误 | SDK 版本不匹配 | 安装项目属性中指定的 SDK (10.0.22621.0) 或修改项目属性匹配已安装的版本 |

---

## 6. 安全警告

- 本工具仅用于**学习和研究**目的
- 使用游戏辅助工具违反 MapleStory 的服务条款，可能导致账号被封禁
- 使用此类工具存在安全风险，请自行承担责任
