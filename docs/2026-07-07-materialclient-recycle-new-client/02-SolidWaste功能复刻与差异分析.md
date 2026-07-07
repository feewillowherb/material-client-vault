# 02 - SolidWaste 功能复刻与差异分析

> **假设**：A1（从已有文档推断 SolidWaste 功能集）、A2（推断 SynchronizationOrderAsync 为数据上报接口）
> **注意**：Vault 中无 SolidWaste 源码，以下分析基于间接证据

---

## 1. SolidWaste 功能推断

### 1.1 从已知信息推断的功能集

根据 Vault 文档中关于 SolidWaste（ProductCode 5010）的描述和 MaterialClient 通用能力，SolidWaste 客户端应包含以下功能：

| 功能模块 | 功能描述 | 复用基础 |
|---------|---------|---------|
| **称重数据采集** | 串口通信连接地磅，实时读取重量数据 | `MaterialClient.Common/Services/` 串口服务 |
| **重量稳定性检测** | 时间窗口分析（3000ms），判定重量稳定 | `TruckScaleWeightService`（Common 层） |
| **车牌识别** | 海康威视 SDK 集成，被动识别 + 主动抓拍 | `HikvisionLprService`（Common 层） |
| **图片保存** | LRP 抓拍图片保存为 `AttachmentFile`（`AttachType.LprCapturePhoto = 4`） | `AttachmentService` + `PathManager`（Common 层） |
| **称重记录持久化** | 稳定后写入本地 SQLite（`WeighingRecord` 实体） | Common 层 EF Core |
| **数据上报** | 定时扫描未同步记录，通过 `SynchronizationOrderAsync` 上报到服务端 | **SolidWaste 特有服务** |
| **同步状态管理** | 未同步/失败/成功/不再上传，失败重试（MaxFailCount=9） | 可复用同步状态枚举和逻辑 |
| **附件同步** | 称重关联的图片随称重记录一并上报 | `AttachmentService`（Common 层） |
| **授权验证** | `SendAuthLicense` / `DownloadAuth`（非 JWT） | Common 层授权服务 |
| **设备在线状态** | 本地检测地磅、相机等设备在线状态 | `LprDeviceOnlineStatusService`（Common 层） |
| **SignalR 通信** | 与服务端实时通信（状态同步、JWT 验签等） | `DeviceStatusSignalRClient`（Common 层） |

### 1.2 SolidWaste 的特征（与 Urban 的对比）

| 维度 | SolidWaste (5010) | MaterialClient.Urban (5001) | Recycle (5020) |
|------|-------------------|----------------------------|----------------|
| **授权方式** | SendAuthLicense / DownloadAuth（非 JWT） | JWT（`.urban` + 在线激活） | **同 SolidWaste** |
| **数据上报接口** | `SynchronizationOrderAsync` | `IUrbanManagementApi`（Refit） | **资源化利用厂接口 §2.2** |
| **数据中转** | 直连或通过 UrbanManagement 中转 | 通过 UrbanManagement | **直连资源化利用厂 API** |
| **AccessCode** | 有（接入码分列） | 有（`LicenseInfo.AccessCode`） | **有** |
| **MachineCode** | 有（设备码） | 有（JWT claim `machineCode`） | **有** |
| **政府平台出站** | 可能通过 UrbanManagement 转发政府平台 | 是（`GovSyncBackgroundWorker`） | **否（走资源化利用厂接口）** |

---

## 2. Recycle 与 SolidWaste 的差异点

### 2.1 唯一核心差异：数据上报接口

| 维度 | SolidWaste | Recycle |
|------|-----------|---------|
| **上报接口** | `SynchronizationOrderAsync`（服务端接口，推测在 MaterialPlatform / UrbanManagement 中） | 杭州市资源化利用厂数据接入接口 V1.0 §2.2（**外部接口**） |
| **上报方向** | 客户端 → 内部服务端（MaterialPlatform / UrbanManagement） | 客户端 → **外部第三方**（资源化利用厂） |
| **接口规范** | 内部 Refit 接口，签名可控 | **外部文档约束**，需严格遵循 §2.2 定义的字段、格式、认证方式 |

### 2.2 其它维度完全一致

任务明确声明「所有客户端功能与 SolidWaste 一致」，因此除数据上报接口外，以下模块应完全复用：

- 称重数据采集（串口通信、稳定性检测）
- 车牌识别（海康威视 SDK）
- 图片保存（`AttachmentFile` 体系）
- 称重记录持久化
- 同步状态管理
- 授权验证（非 JWT）
- 设备在线检测
- UI 交互（Avalonia 界面）

### 2.3 推断的 Recycle 模块结构

```
MaterialClient.Recycle/
├── MaterialClientRecycleModule.cs     ← ABP 模块注册
├── Api/
│   └── IRecycleDataApi.cs              ← Refit 接口（对接资源化利用厂 §2.2）
├── Models/
│   ├── RecycleSyncOptions.cs           ← 配置模型
│   └── RecycleWeightRequest.cs         ← §2.2 请求体 DTO（待接口文档确认）
├── Services/
│   ├── RecycleDataSyncService.cs       ← 核心同步服务（替代 SynchronizationOrderAsync）
│   └── RecycleUploadService.cs         ← 数据转换 + 上报逻辑
└── Configuration/
    └── RecycleDefaults.cs             ← 默认配置值
```

---

## 3. WeighingMode = 301 分析

### 3.1 WeighingMode 的可能含义

**假设 A4**：`WeighingMode` 是 MaterialClient 内部的称重业务模式枚举，类似于 GovClient 中的 `networkCompany` 枚举。

基于已有证据推断：

| WeighingMode (推测) | 含义 | 类比 |
|--------------------|------|------|
| 未知（其他值） | 标准/普通模式 | MaterialClient 5000 |
| 未知（其他值） | 城管/政府模式 | MaterialClient.Urban 5001 |
| 未知（其他值） | 固废模式 | SolidWaste 5010 |
| **301** | **资源化利用/回收模式** | **MaterialClient.Recycle 5020** |

### 3.2 WeighingMode 可能影响的维度

根据 GovClient 中 `networkCompany` 枚举的影响范围推断：

| 维度 | 影响内容 |
|------|---------|
| **数据模型字段** | 不同模式可能需要不同的字段组合（如固废模式的特有字段） |
| **API 端点选择** | 根据模式选择不同的上报 URL / 接口 |
| **数据转换逻辑** | 字段映射、单位转换、格式化方式 |
| **业务规则** | 如是否需要进出匹配、皮重处理等 |
| **UI 行为** | 称重界面可能根据模式显示不同控件 |

### 3.3 301 在代码中的使用位置（推测）

```csharp
// 推测位置 1：配置文件
"WeighingMode": 301

// 推测位置 2：启动时模块选择
if (weighingMode == 301)
{
    // 加载 MaterialClient.Recycle 模块
}

// 推测位置 3：同步服务中的数据转换
if (WeighingMode == 301)
{
    // 使用 RecycleWeightRequest 格式
    // 调用资源化利用厂接口
}
else if (/* SolidWaste 模式 */)
{
    // 使用 SynchronizationOrderAsync
}
```

---

## 4. 复用与新增对照表

### 4.1 可直接从 Common 层复用（零改动）

| 组件 | 说明 |
|------|------|
| `HikvisionLprService` | 海康威视车牌识别 |
| `AttachmentFile` + `AttachmentService` | 附件保存、路径管理 |
| `PathManager` | 相对/绝对路径转换 |
| `LprDeviceOnlineStatusService` | 设备在线检测 |
| `DeviceStatusSignalRClient` | SignalR 通信 |
| `ISettingsService` | 配置管理 |
| `TruckScaleWeightService` | 称重服务（串口 + 稳定性检测） |
| `WeighingRecord` 实体 | 称重记录持久化 |
| 同步状态枚举和管理逻辑 | SyncStatus / FailCount 重试机制 |
| `AttachType.LprCapturePhoto = 4` | LRP 图片类型枚举（已在移植方案中规划） |

### 4.2 从 SolidWaste 复制并修改

| 组件 | 原始（SolidWaste） | 修改为（Recycle） |
|------|-------------------|-------------------|
| 同步服务 | `SynchronizationOrderAsync` 调用 | 替换为 `IRecycleDataApi` 调用 |
| 请求 DTO | SolidWaste 上报 DTO | 新 `RecycleWeightRequest`（按 §2.2 定义） |
| 配置项 | SolidWaste 同步配置 | Recycle 同步配置（URL、端点等） |
| ABP 模块 | SolidWaste Module | Recycle Module（注册新服务） |

### 4.3 全新增

| 组件 | 说明 |
|------|------|
| `IRecycleDataApi.cs` | Refit 接口，对接 §2.2 API |
| `RecycleWeightRequest.cs` | §2.2 请求体 DTO（字段待接口文档确认） |
| `RecycleDataSyncService.cs` | 核心同步服务实现 |
| `RecycleSyncOptions.cs` | 配置模型 |

---

## 5. 证据标注

| 结论 | 证据来源 | 可信度 |
|------|---------|--------|
| SolidWaste 功能集推断 | MaterialClient 通用能力 + 任务描述「功能与 SolidWaste 一致」 | 中（无直接代码证据） |
| 授权方式（非 JWT） | EPIC 明确声明 5010 不走 JWT | 高 |
| AccessCode/MachineCode 分列 | `01-解决方案.md` §3 表格 | 高 |
| 唯一差异是数据上报接口 | 任务描述明确声明 | 高（直接来自任务） |
| WeighingMode 枚举存在 | **无 Vault 证据** | 低 |
| WeighingMode = 301 的含义 | **无 Vault 证据** | 低（仅来自任务描述） |
| SolidWaste 使用 SynchronizationOrderAsync | **无 Vault 证据** | 低（仅来自任务描述） |
