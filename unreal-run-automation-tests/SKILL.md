---
name: unreal-run-automation-tests
description: 运行 UE 自动化测试的正确方式——必须带 -unattended 旗标，结果从 Saved/Logs 解析，含 fixture 陷阱
---

# UE 自动化测试运行

> **平台约定**：本机主力环境是 Linux（Arch）——命令以 bash 为先、可直接执行；Windows 专属步骤一律收进「Windows（PowerShell）」小节，不在 Linux 段落里混用。

## 标准命令

### Linux（bash）

先定位引擎安装根——含 `Engine/` 的那一级目录（源码编译版、安装版都适用；本机路径以实际安装为准，不要照抄示例）：

```bash
# 由 UnrealEditor-Cmd 反推引擎根；深度 8 覆盖 <repo>/ue5.x/Engine/Binaries/Linux/ 这类布局，按本机情况调整搜索范围
ENGINE_BIN=$(find "$HOME" /opt -maxdepth 8 -type f -name UnrealEditor-Cmd 2>/dev/null | head -1)
UE_ROOT="${ENGINE_BIN%/Engine/Binaries/Linux/UnrealEditor-Cmd}"
echo "UE_ROOT=$UE_ROOT"   # 为空说明没搜到（如源码引擎还没编译出该二进制），直接改用绝对路径
```

（安装版引擎的根也会登记在 `~/.config/Epic/UnrealEngine/Install.ini`，可用于交叉核对；上面这条 `find` 在本机实测约 1 秒返回。）

跑测试（`$UE_ROOT` 只在同一次 bash 调用内有效；跨调用先 `export UE_ROOT=...`）：

```bash
"$UE_ROOT/Engine/Binaries/Linux/UnrealEditor-Cmd" \
  "<ProjectRoot>/<Project>/<Project>.uproject" \
  -unattended \
  -ExecCmds="Automation RunTests <Project>" \
  -testexit="Automation Test Queue Empty"
```

`.uproject` 路径与 `-ExecCmds` / `-testexit` 参数两端一致，只有引擎可执行文件路径不同。

### Windows（PowerShell）

Windows 专属路径（Linux 上无对应写法），原文保留：

```powershell
& "~\Project\UnrealEngine\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "<ProjectRoot>/<Project>/<Project>.uproject" -unattended -ExecCmds="Automation RunTests <Project>" -testexit="Automation Test Queue Empty"
```

（安装版引擎根通常为 `C:\Program Files\Epic Games\UE_x.y`；上例是自编译引擎路径。Linux 侧可执行文件是 `Engine/Binaries/Linux/UnrealEditor-Cmd`，无 `.exe` 后缀。）

## 关键陷阱

- **必须带 `-unattended`**。不带时自动化系统在 `FWaitForInteractiveFrameRate` 上永久等待（编辑器无交互帧率，白天机器被使用时必现），测试永不开始，编辑器看似挂死。日志特征：`LogEngineAutomationLatentCommand: FWaitForInteractiveFrameRate: Waited N seconds` 反复出现。
- 加 `-unattended` 后 68 个测试约 45-60 秒跑完；stdout 只有 OpenXR 报错属正常（无头环境无 XR 运行时）。
- 不要加 `-nullrhi`（会导致测试不执行直接退出）。

## 结果解析

结果在 `<Project>/Saved/Logs/<Project>.log`（每次运行覆盖），解析 `Result={成功}` / `Result={失败}` 计数，失败详情在 `BeginEvents: <测试路径>` 与 `EndEvents:` 之间。日志内时间戳是 UTC（本地 UTC+8）。

在项目根用 bash 提取（`Result={...}` 的具体取值以实际日志为准，下面按取值聚合，不写死文案）：

```bash
LOG="Saved/Logs/<Project>.log"
grep -o 'Result={[^}]*}' "$LOG" | sort | uniq -c   # 成功/失败计数
grep -n 'BeginEvents:' "$LOG" | tail -20           # 失败用例入口，详情看紧随其后的行
```

## 测试代码注意事项

- fixture 用 `NewObject` 创建 UUserWidget 后，若被测代码内部走 `CreateWidget(this, Class)`，会 ensure 失败（`ParentUserWidget->WidgetTree` null）——需手动 `Widget->WidgetTree = NewObject<UWidgetTree>(Widget)`。
- 反射检查（FindFunctionByName）的目标函数必须标 UFUNCTION，否则静默找不到。
- UE RPC 的 cpp 定义必须用 `_Implementation` 后缀（否则 LNK2005+LNK2001）。
