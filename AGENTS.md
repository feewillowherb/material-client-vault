# CodeRef Vault Guide

This vault follows the standard CodeRef root layout.

- `docs/` stores research notes, explanations, and walkthroughs（历史调研可保留于此）。
- `repos/` 目录保留但当前不存放源码仓库（Windows Junction 本地映射已在 `.gitignore` 中排除）。
- `index.yaml` is the canonical metadata entry for this vault root.

## 项目仓库位置

**项目源码与 OpenSpec 位于 [MaterialMonospec](../../../MaterialMonospec/)**（OpenSpec 单仓库），包含：

| 仓库 | 说明 |
|------|------|
| `MaterialClient` | 客户端主仓库（含 MaterialClient.Common、MaterialClient.Urban、SolidWaste 等模块） |
| `UrbanManagement` | 城管服务端（ABP 10 + SignalR Hub） |
| `BasePlatform` | 授权与产品管理平台 |
| `BasePlatform.PublicApi` | 公共 API 层 |

**新建调研文档的产出格式与语言约定**以 MaterialMonospec 为准：

→ [`MaterialMonospec/docs/AGENTS.md`](../../../MaterialMonospec/docs/AGENTS.md)

调研中引用的源码路径（如 `MaterialClient.Common/Services/DeviceStatusSignalRClient.cs`）均指 MaterialMonospec 内 `repos/` 下的对应文件。

The canonical `index.yaml` fields are:

- `schemaVersion`
- `type`
- `title`
- `docsRoot`
- `reposRoot`

Keep this structure stable so assistants and tools can understand the vault quickly.
