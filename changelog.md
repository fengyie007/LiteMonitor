# Changelog

## 2026-10-21

### refactor: 横屏间距与 Win11 圆角改为由主题配置驱动

**背景：**
1. 横屏模式下「组间距」与「标签↔值间距」原先硬编码读取设置项 `HorizontalItemSpacing` / `HorizontalInnerSpacing`，主题里的 `GroupSpacing` / `GroupPadding` 在横屏下完全不生效
2. Win11 下窗口圆角写死为 DWM 默认圆角，主题 `CornerRadius` 被忽略，无法通过主题关闭或调小

**修改方案：**
- 横屏「组间距」改为读取主题 `Layout.GroupSpacing`（`HorizontalLayout.cs`）
- 横屏「标签↔值间距」改为读取主题 `Layout.GroupPadding`（`HorizontalLayout.cs`）
- Win11 圆角按主题 `Layout.CornerRadius` 映射 DWM 策略：`0 → 不圆角`、`≤6 → 小圆角`、`>6 → 默认圆角`（`MainFormWinHelper.cs`）
- 移除已失效的设置项 `HorizontalItemSpacing`、`HorizontalInnerSpacing` 及其设置界面入口

**修改文件：**
- `src/UI/HorizontalLayout.cs` — 横屏组间距 / 标签↔值间距改读主题
- `src/UI/Helpers/MainFormWinHelper.cs` — Win11 圆角改读主题，新增 `DWMWCP_DONOTROUND` / `DWMWCP_ROUNDSMALL` 常量
- `src/Core/Settings.cs` — 移除 `HorizontalItemSpacing`、`HorizontalInnerSpacing`
- `src/UI/Settings/MainPanelPage.cs` — 移除横屏间距设置入口
- `resources/themes/my.json` — 自定义主题紧凑化（缩小行高、内边距、字号与圆角）

**其它：**
- 主面板宽度预设新增 `120` / `150` 选项（`src/UI/MenuManager.cs`）

**注意：** 主题布局值运行时会乘 `DPI × UIScale`。内置主题 `GroupSpacing` 默认 `30`、`CornerRadius` 默认 `12`，横屏间距与 Win11 圆角会随之变化，可按需调小。

## 2026-03-22

### fix: 修复置顶在虚拟桌面切换和全屏应用中失效的问题

**问题原因：**
1. 原有置顶逻辑仅在窗口启动时通过 `TopMost = false → true` 的 toggle hack 设置一次，之后无任何周期性维护机制
2. 在全屏应用（如 VMware、无边框全屏游戏）中，`WS_EX_TOPMOST` 标志可能仍然存在但窗口实际被覆盖，仅检查标志位无法发现问题

**修复方案：**
- `MainFormWinHelper` 新增 `ReapplyTopMost()` 方法，通过 Win32 `SetWindowPos(HWND_TOPMOST)` 无条件强制置顶，不依赖 `WS_EX_TOPMOST` 标志位检测
- `MainForm_Transparent` 新增置顶守护定时器（3秒间隔），周期性强制恢复置顶状态
- 定时器包含弹窗保护：当本应用有其他活动窗口（如设置面板）时跳过，避免主窗口覆盖自身对话框
- 移除 `OnShown` 中的旧 toggle hack（`TopMost = false → delay → true → BringToFront`）

**修改文件：**
- `src/UI/Helpers/MainFormWinHelper.cs` — 新增 `ReapplyTopMost()` 方法及相关 Win32 API 声明
- `src/UI/MainForm_Transparent.cs` — 新增置顶守护定时器，替换启动时的一次性 toggle hack

dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true
