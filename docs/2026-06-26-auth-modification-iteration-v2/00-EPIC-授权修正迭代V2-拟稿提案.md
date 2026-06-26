# 授权修正迭代 V2 — EPIC 拟稿提案

> **文档类型**：EPIC Draft Proposal（拟稿提案）
> **创建日期**：2026-06-26
> **状态**：拟稿
> **迭代版本**：V2（功能修正）
> **范围**：`MaterialClient.Urban` + `MaterialClient.Common` + `UrbanManagement` + `BasePlatform`（仅 ProductCode 5001）
> **前置基线**：
> - [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)（V1 EPIC）
> - [07-客户端服务端EPIC缺口对齐拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md)（V1 缺口对齐）
> - [06-MaterialClient.Urban迁移拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/06-MaterialClient.Urban迁移拟稿提案.md)（客户端迁移）

---

## 0. 文档定位

| 维度 | 说明 |
|------|------|
| **迭代性质** | 基于 V1 EPIC 实施反馈的**功能修正**迭代，非新增架构 |
| **与 V1 的关系** | 不推翻 V1 设计决策（JWT 唯一权威、仅 5001、iss=BasePlatform），在 V1 基础上做**裁剪、补充和规范修正** |
| **驱动来源** | 运营与开发团队在 V1 实施/联调阶段发现的功能性问题 |

---

## 1. 提案摘要

| # | 修正主题 | 归属项目 | 优先级 | 摘要 |
|---|---------|---------|--------|------|
| **F1** | 客户端改为纯在线验证，隐藏离线 UI | MaterialClient.Urban · BasePlatform | **P0** | 离线验证业务逻辑代码保留（`StaticLicenseChecker`、`.urban` bootstrap）；离线导入 UI 删除；BasePlatform 下载授权功能 UI 隐藏 |
| **F2** | 称重记录新增提交时机器码字段 | UrbanManagement | **P1** | `UrbanWeighingExtension`、`UrbanWeighingRecord` 新增 `SubmitMachineCode` 字段，提交数据时写入 |
| **F3** | ABP 审计字段改由框架控制 | UrbanManagement · MaterialClient | **P1** | 移除领域方法中手动赋值 `CreationTime`/`CreatorUserId`/`LastModificationTime`/`LastModifierUserId` 等字段，改用 ABP 拦截器自动填充 |
| **F4** | 签名版本控制机制（讨论稿） | BasePlatform · UrbanManagement | **P2** | 服务端存储最新签名版本号，客户端携带签名版本；旧版本签名连接时主动失效 |

---

## 2. F1 — 客户端纯在线验证 + 离线 UI 隐藏

### 2.1 背景

V1 EPIC 设计了**离线 + 在线**双路径（§2 双路径表）。运营反馈：实际部署中 5001 客户端均在有网环境运行，离线路径增加了维护复杂度且从未在生产中使用。因此决定：

- **客户端统一走在线激活**（`POST /api/urban/auth/activate`）
- **离线验证的业务逻辑代码保留**（防回退 / 应急），但**离线导入相关 UI 入口移除**
- **BasePlatform 管理后台下载授权功能 UI 隐藏**（不再向运营暴露离线入口）

### 2.2 MaterialClient.Urban 改动

#### 保留（代码层面）

| 组件 | 保留理由 |
|------|---------|
| `StaticLicenseChecker` | JWT 本地验签仍为启动门禁；离线应急可用 |
| `license.urban` 文件读取逻辑 | 代码路径保留；启动时若本地已有 JWT 仍可 bootstrap 验签 |
| `LatestJwtToken` 持久化 | 在线激活成功后仍写入 |
| SignalR `VerifyJwtAsync` / `GetClientProjectLicenseInfo` | 保留；与在线激活配合 |
| `LicenseInfo.LatestJwtToken` 字段 | 保留 |
| `LicenseInfo.MachineCode` 字段 | 保留 |

#### 删除（UI 层面）

| UI 组件 | 说明 |
|---------|------|
| 离线授权文件导入对话框 | 用户不再需要手动导入 `.urban` 文件的 UI 入口 |
| 设置页面中「离线授权」相关 UI 区域 | 导出/导入按钮、文件路径选择等 |
| 任何引导用户走离线流程的提示文案 | 如「请导入离线授权文件」 |

#### 保留但隐藏

| UI 组件 | 处理方式 |
|---------|---------|
| `UnauthorizedNoticeWindow` 中的「离线导入」按钮 | 移除或注释；仅保留「在线激活」入口 |

#### 启动流程变更

```
V1 流程：
  LatestJwtToken 有值 → 验签
  LatestJwtToken 无值 → 回退读 license.urban → 验签
  两者均无 → 弹出 UnauthorizedNoticeWindow（含离线导入 + 在线激活）

V2 流程：
  LatestJwtToken 有值 → 验签 → 通过 → 正常启动
  LatestJwtToken 无值 / 验签失败 → 弹出 UnauthorizedNoticeWindow（仅在线激活）
  license.urban bootstrap 路径：代码保留（启动时仍尝试读取），但不暴露 UI 导入入口
```

### 2.3 BasePlatform 改动

| 改动项 | 说明 |
|--------|------|
| 下载授权管理页面 UI 隐藏 | `DownloadUrbanLicense` 相关管理后台入口隐藏（CSS `display:none` / 菜单权限 / 配置开关） |
| `GET /api/auth/license-file` API | **保留**（代码不删，应急/运维路径仍可用），仅 UI 入口隐藏 |
| `GET /api/auth/license-file` 门禁 | 可考虑增加权限标记（如仅管理员角色可访问），防止普通运营误操作 |

### 2.4 影响评估

| 项目 | 影响范围 |
|------|---------|
| MaterialClient.Urban | UI 层裁剪（移除离线导入对话框及相关菜单项）；业务逻辑零改动 |
| MaterialClient.Common | 无改动 |
| UrbanManagement | 无改动 |
| BasePlatform（管理后台） | 下载授权 UI 隐藏；API 保留 |

---

## 3. F2 — 称重记录新增提交时机器码字段

### 3.1 背景

当前 `UrbanWeighingRecord` 和 `UrbanWeighingExtension` 不记录提交数据时客户端的机器码。新增 `SubmitMachineCode` 字段可在数据溯源和审计时定位数据来源设备。

### 3.2 数据模型变更

#### UrbanWeighingRecord

```csharp
public class UrbanWeighingRecord : Entity<Guid>
{
    // ... 现有字段保留 ...

    // ===== 新增 =====
    /// <summary>
    /// 提交数据时的客户端机器码。
    /// 由客户端在上传时附带，服务端不做校验，仅做记录。
    /// </summary>
    public string? SubmitMachineCode { get; set; }
}
```

#### UrbanWeighingExtension

```csharp
public class UrbanWeighingExtension
{
    // ... 现有字段保留 ...

    // ===== 新增 =====
    /// <summary>
    /// 提交数据时的客户端机器码（客户端副本）。
    /// </summary>
    public string? SubmitMachineCode { get; set; }
}
```

### 3.3 数据流

```
MaterialClient.Urban（客户端）
  ├─ 本地 UrbanWeighingExtension.SubmitMachineCode ← MachineCodeService.GetMachineCode()
  └─ 上传 DTO 新增 submitMachineCode 字段
       ↓
UrbanManagement（服务端）
  ├─ DTO → UrbanWeighingRecord.SubmitMachineCode（直接透传）
  └─ 不校验值是否与授权机器码一致（仅记录）
```

### 3.4 数据库迁移

```sql
-- UrbanManagement
ALTER TABLE UrbanWeighingRecords ADD COLUMN SubmitMachineCode NVARCHAR(128) NULL;

-- MaterialClient.Urban（SQLite）
ALTER TABLE UrbanWeighingExtensions ADD COLUMN SubmitMachineCode TEXT NULL;
```

### 3.5 涉及文件

| 项目 | 文件 | 改动 |
|------|------|------|
| UrbanManagement | `Entities/UrbanWeighingRecord.cs` | 新增 `SubmitMachineCode` |
| UrbanManagement | `EntityFrameworkCore/*DbContext*.cs` | EF Core 映射 |
| UrbanManagement | `Migrations/*` | 新 Migration |
| MaterialClient.Urban | `Entities/UrbanWeighingExtension.cs` | 新增 `SubmitMachineCode` |
| MaterialClient.Urban | `Services/UrbanServerUploadService.cs` | 上传 DTO 携带 `submitMachineCode` |
| MaterialClient.Urban | `MaterialClient.Common/Migrations/*` | 新 Migration |

---

## 4. F3 — ABP 审计字段改由框架控制

### 4.1 背景

当前 `UrbanManagement` 和 `MaterialClient` 项目中，部分实体在领域方法或服务层**手动赋值**审计字段（如 `CreationTime`、`CreatorUserId`、`LastModificationTime` 等），未充分利用 ABP 框架的自动审计填充机制。这导致：

- 审计逻辑分散，容易遗漏
- 与 ABP 规范不一致，增加维护负担
- 部分字段名不标准（如 `AddTime` 而非 `CreationTime`）

### 4.2 ABP 标准审计接口映射

| ABP 接口 | 自动填充字段 | 对应当前非标准字段 |
|----------|------------|-------------------|
| `IHasCreationTime` | `CreationTime` | `AddTime` |
| `ICreationAudited`（继承 `IHasCreationTime`） | `CreationTime` + `CreatorUserId` | `AddTime` + 无对应 |
| `IModificationAudited`（继承 `IHasModificationTime`） | `LastModificationTime` + `LastModifierUserId` | 无标准对应 |
| `ISoftDelete` | `IsDeleted` | `DeleteStatus`（如有） |
| `IDeletionAudited`（继承 `ISoftDelete`） | `DeletionTime` + `DeleterUserId` | 无标准对应 |

### 4.3 实施方案

#### 步骤一：实体接口声明

将实体改为实现 ABP 标准审计接口，由框架自动填充：

```csharp
// 改造前（示例）
public class GovProject : Entity<Guid>
{
    public DateTime? AddTime { get; set; }  // 手动赋值
}

// 改造后
public class GovProject : FullAuditedAggregateRoot<Guid>
{
    // ABP 自动管理：CreationTime, CreatorUserId,
    //              LastModificationTime, LastModifierUserId,
    //              IsDeleted, DeletionTime, DeleterUserId
}
```

> **注意**：基类选择需按实际场景调整。若仅需创建审计可用 `CreationAuditedAggregateRoot<Guid>`；若需软删除 + 完整审计用 `FullAuditedAggregateRoot<Guid>`。

#### 步骤二：移除手动赋值代码

在领域方法和应用服务中移除所有手动设置审计字段的代码：

```csharp
// 移除这类代码
entity.CreationTime = DateTime.Now;
entity.CreatorUserId = _currentUser.Id;
entity.LastModificationTime = DateTime.Now;

// 改为 ABP 自动拦截器处理（EF Core SaveChanges 拦截）
```

#### 步骤三：数据库列映射兼容

对于使用非标准字段名的实体（如 `AddTime`），通过 EF Core 映射兼容：

```csharp
// 方案 A（推荐）：新列 + 数据迁移
// 1. 添加标准列 CreationTime
// 2. 数据迁移：UPDATE SET CreationTime = AddTime WHERE CreationTime IS NULL
// 3. 后续版本移除 AddTime

// 方案 B：列名映射（过渡期）
builder.Property(e => e.CreationTime)
       .HasColumnName("AddTime");  // 映射到旧列名
```

#### 步骤四：MaterialClient 审计对齐

`MaterialClient` 使用 SQLite + EF Core，**非 ABP 宿主**。处理方式：

```csharp
// MaterialClient.Common — 自定义 SaveChangesInterceptor
public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        foreach (var entry in eventData.Context.ChangeTracker.Entries())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Property("CreationTime").CurrentValue = DateTime.Now;
            }
            if (entry.State == EntityState.Modified)
            {
                entry.Property("LastModificationTime").CurrentValue = DateTime.Now;
            }
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}
```

### 4.4 涉及实体清单

#### UrbanManagement

| 实体 | 当前审计字段 | 改造方向 |
|------|------------|---------|
| `GovProject` | `AddTime` | 实现 `IHasCreationTime`；`AddTime` → `CreationTime` |
| `GovSyncData` | 待确认 | 实现 `ICreationAudited` |
| `GovLog` | 待确认 | 实现 `ICreationAudited` |
| `UrbanWeighingRecord` | 待确认 | 实现 `ICreationAudited` + `IModificationAudited` |
| `UrbanWeighingExtension`（如有独立实体） | 待确认 | 实现 `ICreationAudited` |

#### MaterialClient

| 实体 | 当前审计字段 | 改造方向 |
|------|------------|---------|
| `LicenseInfo` | `CreatedAt`, `UpdatedAt` | 添加 `CreationTime`/`LastModificationTime` + 拦截器 |
| `UrbanWeighingExtension` | 待确认 | 同上 |

### 4.5 注意事项

- **`UrbanWeighingRecord`**：若从客户端接收数据时直接插入，`CreatorUserId` 在 UrbanManagement 侧可能无意义（API 无认证上下文）。需讨论：不使用 `ICreationAudited`（仅 `IHasCreationTime`），或在接收接口层注入系统用户 ID。
- **数据迁移**：`AddTime → CreationTime` 需要数据迁移脚本，确保历史数据不丢失。
- **JSON 序列化**：对外 API 响应中审计字段名变更需同步调整客户端映射（如有）。

---

## 5. F4 — 签名版本控制机制（讨论稿）

### 5.1 背景

当前 JWT 签名使用固定的 RSA 密钥对。若未来需要更换密钥（密钥泄露、定期轮换），已分发的旧 JWT 将继续有效直到自然过期。缺乏机制让服务端主动使旧签名失效。

### 5.2 目标

设计一种机制，使服务端（BasePlatform 或 UrbanManagement）能够：

1. **存储当前有效的签名版本标识**
2. 当持有旧版本签名的客户端尝试与服务端通信时，**主动使其失效**
3. 客户端收到失效通知后，**强制走在线激活**重新获取新签名 JWT

### 5.3 方案讨论

#### 方案 A：JWT Claim 嵌入签名版本 + 服务端校验

```
签发时：
  JWT Claims 新增 "sigVer": "v2"

验证时：
  客户端本地验签（iss/machineCode/exp）— 不变
  客户端连 Urban 时，Urban Hub 返回当前 sigVer
  客户端比对本地 JWT sigVer 与服务端 sigVer
  若不一致 → 强制在线激活
```

| 优点 | 缺点 |
|------|------|
| 实现简单，仅需新增 claim + Hub 字段 | 客户端需修改才能感知版本 |
| 完全向后兼容（旧 JWT 无 sigVer claim，视为 v1） | 旧版客户端无此逻辑，不会主动刷新 |
| 不需要服务端存储版本状态（可硬编码或配配置） | — |

#### 方案 B：服务端存储签名版本 + 连接时校验

```
BasePlatform / UrbanManagement：
  新增配置表/字段：CurrentSignatureVersion（如 "v2"）

客户端连接 UrbanManagement 时：
  Hub VerifyJwtAsync 或专用方法返回 currentSignatureVersion
  客户端比对 → 不一致则清除 LatestJwtToken → 强制激活
```

| 优点 | 缺点 |
|------|------|
| 版本由服务端统一控制 | Urban 和 BasePlatform 需同步版本号 |
| 可动态切换，无需发版客户端 | 需新增 Hub 方法或扩展现有方法 |

#### 方案 C：密钥轮换 + 撤销列表（CRL 式）

```
BasePlatform：
  维护签名密钥版本列表（key_v1, key_v2, ...）
  每个版本关联生效时间和撤销标记

JWT 验证：
  客户端本地验签保留（多公钥校验）
  服务端维护已撤销的 jti 或 sigVer 列表
```

| 优点 | 缺点 |
|------|------|
| 最灵活，支持渐进式轮换 | 实现复杂度高 |
| 可精确控制单设备失效 | 客户端需支持多公钥 |
| — | 需要额外的密钥管理基础设施 |

### 5.4 建议

> **推荐方案 B**（服务端存储 + 连接时校验），理由：
> 1. 改动量最小——仅新增 Hub 字段 + 客户端比对逻辑
> 2. 版本由服务端统一管控，无需客户端硬编码
> 3. 利用现有 SignalR 连接通道，无需新增 API
> 4. 与现有 `VerifyJwtAsync` 流程自然集成

### 5.5 待讨论项

| 讨论项 | 选项 |
|--------|------|
| 签名版本存储位置 | BasePlatform（权威源） vs UrbanManagement（代理缓存） |
| 版本号格式 | 语义版本 `v2.0` vs 整数 `2` vs 时间戳 |
| 旧签名宽限期 | 立即失效 vs N 天宽限期 |
| 离线场景 | 离线客户端无法感知版本变化——是否需要处理（F1 已隐藏离线 UI，影响较小） |
| 触发条件 | 仅密钥轮换时手动更新 vs 定期自动轮换 |

---

## 6. 实施优先级与阶段划分

### 阶段一（P0，立即）

| 编号 | 工作 | 归属 |
|------|------|------|
| F1-1 | MaterialClient.Urban 离线导入 UI 移除 | MaterialClient |
| F1-2 | BasePlatform 下载授权管理页面 UI 隐藏 | BasePlatform |
| F1-3 | UnauthorizedNoticeWindow 裁剪（仅保留在线激活） | MaterialClient |

### 阶段二（P1，短中期）

| 编号 | 工作 | 归属 |
|------|------|------|
| F2-1 | UrbanWeighingRecord / Extension 新增 `SubmitMachineCode` | UrbanManagement + MaterialClient |
| F2-2 | 上传 DTO 新增字段 + 数据库迁移 | UrbanManagement + MaterialClient |
| F3-1 | UrbanManagement 实体实现 ABP 审计接口 | UrbanManagement |
| F3-2 | 移除领域方法中手动审计赋值 | UrbanManagement |
| F3-3 | 数据库列名标准化迁移 | UrbanManagement |
| F3-4 | MaterialClient SaveChangesInterceptor | MaterialClient |

### 阶段三（P2，中远期 / 待讨论）

| 编号 | 工作 | 归属 |
|------|------|------|
| F4-1 | 签名版本方案评审与确认 | 全部 |
| F4-2 | 服务端签名版本存储实现 | BasePlatform / UrbanManagement |
| F4-3 | 客户端版本比对与强制激活逻辑 | MaterialClient |

---

## 7. 跨项目依赖与发版协调

### 依赖关系

```
F1（在线验证裁剪）
  └─ MaterialClient 可独立发版（纯 UI 裁剪）
  └─ BasePlatform 可独立发版（UI 隐藏）

F2（机器码字段）
  ├─ MaterialClient 先发版（DTO 新增字段）
  └─ UrbanManagement 同步或延后一版本（接收新字段）

F3（ABP 审计）
  ├─ UrbanManagement 可独立发版
  └─ MaterialClient 可独立发版
  └─ 注意：若 UrbanManagement 对外 API 响应中审计字段名变更，需与 MaterialClient 联调

F4（签名版本）
  ├─ 需 F1 完成后再实施（与在线激活强制刷新配合）
  └─ 需三方协调发版
```

### 建议发版顺序

| 版本 | 内容 | 涉及项目 |
|------|------|---------|
| V2.1 | F1（UI 裁剪） | MaterialClient + BasePlatform |
| V2.1 或 V2.2 | F2（SubmitMachineCode） | UrbanManagement + MaterialClient |
| V2.2 | F3（ABP 审计，UrbanManagement 侧） | UrbanManagement |
| V2.2 或 V2.3 | F3（ABP 审计，MaterialClient 侧） | MaterialClient |
| V2.3+ | F4（签名版本，待讨论确认） | 全部 |

---

## 8. 与 V1 EPIC 的对照

| V1 EPIC 设计 | V2 修正 | 理由 |
|-------------|---------|------|
| 离线 + 在线双路径（§2 双路径表） | **在线为主，离线 UI 隐藏** | 运营反馈：离线路径未使用，增加维护负担 |
| `StaticLicenseChecker` + `.urban` bootstrap | **代码保留，UI 移除** | 离线能力保留为应急，但不暴露给用户 |
| `GET /api/auth/license-file`（BasePlatform 离线下载） | **API 保留，管理后台 UI 隐藏** | 运维应急路径保留 |
| `UrbanWeighingRecord` 无机器码记录 | **新增 `SubmitMachineCode`** | 数据溯源需求 |
| 审计字段手动赋值 | **改由 ABP 框架自动管理** | 规范化、减少遗漏 |
| JWT 签名固定密钥 | **新增签名版本控制**（讨论稿） | 密钥轮换与安全需求 |

---

## 9. 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 离线 UI 移除后应急场景无法操作 | 极端情况下无法离线导入 | 保留代码路径；可通过命令行或手动文件拷贝应急 |
| ABP 审计字段名变更导致政府出站数据格式变化 | 政府平台对接异常 | 政府出站 DTO 做独立映射，不直接暴露审计字段 |
| `AddTime → CreationTime` 数据迁移失败 | 历史数据审计断裂 | 迁移脚本充分测试；保留旧列做过渡 |
| F4 签名版本方案未确定 | 密钥轮换窗口期安全风险 | 可先完成 F1–F3，F4 单独立项 |

---

## 10. 非目标

- V1 EPIC 架构设计调整（JWT 唯一权威、仅 5001、iss=BasePlatform 不变）
- 非 5001 产品改动
- `POST /api/urban/auth/activate` 代理实现（属于 V1 P0 遗留，见 [07 拟稿 §4.2](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md)）
- SignalR `UpdateClientLicense` 推送实现（V1 P1 可选）

---

## 11. 文档索引

| 编号 | 文档 |
|------|------|
| V1-EPIC | [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) |
| V1-缺口 | [07-客户端服务端EPIC缺口对齐拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md) |
| V1-客户端 | [06-MaterialClient.Urban迁移拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/06-MaterialClient.Urban迁移拟稿提案.md) |
| V2-本文档 | **00-EPIC-授权修正迭代V2-拟稿提案** |

---

## 12. 待确认项

| # | 问题 | 涉及 |
|---|------|------|
| 1 | F4 签名版本方案选型（A / B / C 或其他） | 全部 |
| 2 | UrbanManagement `AddTime` 是否所有实体均有此字段？是否有其它非标准审计字段名？ | UrbanManagement |
| 3 | 政府出站 DTO 是否直接使用实体审计字段？字段名变更的影响范围？ | UrbanManagement |
| 4 | `UrbanWeighingRecord.CreatorUserId` 在 API 无认证上下文时如何填充？ | UrbanManagement |
| 5 | BasePlatform 下载授权 UI 隐藏方式偏好（菜单权限 vs CSS 隐藏 vs 配置开关）？ | BasePlatform |

---

**文档版本**：0.1（初稿）
**创建日期**：2026-06-26
**最后更新**：2026-06-26
**状态**：拟稿 — 待团队评审
