# Recycle 独立表单页面方案

> **日期**：2026-07-09  
> **状态**：UI 需求（待 OpenSpec / 实现）  
> **关联**：[00-调研总览](./00-调研总览.md)、[04-仅有Lpr设备方案.md](./04-仅有Lpr设备方案.md)

---

## 需求摘要

Recycle 客户端（5020，`WeighingMode.Recycle`）**不应继续直接复用** `SolidWasteModeFormView`，需要**单独的表单页面**（建议命名 `RecycleModeFormView` + 对应 ViewModel，如 `RecycleWeighingDetailViewModel`）。

- **布局**：与 SolidWaste 表单**保持一致**（同样的 Grid 标签列宽、间距、边框样式、`SearchableSelectionBox` 交互）。
- **字段**：在 SolidWaste 基础上**删除** Recycle 不使用的三项，其余保留。

---

## 当前实现 vs 目标

| 项 | 当前 | 目标 |
|----|------|------|
| 表单 XAML | `SolidWasteModeFormView.axaml` | **`RecycleModeFormView.axaml`（新建）** |
| 详情 VM | `SolidWasteWeighingDetailViewModel`（Recycle 共用） | **`RecycleWeighingDetailViewModel`（新建或拆分）** |
| DataTemplate | `AttendedWeighingDetailView` 中 SolidWaste 模板 | 增加 `RecycleWeighingDetailViewModel` → `RecycleModeFormView` |
| 完成校验 | 强制联单、镇街（SolidWaste 规则） | **不校验**已删除的三字段 |

`AttendedWeighingViewModel` 中已有 `WeighingMode.Recycle` 分支指向 SolidWaste VM，实现时应改为挂载 Recycle 专用 VM / View。

---

## 字段对照（SolidWaste → Recycle）

### 保留字段（布局与 SolidWaste 相同）

| 序号 | 标签 | 绑定 / 控件 | 说明 |
|------|------|-------------|------|
| 1 | 称重类型 | `DeliveryTypeOptions` / `SelectedDeliveryType` | 收料 / 发料；Waybill 只读展示 |
| 2 | 车牌号 | `PlateNumber` | TextBox |
| 3 | 供应商 / 发货单位 / 收货单位 | `ProviderLabelText` + `SearchableSelectionBox` | 随 `DeliveryType` 切换标签 |
| 4 | 材料名称 | `SearchableSelectionBox` → `Material.Name` | 映射市平台 `productName` / `materialName` |
| 5 | 备注 | `Remark` | TextBox |

父级 **`AttendedWeighingDetailView`**（毛/皮/净重、进/出场时间）与 **`AttendedWeighingWindow`**（收/发切换、PhotoGrid）**不变**，仍与 SolidWaste 共用；仅**表单区**独立。

### 删除字段（Recycle 不使用）

| 标签 | SolidWaste 绑定 | 存储（ExtraProperties） | Recycle 处理 |
|------|-----------------|-------------------------|--------------|
| **联单编号** | `SolidWasteOrderNumber` | `SolidWasteInfo.SolidWasteOrderNumber` | ❌ **不展示、不录入、完成时不校验** |
| **所属镇街** | `SelectedStreetItem` / `SelectedStreet` | `SolidWasteInfo.Street` | ❌ 同上 |
| **类型选择** | `SelectedSolidWasteType` | `SolidWasteInfo.SolidWasteType` | ❌ 同上 |

上述三项为 SolidWaste / 内部导出业务字段，**市平台 §2.2 / §2.3 接口不要求**（见 [01-字段对照-22发料.md](./01-字段对照-22发料.md)）。Recycle 上报 `dataNo` 等业务唯一号由同步层从 `OrderNo` 等生成，**不依赖联单编号 UI**。

---

## 布局示意

与 `SolidWasteModeFormView` 相同的外壳：

```
┌─ Border (#B0EBFF 顶边) ─────────────────────┐
│  称重类型    [收料 ▼]                          │
│  车牌号      [________]                        │
│  发货单位    [SearchableSelectionBox]          │
│  材料名称    [SearchableSelectionBox]          │
│  备注        [________]                        │
└──────────────────────────────────────────────┘
┌─ 下方空白区（与 SolidWaste 同 RowDefinition）─┐
└──────────────────────────────────────────────┘
```

**不包含**：联单编号、所属镇街、类型选择 三行 Grid。

---

## ViewModel 行为差异

`RecycleWeighingDetailViewModel` 建议相对 `SolidWasteWeighingDetailViewModel`：

| 行为 | SolidWaste | Recycle |
|------|------------|---------|
| `LoadModeSpecificDataAsync` | 加载 Streets、SolidWasteTypes 配置 | **跳过**镇街/类型配置 |
| `SaveModeSpecificAsync` | 写入联单、镇街、类型 | **不写入**三项 ExtraProperties |
| `CompleteModeSpecificAsync` | 校验供应商、材料、**镇街、联单** | 校验供应商、材料；**不要求**镇街/联单/类型 |
| `IsSolidWasteMode` | `true` | 改为 `IsRecycleMode` 或独立标志，避免误走 SolidWaste 导出逻辑 |

保存/完成仍通过 `IWeighingMatchingService.UpdateSolidWasteModeAsync` 时，Recycle VM 应对 SolidWaste 专用参数传 `null`，或后续提案增加 `UpdateRecycleModeAsync` 专用入口（实现阶段再定）。

---

## 与 AttendedWeighingDetailView 集成

`AttendedWeighingDetailView.axaml` 当前 DataTemplate：

```xml
<DataTemplate DataType="vm:SolidWasteWeighingDetailViewModel">
    <aw:SolidWasteModeFormView />
</DataTemplate>
```

目标增加：

```xml
<DataTemplate DataType="vm:RecycleWeighingDetailViewModel">
    <aw:RecycleModeFormView />
</DataTemplate>
```

`AttendedWeighingViewModel` 创建详情 VM 处（约 `WeighingMode.Recycle` 分支）由 `SolidWasteWeighingDetailViewModel` 改为 `RecycleWeighingDetailViewModel`。

---

## 实现任务清单（参考）

1. 复制 `SolidWasteModeFormView.axaml` → `RecycleModeFormView.axaml`，删除三字段行。
2. 新建 `RecycleWeighingDetailViewModel`（继承 `AttendedWeighingDetailViewModelBase` 或从 SolidWaste VM 抽取共享基类）。
3. 注册 DI；更新 `AttendedWeighingViewModel` Recycle 分支。
4. 调整 `CompleteModeSpecificAsync` 去掉联单/镇街/类型校验。
5. （可选）Recycle 主窗口隐藏 BillPhoto / 无 CameraConfigs 时依赖 [04-仅有Lpr设备方案.md](./04-仅有Lpr设备方案.md)。

---

## 验收建议

1. Recycle 客户端详情区显示 `RecycleModeFormView`，**无**联单、镇街、类型三行。
2. 完成收货/发货时，未填联单、镇街仍可成功完成（供应商、材料仍必填）。
3. SolidWaste 客户端仍使用原 `SolidWasteModeFormView`，三字段**不受影响**。
4. 布局（列宽 72、Spacing 6、顶边色 `#B0EBFF`）与 SolidWaste 视觉一致。

---

## 关键代码索引

| 用途 | 路径 |
|------|------|
| SolidWaste 表单（复制基准） | `MaterialClient.AttendedWeighing/Views/Controls/SolidWasteModeFormView.axaml` |
| 详情模板切换 | `MaterialClient.AttendedWeighing/Views/Controls/AttendedWeighingDetailView.axaml` |
| SolidWaste VM（行为参考） | `MaterialClient.AttendedWeighing/ViewModels/SolidWasteWeighingDetailViewModel.cs` |
| Recycle 分支 | `MaterialClient.AttendedWeighing/ViewModels/AttendedWeighingViewModel.cs` |
