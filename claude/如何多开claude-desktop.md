## 对话概览
用户希望在 Windows 上同时打开多个 Claude Desktop 窗口（例如个人账号和工作账号各一个）。本次对话涵盖了 `--user-data-dir` 技巧的使用方法、用户遇到报错的原因分析，以及正确的解决方案。

---

## 核心议题

### 议题一：Claude Desktop 默认只能开一个窗口吗？

**讨论内容：**
Claude Desktop（聊天版）默认是单窗口、单账号的应用程序。但可以通过传入 `--user-data-dir` 参数来启动多个完全隔离的实例。

**解决方法（快捷方式目标路径）：**

```
C:\Users\%USERNAME%\AppData\Local\AnthropicClaude\claude.exe --user-data-dir="C:\Users\%USERNAME%\AppData\Roaming\Claude-Work"
```

每个不同的 `--user-data-dir` 路径都会创建一个独立实例，各自拥有：
- 独立的登录账号
- 独立的 MCP 服务器配置
- 独立的设置与记忆

> ⚠️ 注意：`%USERNAME%` 在快捷方式的"目标"栏中可能无法自动展开，建议直接填写你的实际 Windows 用户名。

---

### 议题二：报错——"找不到 claude.exe"

**问题描述：**
用户虽然是从 Google Chrome 下载 Claude Desktop 的，但实际安装的是 **微软商店（Microsoft Store）版本**。商店版应用以 MSIX 格式安装，存放在系统保护目录 `C:\Program Files\WindowsApps\` 中，无法直接通过路径访问 `.exe` 文件。

**确认方法：**
在 PowerShell 中运行以下命令，可以看到 Claude 是商店包（`InstallLocation` 指向 `WindowsApps` 目录）：

```powershell
Get-AppxPackage *Claude*
```

**根本原因：**
`--user-data-dir` 技巧只适用于**直接下载的 `.exe` 安装版**，不适用于商店/MSIX 版本。

---

### 议题三：解决方案——安装直接下载的 EXE 版本

**第一步：** 从以下官方直链下载安装包（非商店版）：

```
https://claude.ai/api/win32/exe/latest/redirect
```

> 这是在 Google 搜索 "Claude desktop exe" 时的第二条结果，直接下载 `.exe` 安装包。

**第二步：** 安装后，Claude 会被放置在：

```
C:\Users\你的用户名\AppData\Local\AnthropicClaude\claude.exe
```

**第三步：** 创建快捷方式并加上 `--user-data-dir` 参数，即可正常使用多开功能。

---

## 完整多开设置示例

创建两个 Windows 快捷方式，目标路径分别为：

**个人账号：**
```
C:\Users\你的用户名\AppData\Local\AnthropicClaude\claude.exe --user-data-dir="C:\Users\%USERNAME%\AppData\Roaming\Claude-Personal"
```

**工作账号：**
```
C:\Users\你的用户名\AppData\Local\AnthropicClaude\claude.exe --user-data-dir="C:\Users\%USERNAME%\AppData\Roaming\Claude-Work"
```

建议将两个快捷方式分别命名为"Claude - 个人"和"Claude - 工作"，方便区分。

---

## 关键概念解释

### `--user-data-dir` 参数
这是 Electron 框架（Claude Desktop 的底层技术）提供的命令行参数。它告诉应用程序将所有数据（登录状态、设置、MCP 配置等）存储在指定的自定义文件夹中，而不是默认位置。两个指向不同文件夹的实例之间完全隔离，互不影响。

### MSIX / 微软商店包（Microsoft Store Package）
一种现代 Windows 应用打包格式。以此方式安装的应用存放在系统保护目录中，无法直接通过路径调用其 `.exe` 文件，因此 `--user-data-dir` 技巧对商店版无效。

### MCP 服务器（Model Context Protocol，模型上下文协议）
可以连接到 Claude Desktop 的扩展/集成功能（例如文件访问、浏览器控制、第三方工具等）。每个 `--user-data-dir` 实例都有独立的 MCP 配置，需要分别单独设置。

### Electron 框架
用于构建 Claude Desktop 的跨平台桌面应用框架（VS Code、Slack、Discord 也使用同一框架）。本质上是一个内嵌 Chromium 浏览器运行网页应用的环境。`--user-data-dir` 是 Electron/Chromium 的标准功能。

---

## 行动清单

| 序号 | 任务 | 状态 |
|------|------|------|
| 1 | 从 `https://claude.ai/api/win32/exe/latest/redirect` 下载直装版 `.exe` | 待处理 |
| 2 | 安装（可与商店版共存，也可卸载商店版） | 待处理 |
| 3 | 创建带 `--user-data-dir` 参数的快捷方式 | 待处理 |
| 4 | 如需 MCP 扩展，为每个实例分别配置 | 待处理 |

---

## 总结

`--user-data-dir` 是在 Windows 上实现 Claude Desktop 多开的正确方法，但**仅适用于直接下载的 EXE 安装版**，对微软商店版无效。商店版将 `.exe` 隐藏在受保护的系统目录中，导致参数无法传入。解决方法很简单：从 `https://claude.ai/api/win32/exe/latest/redirect` 下载直装版并安装，然后为每个账号创建带不同 `--user-data-dir` 路径的快捷方式即可实现多开。
