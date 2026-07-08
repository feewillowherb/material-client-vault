# 01 - MaterialClient 架构与客户端扩展模式

> **证据来源**：Vault 中已有调研文档（2026-05-12 ~ 2026-06-26）+ MaterialMonospec 源码
> **注意**：以下架构描述基于已有调研文档中的代码片段、路径引用及 MaterialMonospec 源码交叉验证

---

## 1. MaterialClient 整体架构

### 1.1 解决方案结构

MaterialClient 采用 **模块化 .NET 解决方案**，核心分为共享层与产品特定模块：

```
MaterialClient.sln
├── MaterialClient.Common/              ← 共享层（所有客户端共用）
│   ├── Entities/                       ← 实体（LicenseInfo, AttachmentFile, WeighingRecord 等）
│   ├── Entities/Enums/                 ← 枚举（WeighingMode, ProductCode, AttachType, SyncStatus 等）
│   ├── Services/                       ← 共享服务
│   │   ├── Hikvision/                  ← 海康威视车牌识别
│   │   ├── Huaxiazhixin/               ← 华夏智信 LPR
│   │   ├── Vzvision/                   ← 臻识 LPR
│   │   ├── Authentication/             ← 授权相关（LicenseService）
│   │   ├── DeviceStatusSignalRClient/ ← SignalR 客户端
│   │   ├── WeighingMatchingService.cs  ← 运单匹配 + 同步（含 SyncNewWaybillAsync）
│   │   ├── SolidWasteService.cs       ← 固废专用服务（Excel 导出等）
│   │   └── AttachmentService.cs        ← 附件管理
│   ├── Api/                            ← Refit 接口定义
│   │   ├── IMaterialPlatformApi.cs     ← 主业务 API（含 SynchronizationOrderAsync）
│   │   ├── IBasePlatformApi.cs         ← 基础授权 API
│   │   └── Dtos/
│   │       ├── SynchronizationOrderInputDto.cs  ← 订单同步 DTO（含 SolidWasteInfo 扩展）
│   │       └── SolidWasteInfoDto.cs             ← 固废信息 DTO
│   ├── Models/                         ← DTO/模型
│   ├── Configuration/                  ← 配置（IOptions 模式）
│   ├── Utils/                          ← 工具类（PathManager 等）
│   └── EntityFrameworkCore/            ← EF Core DbContext + 迁移
│
├── MaterialClient/                     ← 主程序（ProductCode 5000）
│   └── Services/
│
├── MaterialClient.Urban/               ← 城管客户端（ProductCode 5030）
│   ├── Api/
│   │   └── IUrbanManagementApi.cs      ← Refit 接口
│   ├── Services/
│   │   ├── UrbanServerUploadService.cs ← 上传服务
│   │   └── UrbanAttachmentSyncService.cs
│   └── MaterialClientUrbanModule.cs    ← ABP 模块入口
│
└── (新增) MaterialClient.Recycle/      ← 资源化利用厂客户端（ProductCode 5020）
    └── (遵循 Urban 扩展模式)
```

> **重要修正**：SolidWaste（ProductCode 5010）**不是独立客户端项目**。它是 MaterialClient 主程序内的 `WeighingMode.SolidWaste = 1` 业务模式，与 Standard（0）和 UrbanMode（201）共存，通过模式切换区分行为。SolidWaste 的数据上报通过共享的 `IMaterialPlatformApi.SynchronizationOrderAsync` 完成。

### 1.2 技术栈

| 组件 | 技术选型 |
|------|---------|
| UI 框架 | Avalonia |
| DI 容器 | ABP / Microsoft DI |
| ORM | EF Core + SQLite |
| HTTP 客户端 | Refit / IHttpClientFactory |
| 事件机制 | ILocalEventBus |
| 后台服务 | ABP `AsyncPeriodicBackgroundWorkerBase` |
| 日志 | Microsoft.Extensions.Logging |
| 异步模式 | async/await + Rx.NET |
| 实时通信 | SignalR（设备状态同步） |
| 附件存储 | `AttachmentFile` 实体 + `PathManager` 相对路径 |

---

## 2. ProductCode 体系

### 2.1 已有产品

| ProductCode | 产品名称 | WeighingMode | 客户端模块 | 授权方式 | 数据上报 |
|-------------|---------|-------------|-----------|---------|---------|
| **5000** | MaterialClient | `Standard = 0` | `MaterialClient` | `DownloadAuth` → `mlic.lic`；`SetCorpAuthMachineCode` | `IMaterialPlatformApi` |
| **5010** | SolidWaste | `SolidWaste = 1` | `MaterialClient`（同一主程序） | `SendAuthLicense` / `DownloadAuth`（**不使用 JWT**） | `SynchronizationOrderAsync`（`/api/Order/SynchronizationOrder`） |
| **5030** | 城管地磅 | `UrbanMode = 201` | `MaterialClient.Urban` | JWT（`.urban` + 在线激活） | `IUrbanManagementApi` → UrbanManagement |

> **来源**：WeighingMode 枚举定义位于 MaterialMonospec `MaterialClient.Common/Entities/Enums/WeighingMode.cs`；ProductCode 枚举位于 `MaterialClient.Common/Entities/Enums/ProductCode.cs`。

### 2.2 新增产品

| ProductCode | 产品名称 | WeighingMode | 客户端模块 | 授权方式 | 数据上报 |
|-------------|---------|-------------|-----------|---------|---------|
| **5020** | MaterialClient.Recycle | `Recycle = 301`（**需新增**） | `MaterialClient.Recycle`（新增） | 沿用 5010 非 JWT 模式 | **杭州市资源化利用厂接口 V1.0 §2.2**（HMAC-SHA256 认证） |

---

## 3. WeighingMode 枚举（已确认）

| 枚举成员 | 值 | 含义 | ProductCode |
|---------|---|------|-------------|
| `Standard` | 0 | 标准物料验收模式 | 5000 |
| `SolidWaste` | 1 | 固废称重模式 | 5010 |
| `UrbanMode` | 201 | 城管地磅模式 | 5030 |
| **`Recycle`** | **301**（**需新增**） | **资源化利用厂模式** | **5020** |

> **证据**：Standard/SolidWaste/UrbanMode 值已从 MaterialMonospec 源码确认。Recycle=301 来自任务描述，枚举中尚不存在，需在 `WeighingMode.cs` 中新增。

---

## 4. 客户端扩展模式（基于 MaterialClient.Urban 先例）

### 4.1 MaterialClient.Urban 的扩展路径

基于 `06-MaterialClient.Urban迁移拟稿提案.md` 的分析，Urban 客户端的创建涉及：

| 层级 | 改动 | 具体文件 |
|------|------|---------|
| **新增项目** | 新 ABP 模块 | `MaterialClient.Urban/MaterialClientUrbanModule.cs` |
| **实体** | 无需改动 Common 实体（共用 `LicenseInfo`、`WeighingRecord`） | — |
| **API 接口** | 新增 Refit 接口 | `MaterialClient.Urban/Api/IUrbanManagementApi.cs` |
| **上传服务** | 产品特定同步实现 | `MaterialClient.Urban/Services/UrbanServerUploadService.cs` |
| **SignalR** | 复用 Common 层 `DeviceStatusSignalRClient` | — |
| **启动模块** | 新 ABP Module 类 | 注册依赖、配置产品特定服务 |

### 4.2 推断的 Recycle 扩展路径

遵循相同模式：

| 层级 | 改动 | 说明 |
|------|------|------|
| **新增项目** | `MaterialClient.Recycle` | 新 ABP 模块项目 |
| **ABP 模块类** | `MaterialClientRecycleModule.cs` | 注册 Recycle 特定的服务、配置、Refit 客户端 |
| **API 接口** | `IRecycleDataApi.cs` | Refit 接口，对接资源化利用厂 §2.2 端点 |
| **上传/同步服务** | `RecycleDataSyncService.cs` | 替代 SolidWaste 的 `SynchronizationOrderAsync`，调用新接口 |
| **HMAC 签名服务** | `RecycleHmacSignService.cs` | §2.2 接口要求 HMAC-SHA256 签名认证 |
| **WeighingMode 配置** | `appsettings.json` | `WeighingMode: 301` + 目标 API URL + accessKey/secretKey |
| **启动选择** | `Program.cs` 或启动配置 | 根据 ProductCode/WeighingMode 加载对应模块 |

---

## 5. 授权体系对 5020 的影响

### 5.1 现网授权规则（来自 EPIC 文档）

| ProductCode | JWT | SendAuthLicense | DownloadAuth | 备注 |
|-------------|-----|----------------|--------------|------|
| 5000 | ❌ | ✅ | ✅ | `SetCorpAuthMachineCode` |
| 5001 | ✅ | ✅（扩展 AccessCode） | ✅（改为 `DownloadUrbanLicense`） | `.urban` + 在线激活 |
| 5010 | ❌ | ✅ | ✅ | AccessCode 分列，现网逻辑不变 |
| **5020（新增）** | **❌** | **✅** | **✅** | **沿用 5010 模式** |

### 5.2 BasePlatform 侧需要的改动

| 改动 | 位置 | 说明 |
|------|------|------|
| ProductCode 枚举/配置 | BasePlatform.Model | 注册 5020 |
| 授权页 UI | `ProjectAuthAdd.cshtml` / `CompanyAuthAdd.cshtml` | 5020 显示 AccessCode + MachineCode（同 5010 模式） |
| `ListProjects` 筛选 | `ProjectCatalogController.cs` | 包含 5020 |
| `SendAuthLicense` | Redis 载荷 | 5020 沿用现网 JSON（不增加 AccessCode） |

### 5.3 授权方式推荐

**推荐**：5020 沿用 5010 的非 JWT 授权模式。

理由：
1. JWT 仅限 5001，体系已隔离
2. 5010 的 `SendAuthLicense`/`DownloadAuth` 流程成熟
3. Recycle 功能与 SolidWaste 一致，授权对齐更合理
4. 避免 JWT 体系的额外复杂度（私钥管理、`iss` 验签等）

---

## 6. 配置体系

### 6.1 Recycle 配置结构

```json
{
  "ProductCode": 5020,
  "WeighingMode": 301,
  "RecycleSync": {
    "Enabled": true,
    "ApiUrl": "<资源化利用厂接口基础地址>",
    "ApiPath": "/dataCenter/resourcePlace/productTransportRecord/v1/addBatch",
    "AccessKey": "<平台颁发的 accessKey>",
    "SecretKey": "<平台颁发的 secretKey>",
    "PointNumber": "<资源化利用厂唯一标识>",
    "ProductName": "<成品名称>",
    "PollIntervalSeconds": 5,
    "MaxFailCount": 9,
    "TimeoutSeconds": 30
  }
}
```

> **证据**：ApiUrl 路径来自 MaterialMonospec `docs/SyncDoc/杭州市资源化利用厂数据接入接口V1.0.md` §2.2。

### 6.2 配置类

```csharp
public class RecycleSyncOptions
{
    public bool Enabled { get; set; }
    public string ApiUrl { get; set; } = string.Empty;
    public string ApiPath { get; set; } = "/dataCenter/resourcePlace/productTransportRecord/v1/addBatch";
    public string? AccessKey { get; set; }
    public string? SecretKey { get; set; }
    public string? PointNumber { get; set; }
    public string? ProductName { get; set; }
    public int PollIntervalSeconds { get; set; } = 5;
    public int MaxFailCount { get; set; } = 9;
    public int TimeoutSeconds { get; set; } = 30;
}
```

---

## 7. 证据标注

| 结论 | 证据来源 | 可信度 |
|------|---------|--------|
| MaterialClient 模块化结构 | `06-MaterialClient.Urban迁移拟稿提案.md` §8 文件清单 | 高（含具体文件路径） |
| ProductCode 5000/5010/5030 区分 | `01-解决方案.md` §3 表格；EPIC §1.3；MaterialMonospec 源码 | 高（跨文档 + 源码一致） |
| 5010 不使用 JWT | `03-BasePlatform-JWT签发迁移拟稿提案.md` §5.3；EPIC §4 | 高（多处明确声明） |
| ABP + EF Core + SQLite 技术栈 | 多个调研文档描述 | 高 |
| `AttachmentFile` 附件体系 | `02-MaterialClient现有能力对照.md` §1.4 | 高（含实体字段级定义） |
| `WeighingMode` 枚举（Standard=0, SolidWaste=1, UrbanMode=201） | MaterialMonospec 源码 `MaterialClient.Common/Entities/Enums/WeighingMode.cs` | **高（源码确认）** |
| `SynchronizationOrderAsync` 接口签名 | MaterialMonospec 源码 `IMaterialPlatformApi.cs` + `SynchronizationOrderInputDto.cs` | **高（源码确认）** |
| SolidWaste 不是独立客户端，是 WeighingMode=1 模式 | MaterialMonospec 源码 + OpenSpec 规范 | **高（源码 + 规范一致）** |
| §2.2 接口路径和认证方式 | MaterialMonospec `docs/SyncDoc/杭州市资源化利用厂数据接入接口V1.0.md` | **高（文档确认）** |
| `Recycle = 301` / `ProductCode = 5020` 需新增 | MaterialMonospec 源码中不存在，来自任务描述 | **高（确认不存在）** |
