# Changelog

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
