---
name: unreal-run-automation-tests
description: 运行 UE 自动化测试的正确方式——必须带 -unattended 旗标，结果从 Saved/Logs 解析，含 fixture 陷阱
---

# UE 自动化测试运行

## 标准命令

```powershell
& "~\Project\UnrealEngine\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "<ProjectRoot>/<Project>/<Project>.uproject" -unattended -ExecCmds="Automation RunTests <Project>" -testexit="Automation Test Queue Empty"
```

## 关键陷阱

- **必须带 `-unattended`**。不带时自动化系统在 `FWaitForInteractiveFrameRate` 上永久等待（编辑器无交互帧率，白天机器被使用时必现），测试永不开始，编辑器看似挂死。日志特征：`LogEngineAutomationLatentCommand: FWaitForInteractiveFrameRate: Waited N seconds` 反复出现。
- 加 `-unattended` 后 68 个测试约 45-60 秒跑完；stdout 只有 OpenXR 报错属正常（无头环境无 XR 运行时）。
- 不要加 `-nullrhi`（会导致测试不执行直接退出）。

## 结果解析

结果在 `<Project>/Saved/Logs/<Project>.log`（每次运行覆盖），解析 `Result={成功}` / `Result={失败}` 计数，失败详情在 `BeginEvents: <测试路径>` 与 `EndEvents:` 之间。日志内时间戳是 UTC（本地 UTC+8）。

## 测试代码注意事项

- fixture 用 `NewObject` 创建 UUserWidget 后，若被测代码内部走 `CreateWidget(this, Class)`，会 ensure 失败（`ParentUserWidget->WidgetTree` null）——需手动 `Widget->WidgetTree = NewObject<UWidgetTree>(Widget)`。
- 反射检查（FindFunctionByName）的目标函数必须标 UFUNCTION，否则静默找不到。
- UE RPC 的 cpp 定义必须用 `_Implementation` 后缀（否则 LNK2005+LNK2001）。
