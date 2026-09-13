---
name: read-uasset
description: 离线读取 Unreal .uasset 的证据提取（不开编辑器）：字符串提取与结构化元数据/依赖分析。USE FOR 离线读 uasset、无头证据、需在无编辑器环境核查资产内容、grep 搜不到资产内部引用、批量扫描资产元数据。实时资产操作优先用 unreal-mcp。
---

# Offline UAsset Extraction

在**不开编辑器**的前提下提取 `.uasset` 的证据。实时资产操作优先走 `unreal-mcp`；本技能用于离线核查、无头环境、或 MCP 不可用时。

## 工作流

1. 确认目标文件在本地、只读访问即可满足需求。
2. 快速字符串证据用字符串提取脚本；结构化元数据与依赖线索用解析脚本。
3. 每条结论都标注是「字符串提取」还是「解析出的元数据」。**不推断未读到的图状态、不修改资产**。

### Linux（bash）

```bash
python scripts/parse_uasset.py "<asset.uasset>" --summary
python scripts/parse_uasset.py "<asset.uasset>" --deep --format text
```

### Windows（PowerShell）

```powershell
powershell -File extract_uasset_strings.ps1 -Path "<asset.uasset>"
python scripts\parse_uasset.py "<asset.uasset>" --summary
```

更多模式见同目录 `COMMON_PATTERNS.md`。

完成标准：报告写明读了哪些文件、用的哪种提取方式、以及哪些不确定处需要回到实时编辑器验证。
