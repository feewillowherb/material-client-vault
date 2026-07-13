# CodeRef Vault Guide

This vault follows the standard CodeRef root layout.

- `docs/` stores research notes, explanations, and walkthroughs.
- `repos/` 目录保留但当前不存放源码仓库（Windows Junction 本地映射已在 `.gitignore` 中排除）。
- `index.yaml` is the canonical metadata entry for this vault root.

## 项目仓库位置

**项目源码仓库不在本 vault 内，而位于 [MaterialMonospec](../../../MaterialMonospec/)（OpenSpec 单仓库）**，包含：

| 仓库 | 说明 |
|------|------|
| `MaterialClient` | 客户端主仓库（含 MaterialClient.Common、MaterialClient.Urban、SolidWaste 等模块） |
| `UrbanManagement` | 城管服务端（ABP 10 + SignalR Hub） |
| `BasePlatform` | 授权与产品管理平台 |
| `BasePlatform.PublicApi` | 公共 API 层 |

MaterialMonospec 同时承载 OpenSpec 规范（`openspec/changes/`），是规范与代码的统一仓库。调研文档中引用的源码路径（如 `MaterialClient.Common/Services/DeviceStatusSignalRClient.cs`）均指 MaterialMonospec 内的对应文件。

The canonical `index.yaml` fields are:

- `schemaVersion`
- `type`
- `title`
- `docsRoot`
- `reposRoot`

Keep this structure stable so assistants and tools can understand the vault quickly.

## Output Language

- 默认输出语言为中文。专用名词（如函数名、命名空间、NuGet 包名、API 名称等）保留原文不翻译。

## Research Output Format

每次调研产出统一放置在 `docs/` 下的独立文件夹中，而非散落为多个独立文件。文件夹命名格式：

```
docs/<YYYY-MM-DD>-<提案或主题名称>/
```

示例：

```
docs/2026-01-01-topic-name/
├── 00-调研总览.md
├── 01-使用指南.md
├── 02-技术参考.md
├── 03-快速参考.md
└── ...
```

规则：

- 文件夹名称由日期和提案名称组成，使用连字符分隔。
- 文件夹内的文档按编号排序，编号从 `00` 开始。
- 每个调研文件夹应包含一个 `00-调研总览.md` 作为入口索引。
