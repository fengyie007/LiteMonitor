# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build
dotnet build -c Release

# Publish (single output)
dotnet publish -c Release

# Run (requires admin elevation - see app.manifest)
dotnet run
```

**Requirements**: .NET 8 SDK, Windows only (WinForms). The app requires administrator privileges for hardware monitoring via LibreHardwareMonitorLib.

No test projects exist. No CI/CD pipelines are configured.

## Architecture Overview

LiteMonitor is a Windows desktop system monitor (CPU, GPU, RAM, disk, network, battery, FPS) built with **WinForms** on .NET 8. It renders a transparent overlay window with real-time hardware metrics.

### Layer Structure

```
src/
├── System/          # Core services: hardware polling, auto-update, auto-start, web server
├── UI/              # WinForms windows, renderers, helpers, custom controls
├── Core/            # Settings, themes, languages, metrics, utilities
├── Plugins/         # JSON-driven plugin system (weather, crypto, stocks, etc.)
└── ThemeEditor/     # Built-in theme editor
```

### Key Design Patterns

- **Singleton services**: `Settings`, `ThemeManager`, `LanguageManager`, `HardwareMonitor` — accessed via static `Instance` properties
- **Dual-Helper architecture**: `MainForm_Transparent` delegates work to `MainFormWinHelper` (window behavior) and `MainFormBizHelper` (business logic/menus)
- **Draft-Commit settings**: `Settings` uses `Draft` property for editing; changes are saved to `settings.json` only on explicit save
- **Strategy for taskbar**: `TaskbarStrategy` with Win10/Win11 variants in `src/UI/Helpers/TaskbarStrategies/`
- **Dual renderer**: `UIRenderer` (vertical) and `HorizontalRenderer` (horizontal) handle different layout modes

### Data Flow

1. `HardwareMonitor` polls hardware sensors via LibreHardwareMonitorLib + performance counters
2. Hardware services in `src/System/HardwareServices/` collect individual metrics (CPU, GPU, etc.)
3. `MetricItem` objects carry metric data (label, value, unit, alert level)
4. `UIController` orchestrates timer-driven refresh → calls active renderer
5. Renderers draw metrics onto the transparent form using GDI+

### Entry Point

`src/System/Program.cs` — enforces single-instance via mutex, sets up global exception handlers, launches `MainForm_Transparent`.

## Key Subsystems

### Settings (`src/Core/Settings.cs`)
100+ properties with JSON serialization. Adding a new setting: add property to `Settings` class, it auto-persists to `settings.json`. UI for settings lives in `src/UI/Settings/` pages.

### Themes (`resources/themes/*.json`)
JSON theme definitions control colors, fonts, opacity, borders. Loaded by `ThemeManager`. 10 built-in themes.

### Languages (`resources/lang/*.json`)
9 languages (en, zh, ja, ko, fr, de, es, ru, zh-tw). `LanguageManager` provides `T(key)` for translations.

### Plugin System (`src/Plugins/`)
Declarative JSON plugins in `resources/plugins/`. Each plugin defines HTTP requests, data extraction (JSONPath/regex), and display formatting. `PluginManager` → `PluginExecutor` → `PluginProcessor` pipeline. Users can add plugins without code changes.

### Hardware Services (`src/System/HardwareServices/`)
14 service classes, each responsible for one hardware category. `HardwareMonitor` coordinates them all. Services use `LibreHardwareMonitorLib` sensors and Windows performance counters as fallbacks.

### Taskbar Integration (`src/UI/TaskbarForm.cs`)
Embeds metrics into the Windows taskbar. Uses different strategies for Win10 vs Win11 (`TaskbarStrategy`).

## Version Management

- Version in `LiteMonitor.csproj` (`<Version>` property) and `resources/version.json`
- Both must be updated together for releases
- `AssemblyVersion` stays at 1.0.0.0 for COM stability

## Code Conventions

- **Language**: C# 12 with nullable references enabled, implicit usings
- **Namespaces**: `LiteMonitor.System`, `LiteMonitor.UI`, `LiteMonitor.Core`, `LiteMonitor.Plugins`
- **UI strings**: Always use `LanguageManager.Instance.T("key")` — never hardcode user-facing text
- **Settings access**: `Settings.Instance.PropertyName` for reads; `Settings.Instance.Draft.PropertyName` for edits in settings UI
- **Rendering**: All drawing in renderers uses GDI+ (`Graphics`), must account for DPI scaling via `UIController.DpiScale`
