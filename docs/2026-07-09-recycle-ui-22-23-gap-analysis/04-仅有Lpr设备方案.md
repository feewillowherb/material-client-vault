# 无 CameraConfigs 时的 LPR → UnmatchedEntryPhoto 方案

> **日期**：2026-07-09  
> **状态**：方案建议（待 OpenSpec / 实现）  
> **关联**：[00-调研总览](./00-调研总览.md)、[02-字段对照-23收料.md](./02-字段对照-23收料.md)（`inPhoto` / `outPhotos`）

---

## 方案摘要

**当系统设置中不存在海康称重相机（`CameraConfigsJson` 为空 / `CameraConfigs` 无设备）时，在创建 `AttachType.Lpr` 附件的同时，使用同一张 LPR 图片再创建一条 `AttachType.UnmatchedEntryPhoto`，二者共享相同的磁盘路径（`LocalPath`）。**

本规则适用于 **MaterialClient 客户端的全部称重模式**（`Standard`、`SolidWaste`、`Recycle` 等），由**是否配置 CameraConfigs** 决定，而非仅由 `WeighingMode.Recycle` 决定。Recycle（5020）现场是典型的「无 CameraConfigs、仅有 LPR」场景，但 SolidWaste / Standard 在无相机配置时同样受益。

---

## 触发条件

| 条件 | 行为 |
|------|------|
| `SettingsEntity.CameraConfigs` **为空**（`CameraConfigsJson` 无设备） | ✅ 启用本方案：LPR 落盘 + 双附件（`Lpr` + `UnmatchedEntryPhoto`） |
| `CameraConfigs` **非空**（已配置 HikCamera） | ❌ 不启用：沿用现有链路（Hik 抓拍 → `UnmatchedEntryPhoto` / `UrbanPhoto`；Urban 另存 `Lpr`） |

判定来源：`SettingsService.GetSettingsAsync()` → `settings.CameraConfigs`（反序列化自 `CameraConfigsJson`）。

`WeighingCaptureService.CaptureAllCamerasAsync` 在 `cameraConfigs.Count == 0` 时已直接返回空列表，无法产生海康抓拍路径——与本方案的触发条件一致。

---

## 设备环境（典型场景）

| 场景 | CameraConfigs | LPR | 成像来源 |
|------|---------------|-----|----------|
| Urban 城管 | 通常有 | 有 | Hik 抓拍 + LPR |
| SolidWaste 主程序 | 可能有 / 可能无 | 可能有 | Hik 或仅 LPR |
| **Recycle（5020）** | **通常无** | **有（唯一）** | **仅 LPR** |

无 CameraConfigs 时**不能依赖** `UrbanPhoto`、海康 `EntryPhoto` / `ExitPhoto` 抓拍链路；操作员可见照片须来自 LPR，并通过 `UnmatchedEntryPhoto` 进入现有 PhotoGrid。

---

## 问题现象

操作员在称重界面看不到 LPR 抓拍图，或选中记录后 `PhotoGridView` 进/出场格均为空。Recycle 现场尤为明显（仅 LPR、无 HikCamera）。

---

## 根因分析

### 1. 磁盘层：非 UrbanMode 下 LPR 可能未落盘

`HikvisionLprService.TrySaveLprAttachment` / `VzvisionLprService.TrySaveVzLprAttachment` 在 `_cachedWeighingMode != WeighingMode.UrbanMode` 时返回 null，不写入 `Lpr/` 目录。

Standard / SolidWaste / Recycle 下即使有 LPR 回调，**也可能无文件**。

### 2. 持久化层：称重记录未挂接 LPR，且无 UnmatchedEntryPhoto 兜底

`WeighingRecordService.CreateWeighingRecordAsync`：

- `SaveCapturePhotosAsync(photoPaths)`：依赖 Hik 抓拍；无 CameraConfigs 时 `photoPaths` 为空。
- `SaveLprAttachmentAsync`：**仅在** `WeighingMode.UrbanMode` 时调用，且只写 `AttachType.Lpr`，**不写** `UnmatchedEntryPhoto`。

### 3. 展示层：UI 不展示 `AttachType.Lpr`

`PhotoGridViewModel.LoadFromListItemAsync` 只渲染 `EntryPhoto`、`UnmatchedEntryPhoto`（进场）、`ExitPhoto`（出场），**不展示** `Lpr`。

因此必须在无相机场景下为 LPR **额外**创建 `UnmatchedEntryPhoto`，PhotoGrid 才能显示。

---

## 推荐方案：无 CameraConfigs 时 LPR 双附件 + 同路径

### 核心约定

```
判定：settings.CameraConfigs.Count == 0

LPR 回调 → 落盘 Lpr/xxx.jpg
创建称重记录后：
    AttachmentFile #1  AttachType = Lpr
    AttachmentFile #2  AttachType = UnmatchedEntryPhoto
    二者 LocalPath 相同（同一 JPG）
    各自 WeighingRecordAttachment → 同一 weighingRecordId
```

**有 CameraConfigs 时**：保持现状，不因 LPR 自动创建 `UnmatchedEntryPhoto`（避免与海康抓拍重复）。

### 为何选 `UnmatchedEntryPhoto`

1. **UI 已支持**：`PhotoGridViewModel` 将 `UnmatchedEntryPhoto` 填入进场格。
2. **匹配链路已支持**：`WeighingMatchingService.CopyAttachmentsToWaybillAsync` 匹配时将 `UnmatchedEntryPhoto` 升级为 `EntryPhoto` 或 `ExitPhoto`。
3. **与 SolidWaste 语义一致**：无 Hik 时本就用 `UnmatchedEntryPhoto` 承载未匹配榜单照片。
4. **保留 `Lpr` 类型**：市平台同步（`RecycleDataSyncService`）、Urban LPR 替换等仍可通过 `AttachType.Lpr` 识别。

### 实现要点（MaterialClient.Common）

| 步骤 | 改动位置 | 说明 |
|------|----------|------|
| A | `HikvisionLprService` / `VzvisionLprService` | 当 **无 CameraConfigs** 时允许 LPR 落盘（不限于 UrbanMode）；或落盘逻辑上移到不依赖模式的公共路径 |
| B | `WeighingRecordService.SaveLprAttachmentAsync` | 增加参数或内部读取 settings：**无 CameraConfigs 时** 除 `Lpr` 外再插入 `UnmatchedEntryPhoto`（相同 `relativePath`） |
| C | `WeighingRecordService.CreateWeighingRecordAsync` | 有 LPR 路径且（UrbanMode **或** 无 CameraConfigs）时调用 `SaveLprAttachmentAsync` |
| D | 匹配后 | `CopyAttachmentsToWaybillAsync` 已有 UnmatchedEntryPhoto → Entry/Exit 改型，一般无需改 |

**注意**：两条 `AttachmentFile` 共享路径时，LPR 替换/删除须同步处理两条记录。

### 收/发料与进/出场格

- **未匹配前**：LPR 图经 `UnmatchedEntryPhoto` 显示在 PhotoGrid「进场」侧。
- **匹配后**：进场记录 → `EntryPhoto`；出场记录 → `ExitPhoto`。
- 市平台 §2.2 / §2.3 字段语义见 [01-字段对照-22发料.md](./01-字段对照-22发料.md)、[02-字段对照-23收料.md](./02-字段对照-23收料.md)。

---

## 数据流对比

### 有 CameraConfigs（不变）

```
Hik 抓拍 → photoPaths → UnmatchedEntryPhoto / UrbanPhoto
LPR（Urban）→ Lpr/
匹配 → EntryPhoto / ExitPhoto
PhotoGrid → Entry / Exit
```

### 无 CameraConfigs（本方案）

```
LPR 回调 → 落盘 Lpr/xxx.jpg
         → SaveLprAttachmentAsync
              ├── AttachType.Lpr
              └── AttachType.UnmatchedEntryPhoto（同路径）
PhotoGrid → 进场格可见（UnmatchedEntryPhoto）
匹配后 → EntryPhoto / ExitPhoto
同步（Recycle）→ 仍可扫 AttachType.Lpr
```

### 当前缺口（未实现本方案时）

```
无 CameraConfigs → CaptureAllCamerasAsync 返回空
非 UrbanMode → LPR 不落盘
SaveLprAttachmentAsync 不调用 / 无双附件
PhotoGrid → 空
```

---

## 适用范围说明

| 客户端 / 模式 | 是否适用 |
|---------------|----------|
| MaterialClient 主程序 · `Standard` | ✅ 无 CameraConfigs 时 |
| MaterialClient 主程序 · `SolidWaste` | ✅ 无 CameraConfigs 时 |
| `MaterialClient.Recycle` · `Recycle` | ✅ 典型场景（通常无 CameraConfigs） |
| `MaterialClient.Urban` · `UrbanMode` | ⚠️ 通常有 CameraConfigs；若 Urban 现场也无相机，同样适用本方案 |

实现应放在 **MaterialClient.Common**（`WeighingRecordService`、LPR 服务），由各 exe（主程序、Recycle、Urban）共用，避免仅在 Recycle 项目重复逻辑。

---

## 中长期：Recycle 独立照片 View（可选）

双附件方案是**最小改动**。Recycle 若长期无四宫格 / 票据拍照需求，可另做 `RecyclePhotoView`、隐藏 `BillPhoto` / Hik 预览等 UI 分支；与本文方案不互斥。

---

## 与市平台上报的关系

`RecycleDataSyncService` 已筛选 `ExitPhoto`、`Lpr`、`UrbanPhoto` 作为 `outPhotos` 来源。

本方案保证：**UI 可见（UnmatchedEntryPhoto）+ 同步可扫（Lpr）+ 匹配后可升级为 Entry/Exit**。

---

## 验收建议

1. **前置**：`CameraConfigsJson` 为空，已配置 LPR。
2. LPR 识别后 `Lpr/` 目录出现 jpg（Standard / SolidWaste / Recycle 均可测）。
3. 创建称重记录后，DB 存在 `Lpr` + `UnmatchedEntryPhoto`，`LocalPath` 相同。
4. `PhotoGridView` 进场格显示该图。
5. 进出场匹配后，运单为 `EntryPhoto` / `ExitPhoto`。
6. （Recycle）§2.2 同步 `outPhotos` Base64 非空。

**负向**：配置 CameraConfigs 后，LPR **不应**再自动创建 `UnmatchedEntryPhoto`（除非产品另有要求）。

---

## 关键代码索引

| 层级 | 路径 |
|------|------|
| CameraConfigs 存储 | `MaterialClient.Common/Entities/SettingsEntity.cs` → `CameraConfigsJson` |
| 无相机时抓拍为空 | `MaterialClient.Common/Services/AttendedWeighing/WeighingCaptureService.cs` |
| LPR 落盘（Hik） | `MaterialClient.Common/Services/Hikvision/HikvisionLprService.cs` |
| LPR 落盘（Vz） | `MaterialClient.Common/Services/Vzvision/VzvisionLprService.cs` |
| 附件保存 | `MaterialClient.Common/Services/AttendedWeighing/WeighingRecordService.cs` |
| 匹配改型 | `MaterialClient.Common/Services/WeighingMatchingService.cs` → `CopyAttachmentsToWaybillAsync` |
| 照片网格 VM | `MaterialClient.AttendedWeighing/ViewModels/PhotoGridViewModel.cs` |
| §2.2 同步取图 | `MaterialClient.Recycle/Services/RecycleDataSyncService.cs` |
