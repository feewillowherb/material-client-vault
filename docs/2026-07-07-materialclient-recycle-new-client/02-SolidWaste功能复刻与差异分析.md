# 02 - SolidWaste 功能复刻与差异分析

> **注意**：SolidWaste（ProductCode 5010）**不是独立客户端**，而是 MaterialClient 主程序内的 `WeighingMode.SolidWaste = 1` 业务模式。以下分析基于 MaterialMonospec 源码和 OpenSpec 规范。

---

## 1. SolidWaste 功能分析

### 1.1 SolidWaste 的功能集

SolidWaste 模式（WeighingMode=1）在 MaterialClient 主程序内运行，通过模式切换区分行为。基于源码和规范，SolidWaste 模式包含以下功能：

| 功能模块 | 功能描述 | 复用基础 |
|---------|---------|---------|
| **称重数据采集** | 串口通信连接地磅，实时读取重量数据 | `MaterialClient.Common/Services/` 串口服务 |
| **重量稳定性检测** | 时间窗口分析（3000ms），判定重量稳定 | `TruckScaleWeightService`（Common 层） |
| **车牌识别** | 海康威视 SDK 集成，被动识别 + 主动抓拍 | `HikvisionLprService`（Common 层） |
| **图片保存** | LRP 抓拍图片保存为 `AttachmentFile`（`AttachType.LrpCapturePhoto = 4`） | `AttachmentService` + `PathManager`（Common 层） |
| **称重记录持久化** | 稳定后写入本地 SQLite（`WeighingRecord` 实体） | Common 层 EF Core |
| **数据上报** | 运单匹配后通过 `SynchronizationOrderAsync` 上报（POST `/api/Order/SynchronizationOrder`） | `WeighingMatchingService.SyncNewWaybillAsync()`（Common 层） |
| **同步状态管理** | 运单 PushCompleted / SetPendingSync，失败重试 | `WeighingMatchingService`（Common 层） |
| **附件同步** | 称重关联的图片通过 `UpdateAttachesAsync` 随订单一并上报 | `AttachmentService`（Common 层） |
| **授权验证** | `SendAuthLicense` / `DownloadAuth`（非 JWT） | Common 层授权服务 |
| **设备在线状态** | 本地检测地磅、相机等设备在线状态 | `LprDeviceOnlineStatusService`（Common 层） |
| **SignalR 通信** | 与服务端实时通信（状态同步、JWT 验签等） | `DeviceStatusSignalRClient`（Common 层） |
| **固废信息字段** | `SolidWasteType`、`Street`、`SolidWasteOrderNumber`、`Shipper` | `SolidWasteInfoDto` + `SynchronizationOrderInputDto.SolidWasteInfo` |
| **固废 Excel 导出** | 导出固废模式已完成运单为 .xlsx | `SolidWasteService.GetExportRowsAsync()` |

### 1.2 SolidWaste 的上报机制（已确认）

SolidWaste 模式的数据上报通过共享的 `WeighingMatchingService` 完成，核心链路：

```
PushWaybillAsync()
  → PushNewWaybillsAsync()          // 查找未同步运单
    → SyncNewWaybillAsync()         // 逐条同步
      → SynchronizationOrderInputDto.FromWaybill(waybill)  // 转换 DTO
        → if (waybill.WeighingMode == WeighingMode.SolidWaste)
             dto.WeighingMode = 1
             dto.SolidWasteInfo = new SolidWasteInfoDto { ... }
      → _materialPlatformApi.SynchronizationOrderAsync(dto)  // Refit 调用
        → POST /api/Order/SynchronizationOrder
      → 成功: waybill.PushCompleted(DateTime.Now)
      → 失败: waybill.SetPendingSync()  // 标记待重试
```

**错误处理**：
- API 返回 `result.Success == false`：标记 `SetPendingSync()`，下次重试
- 抛出异常：日志记录，不计数（网络异常）
- 批量处理统计：记录成功/失败数量

> **来源**：MaterialMonospec `MaterialClient.Common/Services/WeighingMatchingService.cs`（行 908-1328）

### 1.3 SolidWaste 与 Urban 的对比

| 维度 | SolidWaste (5010, WM=1) | MaterialClient.Urban (5030, WM=201) | Recycle (5020, WM=301) |
|------|--------------------------|------------------------------------|-------------------------|
| **客户端形式** | 主程序内模式切换 | 独立 Avalonia 桌面端 | **独立 Avalonia 桌面端** |
| **授权方式** | SendAuthLicense / DownloadAuth（非 JWT） | JWT（`.urban` + 在线激活） | **同 SolidWaste** |
| **数据上报接口** | `SynchronizationOrderAsync`（POST `/api/Order/SynchronizationOrder`） | `IUrbanManagementApi`（Refit） | **资源化利用厂接口 §2.2**（HMAC-SHA256） |
| **数据中转** | 直连 MaterialPlatform 服务端 | 通过 UrbanManagement | **直连资源化利用厂 API** |
| **AccessCode** | 有（接入码分列） | 有（`LicenseInfo.AccessCode`） | **有** |
| **MachineCode** | 有（设备码） | 有（JWT claim `machineCode`） | **有** |
| **政府平台出站** | 否（上报 MaterialPlatform） | 是（`GovSyncBackgroundWorker`） | **否** |

---

## 2. Recycle 与 SolidWaste 的差异点

### 2.1 核心差异：数据上报接口

| 维度 | SolidWaste (WM=1) | Recycle (WM=301) |
|------|-------------------|------------------|
| **上报接口** | `IMaterialPlatformApi.SynchronizationOrderAsync`（POST `/api/Order/SynchronizationOrder`） | POST `/dataCenter/resourcePlace/productTransportRecord/v1/addBatch` |
| **上报方向** | 客户端 → 内部服务端（MaterialPlatform） | 客户端 → **外部第三方**（资源化利用厂） |
| **接口规范** | 内部 Refit 接口，`SynchronizationOrderInputDto` 已定义 | **外部文档约束**，需严格遵循 §2.2 字段、格式 |
| **认证方式** | 内部认证（LicenseInfo / Refit 默认） | **HMAC-SHA256 签名**（accessKey + secretKey） |
| **请求体结构** | 单条 JSON Object | **JSON Array**（批量提交） |
| **图片编码** | 附件先上传 OSS，再推送元数据 | **Base64 内嵌**，不带标识头，逗号分隔 |
| **重量单位** | kg（本地称重记录） | **吨**（需单位转换：÷1000） |

### 2.2 SolidWaste DTO 字段与 §2.2 接口字段映射

| §2.2 字段 | 类型 | 必填 | SolidWaste DTO 对应字段 | 映射说明 |
|----------|------|------|----------------------|---------|
| `dataNo` | String | ✅ | 无直接对应 | 需生成唯一标识（可用 OrderNo 或时间戳组合） |
| `pointNumber` | String | ✅ | 无 | 需配置（资源化利用厂唯一标识） |
| `carNo` | String | ✅ | `TruckNo` | 车牌号 |
| `carrierCompanyName` | String | ❌ | 无 | 可选 |
| `productName` | String | ✅ | 无 | 需配置或从称重记录映射 |
| `netWeight` | BigDecimal | ✅ | `OrderGoodsWeight` | 净重，**需 ÷1000 转吨** |
| `tareWeight` | BigDecimal | ❌ | `OrderTruckWeight` | 皮重，**需 ÷1000 转吨** |
| `grossWeight` | BigDecimal | ❌ | `OrderTotalWeight` | 毛重，**需 ÷1000 转吨** |
| `unitPrice` | BigDecimal | ❌ | 无 | 可选 |
| `payAmount` | BigDecimal | ❌ | 无 | 可选 |
| `outTime` | string | ✅ | `OutTime` | 出场时间，格式 `yyyy-MM-dd HH:mm:ss` |
| `outPhotos` | string | ✅ | 关联 AttachmentFile | **Base64 编码，不带标识头，逗号分隔** |
| `saleContractNo` | string | ❌ | 无 | 可选 |
| `consignee` | string | ❌ | 无 | 可选（收货后再次调用时必传） |
| `consigneeAddress` | string | ❌ | 无 | 可选 |
| `receivingTime` | string | ❌ | 无 | 可选 |
| `receivingProof` | string | ❌ | 无 | 可选（Base64，同 outPhotos 格式） |

> **来源**：§2.2 字段来自 MaterialMonospec `docs/SyncDoc/杭州市资源化利用厂数据接入接口V1.0.md`。

### 2.3 其它维度完全一致

任务明确声明「所有客户端功能与 SolidWaste 一致」，因此除数据上报接口外，以下模块应完全复用：

- 称重数据采集（串口通信、稳定性检测）
- 车牌识别（海康威视 SDK）
- 图片保存（`AttachmentFile` 体系）
- 称重记录持久化
- 授权验证（非 JWT）
- 设备在线检测
- UI 交互（Avalonia 界面）

### 2.4 推断的 Recycle 模块结构

```
MaterialClient.Recycle/
├── MaterialClientRecycleModule.cs     ← ABP 模块注册
├── Api/
│   └── IRecycleDataApi.cs              ← Refit 接口（对接 §2.2）
├── Models/
│   ├── RecycleSyncOptions.cs           ← 配置模型（含 accessKey/secretKey/pointNumber）
│   └── RecycleTransportRecord.cs       ← §2.2 请求体 DTO（字段已确认）
├── Services/
│   ├── RecycleDataSyncService.cs       ← 核心同步服务（替代 SynchronizationOrderAsync）
│   ├── RecycleHmacSignService.cs       ← HMAC-SHA256 签名服务
│   └── RecycleWeightMapper.cs         ← WeighingRecord → §2.2 字段映射
└── Configuration/
    └── RecycleDefaults.cs             ← 默认配置值
```

---

## 3. WeighingMode = 301 分析

### 3.1 WeighingMode 枚举（已确认）

| 枚举成员 | 值 | 含义 | ProductCode | 状态 |
|---------|---|------|-------------|------|
| `Standard` | 0 | 标准物料验收模式 | 5000 | ✅ 已存在 |
| `SolidWaste` | 1 | 固废称重模式 | 5010 | ✅ 已存在 |
| `UrbanMode` | 201 | 城管地磅模式 | 5030 | ✅ 已存在 |
| **`Recycle`** | **301** | **资源化利用厂模式** | **5020** | ❌ **需新增** |

> **来源**：MaterialMonospec `MaterialClient.Common/Entities/Enums/WeighingMode.cs`。

### 3.2 WeighingMode 影响的维度

根据源码分析：

| 维度 | 影响内容 | 源码证据 |
|------|---------|---------|
| **DTO 转换** | `SynchronizationOrderInputDto.FromWaybill()` 根据 `WeighingMode.SolidWaste` 填充固废信息 | `solid-waste-order-sync` spec |
| **产品码映射** | `ISettingsService.GetProductCodeAsync()` 将 WeighingMode 映射到 ProductCode | `system-configuration` spec |
| **启动模块选择** | 根据 ProductCode/WeighingMode 加载对应 ABP 模块 | `materialclient-urban-desktop` spec |
| **Excel 导出可见性** | 固废模式才显示"导出"按钮 | `export-button-ui` spec |
| **详情 ViewModel 分支** | 根据 WeighingMode 选择不同的 DetailViewModel | `detail-viewmodel-hierarchy` spec |
| **城市称重扩展** | UrbanMode 专用 `UrbanWeighingExtension` 实体 | `urban-weighing-extension` spec |

### 3.3 Recycle=301 需要的代码改动

由于 WeighingMode 枚举当前只有 3 个成员，新增 Recycle=301 需要同步修改：

1. **`WeighingMode.cs`**：新增 `Recycle = 301` 枚举成员
2. **`ProductCode.cs`**：新增 `Recycle = 5020` 枚举成员
3. **`ISettingsService`**：新增 `Recycle` → `ProductCode.Recycle` 映射（参考 Urban 的映射逻辑）
4. **`SaveDefaultWeighingModeAsync`**：新增 `ProductCode.Recycle` → `WeighingMode.Recycle` 映射

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
| `ISettingsService` | 配置管理（需扩展映射） |
| `TruckScaleWeightService` | 称重服务（串口 + 稳定性检测） |
| `WeighingRecord` 实体 | 称重记录持久化 |
| `LicenseInfo` 实体 | 授权信息 |
| `AttachType.LprCapturePhoto = 4` | LRP 图片类型枚举 |

### 4.2 从 SolidWaste 模式复制并修改

| 组件 | 原始（SolidWaste WM=1） | 修改为（Recycle WM=301） |
|------|------------------------|------------------------|
| 同步服务 | `WeighingMatchingService.SyncNewWaybillAsync()` → `SynchronizationOrderAsync` | 新 `RecycleDataSyncService` → `IRecycleDataApi` |
| 请求 DTO | `SynchronizationOrderInputDto`（单条 JSON Object） | `RecycleTransportRecord`（JSON Array，按 §2.2 定义） |
| 图片处理 | 附件先上传 OSS，再推送元数据 | **Base64 内嵌到请求体**（不带标识头，逗号分隔） |
| 重量单位 | kg（`OrderGoodsWeight`） | **吨**（÷1000 转换） |
| 认证 | 内部认证 | **HMAC-SHA256 签名** |
| 配置项 | SolidWaste 同步配置 | Recycle 同步配置（URL、accessKey、secretKey、pointNumber） |

### 4.3 全新增

| 组件 | 说明 |
|------|------|
| `IRecycleDataApi.cs` | Refit 接口，对接 §2.2（POST `/dataCenter/resourcePlace/productTransportRecord/v1/addBatch`） |
| `RecycleTransportRecord.cs` | §2.2 请求体 DTO（字段已确认，见 §2.2 映射表） |
| `RecycleHmacSignService.cs` | HMAC-SHA256 签名服务（计算签名 + 构造认证 Header） |
| `RecycleDataSyncService.cs` | 核心同步服务实现 |
| `RecycleWeightMapper.cs` | WeighingRecord → RecycleTransportRecord 字段映射（含 kg→吨转换） |
| `RecycleSyncOptions.cs` | 配置模型（含 accessKey/secretKey/pointNumber） |

---

## 5. 证据标注

| 结论 | 证据来源 | 可信度 |
|------|---------|--------|
| SolidWaste 功能集 | MaterialMonospec 源码 + OpenSpec 规范 | **高（源码 + 规范确认）** |
| SolidWaste 不是独立客户端，是 WeighingMode=1 模式 | MaterialMonospec 源码 + OpenSpec 规范 | **高（源码确认）** |
| `SynchronizationOrderAsync` 签名和调用链 | MaterialMonospec `WeighingMatchingService.cs` + `IMaterialPlatformApi.cs` | **高（源码确认）** |
| SolidWaste DTO 结构 | `SynchronizationOrderInputDto.cs`（含 SolidWasteInfo 扩展） | **高（源码确认）** |
| 授权方式（非 JWT） | EPIC 明确声明 5010 不走 JWT | 高 |
| AccessCode/MachineCode 分列 | `01-解决方案.md` §3 表格 | 高 |
| 唯一差异是数据上报接口 | 任务描述明确声明 | 高（直接来自任务） |
| WeighingMode 枚举（Standard=0, SolidWaste=1, UrbanMode=201） | MaterialMonospec `WeighingMode.cs` | **高（源码确认）** |
| WeighingMode = 301 不存在，需新增 | MaterialMonospec 源码搜索确认 | **高（确认不存在）** |
| §2.2 接口字段和认证方式 | MaterialMonospec `docs/SyncDoc/杭州市资源化利用厂数据接入接口V1.0.md` | **高（文档确认）** |
