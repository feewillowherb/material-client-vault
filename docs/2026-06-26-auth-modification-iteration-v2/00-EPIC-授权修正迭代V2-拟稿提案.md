# 授权修正迭代 V2 — EPIC 拟稿提案

> **文档类型**：EPIC Draft Proposal（拟稿提案）
> **创建日期**：2026-06-26
> **状态**：拟稿
> **迭代版本**：V2（功能修正）
> **范围**：`MaterialClient.Urban` + `MaterialClient.Common` + `UrbanManagement` + `BasePlatform`（仅 ProductCode 5001）
> **前置基线**：
>
> - [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)（V1 EPIC）
> - [07-客户端服务端EPIC缺口对齐拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md)（V1 缺口对齐）
> - [06-MaterialClient.Urban迁移拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/06-MaterialClient.Urban迁移拟稿提案.md)（客户端迁移）

---

## 0. 文档定位


| 维度           | 说明                                                                    |
| ------------ | --------------------------------------------------------------------- |
| **迭代性质**     | 基于 V1 EPIC 实施反馈的**功能修正**迭代，非新增架构                                      |
| **与 V1 的关系** | 不推翻 V1 设计决策（JWT 唯一权威、仅 5001、iss=BasePlatform），在 V1 基础上做**裁剪、补充和规范修正** |
| **驱动来源**     | 运营与开发团队在 V1 实施/联调阶段发现的功能性问题                                           |


---

## 1. 提案摘要


| #      | 修正主题               | 归属项目                                            | 优先级    | 摘要                                                                                                        |
| ------ | ------------------ | ----------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------- |
| **F1** | 客户端改为纯在线验证，隐藏离线 UI | MaterialClient.Urban · BasePlatform             | **P0** | 离线验证业务逻辑代码保留（`StaticLicenseChecker`、`.urban` bootstrap）；离线导入 UI 删除；BasePlatform 下载授权功能 UI 隐藏              |
| **F2** | 称重记录新增提交时机器码字段     | UrbanManagement                                 | **P1** | `UrbanWeighingExtension`、`UrbanWeighingRecord` 新增 `SubmitMachineCode` 字段，提交数据时写入                          |
| **F3** | ABP 审计字段改由框架控制     | UrbanManagement · MaterialClient                | **P1** | 移除领域方法中手动赋值 `CreationTime`/`CreatorUserId`/`LastModificationTime`/`LastModifierUserId` 等字段，改用 ABP 拦截器自动填充 |
| **F4** | 重新激活后旧设备令牌失效机制     | BasePlatform · UrbanManagement · MaterialClient | **P0** | 服务端 `VerifyJwtAsync` 增加 machineCode 比对；新设备激活后旧设备 JWT 在连接服务端时被拒绝并强制终止运行                                    |


---

## 2. F1 — 客户端纯在线验证 + 离线 UI 隐藏

### 2.1 背景

V1 EPIC 设计了**离线 + 在线**双路径（§2 双路径表）。运营反馈：实际部署中 5001 客户端均在有网环境运行，离线路径增加了维护复杂度且从未在生产中使用。因此决定：

- **客户端统一走在线激活**（`POST /api/urban/auth/activate`）
- **离线验证的业务逻辑代码保留**（防回退 / 应急），但**离线导入相关 UI 入口移除**
- **BasePlatform 管理后台下载授权功能 UI 隐藏**（不再向运营暴露离线入口）

### 2.2 MaterialClient.Urban 改动

#### 保留（代码层面）


| 组件                                                       | 保留理由                                |
| -------------------------------------------------------- | ----------------------------------- |
| `StaticLicenseChecker`                                   | JWT 本地验签仍为启动门禁；离线应急可用               |
| `license.urban` 文件读取逻辑                                   | 代码路径保留；启动时若本地已有 JWT 仍可 bootstrap 验签 |
| `LatestJwtToken` 持久化                                     | 在线激活成功后仍写入                          |
| SignalR `VerifyJwtAsync` / `GetClientProjectLicenseInfo` | 保留；与在线激活配合                          |
| `LicenseInfo.LatestJwtToken` 字段                          | 保留                                  |
| `LicenseInfo.MachineCode` 字段                             | 保留                                  |


#### 删除（UI 层面）


| UI 组件               | 说明                            |
| ------------------- | ----------------------------- |
| 离线授权文件导入对话框         | 用户不再需要手动导入 `.urban` 文件的 UI 入口 |
| 设置页面中「离线授权」相关 UI 区域 | 导出/导入按钮、文件路径选择等               |
| 任何引导用户走离线流程的提示文案    | 如「请导入离线授权文件」                  |


#### 保留但隐藏


| UI 组件                                 | 处理方式              |
| ------------------------------------- | ----------------- |
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


| 改动项                              | 说明                                                          |
| -------------------------------- | ----------------------------------------------------------- |
| 下载授权管理页面 UI 隐藏                   | `DownloadUrbanLicense` 相关管理后台入口通过 **CSS `display:none`** 隐藏 |
| `GET /api/auth/license-file` API | **保留**（代码不删，应急/运维路径仍可用），仅 UI 入口隐藏                           |
| `GET /api/auth/license-file` 门禁  | 可考虑增加权限标记（如仅管理员角色可访问），防止普通运营误操作                             |


### 2.4 影响评估


| 项目                    | 影响范围                            |
| --------------------- | ------------------------------- |
| MaterialClient.Urban  | UI 层裁剪（移除离线导入对话框及相关菜单项）；业务逻辑零改动 |
| MaterialClient.Common | 无改动                             |
| UrbanManagement       | 无改动                             |
| BasePlatform（管理后台）    | 下载授权 UI 隐藏；API 保留               |


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


| 项目                   | 文件                                     | 改动                            |
| -------------------- | -------------------------------------- | ----------------------------- |
| UrbanManagement      | `Entities/UrbanWeighingRecord.cs`      | 新增 `SubmitMachineCode`        |
| UrbanManagement      | `EntityFrameworkCore/*DbContext*.cs`   | EF Core 映射                    |
| UrbanManagement      | `Migrations/*`                         | 新 Migration                   |
| MaterialClient.Urban | `Entities/UrbanWeighingExtension.cs`   | 新增 `SubmitMachineCode`        |
| MaterialClient.Urban | `Services/UrbanServerUploadService.cs` | 上传 DTO 携带 `submitMachineCode` |
| MaterialClient.Urban | `MaterialClient.Common/Migrations/*`   | 新 Migration                   |


---

## 4. F3 — ABP 审计字段改由框架控制

### 4.1 背景

当前 `UrbanManagement` 和 `MaterialClient` 项目中，部分实体在领域方法或服务层**手动赋值**审计字段（如 `CreationTime`、`CreatorUserId`、`LastModificationTime` 等），未充分利用 ABP 框架的自动审计填充机制。这导致：

- 审计逻辑分散，容易遗漏
- 与 ABP 规范不一致，增加维护负担
- 部分字段名不标准（如 `AddTime` 而非 `CreationTime`）

### 4.2 ABP 标准审计接口映射


| ABP 接口                                            | 自动填充字段                                        | 对应当前非标准字段          |
| ------------------------------------------------- | --------------------------------------------- | ------------------ |
| `IHasCreationTime`                                | `CreationTime`                                | `AddTime`          |
| `ICreationAudited`（继承 `IHasCreationTime`）         | `CreationTime` + `CreatorUserId`              | `AddTime` + 无对应    |
| `IModificationAudited`（继承 `IHasModificationTime`） | `LastModificationTime` + `LastModifierUserId` | 无标准对应              |
| `ISoftDelete`                                     | `IsDeleted`                                   | `DeleteStatus`（如有） |
| `IDeletionAudited`（继承 `ISoftDelete`）              | `DeletionTime` + `DeleterUserId`              | 无标准对应              |


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

> **已确认**：所有实体均有 `AddTime` 字段。


| 实体                       | 当前审计字段                                                                                                | 改造方向                                                       |
| ------------------------ | ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `GovProject`             | `AddTime`                                                                                             | 实现 `IHasCreationTime`；`AddTime` → `CreationTime`           |
| `GovSyncData`            | `AddTime`                                                                                             | 实现 `IHasCreationTime`                                      |
| `GovLog`                 | `AddTime`                                                                                             | 实现 `IHasCreationTime`                                      |
| `UrbanWeighingRecord`    | `AddTime`（由服务端 `ReceiveAsync` 手动设置 `AddTime = DateTime.Now`，见 `urban-weighing-record-reception` spec） | 实现 `IHasCreationTime`；移除 `ReceiveAsync` 中手动赋值              |
| `UrbanWeighingExtension` | `AddTime`（ABP UoW 中创建，见 `urban-weighing-extension` spec）                                              | 若使用 `AggregateRoot` 基类则 ABP 自动管理；否则显式实现 `IHasCreationTime` |


#### MaterialClient


| 实体                       | 当前审计字段                                                                 | 改造方向                                                                |
| ------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `LicenseInfo`            | `CreatedAt`, `UpdatedAt`                                               | 添加 `CreationTime`/`LastModificationTime` + `SaveChangesInterceptor` |
| `UrbanWeighingExtension` | 由 `IUrbanWeighingExtensionService.CreateForRecordAsync` 创建（ABP UoW 管理） | 同上                                                                  |


> **注意**：`UrbanWeighingRecord` 的 `AddTime` 字段由 `urban-weighing-record-reception` spec 明确定义为"服务端入库时间"。改造为 `IHasCreationTime` 后，ABP 自动填充 `CreationTime`，语义等价。需确认 EF 映射列名兼容（`CreationTime` 映射到 `AddTime` 列或执行列重命名迁移）。

### 4.5 注意事项

- `**UrbanWeighingRecord` 无用户上下文**：称重记录由 MaterialClient 通过 API 推送至 UrbanManagement，API 无用户认证上下文，`CreatorUserId` 无法自动填充。**决议**：仅实现 `IHasCreationTime`（不实现 `ICreationAudited`）；如需记录数据来源项目，可在接收 DTO 中读取客户端上报的 `ProId`，手动写入独立的 `ProjectId` 字段（而非复用审计字段）。
- **数据迁移**：`AddTime → CreationTime` 需要数据迁移脚本，确保历史数据不丢失。所有实体均有 `AddTime`，需统一迁移。
- **JSON 序列化**：对外 API 响应中审计字段名变更需同步调整客户端映射（如有）。
- **政府出站 DTO 独立映射**：F3 实体属性标准化时，`GovSyncWorker` 出站 DTO 通过手动映射或 AutoMapper Profile 显式配置，不直接引用实体属性。出站字段名保持政府协议要求（如 `addTime`），不受实体属性重命名影响。

---

## 5. F4 — 重新激活后旧设备令牌失效机制

### 5.1 业务场景

基于对现有代码链路的分析，需要解决两个具体业务 Case：

#### Case 1：同项目跨设备重新激活，旧设备须失效

```
时间线：
  T1: PC_A（机器码 M-A）激活项目 P，获得 JWT_A（claims: machineCode=M-A, jti=J1）
  T2: PC_B（机器码 M-B）激活同一项目 P，获得 JWT_B（claims: machineCode=M-B, jti=J2）
      → BasePlatform activate-urban 将 JC_ProductAuthority.MachineCode 回写为 M-B
      → UrbanManagement GovProject.MachineCode 更新为 M-B

期望：
  T3: PC_A 持 JWT_A 连接 UrbanManagement SignalR → 服务端判定 JWT 已失效
      → PC_A 客户端收到失效通知 → 终止运行（弹出未授权提示并退出）
```

#### Case 2：同设备授权时间变更重新签发，正常更新

```
时间线：
  T1: PC_A（机器码 M-A）激活项目 P，获得 JWT_A（claims: machineCode=M-A, exp=2026-12-31）
  T2: 运营在 BasePlatform 修改项目 P 的授权时间（如延期至 2027-12-31）
  T3: PC_A 重新在线激活，获得 JWT_B（claims: machineCode=M-A, exp=2027-12-31, jti=J3）

期望：
  T4: PC_A 通过 VerifyJwtAsync → 服务端返回 ServerJwt = JWT_B
      → PC_A 正常更新 LatestJwtToken 和 AuthEndTime，继续运行
```

### 5.2 现状分析

基于 `FdSoft.BasePlatform`、`FdSoft.BasePlatform.PublicApi`、`MaterialMonospec` 规范的实际代码链路：

#### BasePlatform activate-urban（已实现）

```csharp
// FdSoft.BasePlatform.PublicApi/Controllers/AuthController.cs:ActivateUrban
// 1. 验 Redis 一次性授权码
// 2. 回写 JC_ProductAuthority.MachineCode = 客户端上报的 machineCode（无条件覆盖）
// 3. 签发 JWT（claims 含 machineCode, jti=new GUID）
// 4. 返回 jwtToken + proId + proName + accessCode + authEndDate
```

**关键发现**：每次 `activate-urban` 都会**无条件覆盖** `JC_ProductAuthority.MachineCode` 为最新激活设备的机器码。

#### JWT Claims（BasePlatformJwtTokenGenerator）

```csharp
// Claims: proId, proName, accessCode, machineCode, exp, jti
// jti: 每次签发生成新 GUID（天然唯一）
// machineCode: 来自 JC_ProductAuthority.MachineCode（即最新激活设备的机器码）
```

#### UrbanManagement VerifyJwtAsync（jwt-anti-tamper spec）

```
// 现有流程：
// 1. 验证 JWT RS256 签名（iss=BasePlatform, aud=MaterialClient.Urban）
// 2. 从 JWT 提取 proId → 查询 GovProject
// 3. 如果 GovProject 存在且 JWT 有效 → 调 BasePlatform license-file 获取新 JWT
// 4. 返回 JwtAntiTamperResult（Passed=true, ServerJwt=新签发的 JWT）

// 问题：VerifyAndCompareAsync 不比对 JWT 中的 machineCode 与 GovProject.MachineCode
```

#### UrbanManagement SignalR DeviceStatusHub

```
// signalr-device-status-upload spec:
// Hub 支持 JWT Bearer Token 认证
// 客户端连接时携带 JWT Token
// 服务端在 Hub 方法调用时验证 Token 有效性
```

### 5.3 问题根因

**Case 1 不生效的根因**：`VerifyJwtAsync` → `VerifyAndCompareAsync` 只检查：

1. JWT 签名是否有效（RS256）
2. `proId` 是否对应存在的 `GovProject`
3. `exp` 是否过期

**但不检查**：JWT 中的 `machineCode` 是否与 `GovProject.MachineCode` 一致。

因此 PC_A 持 JWT_A（`machineCode=M-A`）连接时，即使 `GovProject.MachineCode` 已更新为 `M-B`，`VerifyJwtAsync` 仍然返回 `Passed=true`。

### 5.4 方案设计

> **核心思路**：不需要新增数据库字段或签名版本号。利用已有的 `GovProject.MachineCode`（权威机器码）与 JWT 中的 `machineCode` claim 做比对即可实现 Case 1。Case 2 天然不需要额外机制。

#### 改动一：UrbanManagement `JwtAntiTamperService.VerifyAndCompareAsync` 增加 machineCode 比对

```
现有流程：
  验签 JWT → 提取 proId → 查 GovProject → 调 BasePlatform 获取新 JWT → 返回 Pass

新增逻辑（在查 GovProject 之后）：
  提取 JWT claims 中的 machineCode
  比对 GovProject.MachineCode
  若不一致 → 返回 Passed=false, Reason="授权设备已变更，请在当前设备重新激活"
```

**不需要新增数据库字段**。`GovProject.MachineCode` 已存在且在每次 `activate-urban` 时由 BasePlatform 更新。

#### 改动二：MaterialClient 处理 VerifyJwtAsync 失败 → 终止运行

```
现有流程（jwt-anti-tamper-sync spec）：
  VerifyJwtAsync 返回 Passed=false → 不修改 LicenseInfo → 跳过同步

新增逻辑：
  VerifyJwtAsync 返回 Passed=false 且 Reason 包含"设备已变更"或"授权已失效"
  → 清除 LicenseInfo.LatestJwtToken
  → 弹出 UnauthorizedNoticeWindow（仅在线激活入口，见 F1）
  → 终止客户端运行
```

#### 改动三（可选增强）：BasePlatform `license-file` API 增加 machineCode 参数校验

当前 `license-file` API 使用 `JCProductAuthority.MachineCode` 签发 JWT，不校验请求参数中的 `machineCode` 是否与库中一致。可增加校验以防御中间人替换：

```csharp
// LicenseFileAppService.BuildLicenseFileAsync 现有逻辑：
// request.MachineCode 仅用于构建 LicenseFileBuildRequest，不校验

// 增强建议：
if (request.MachineCode != productAuth.MachineCode)
    throw new CustomException("请求机器码与授权记录不一致");
```

### 5.5 Case 覆盖分析

#### Case 1 路径（跨设备重新激活，旧设备失效）

```
T1: PC_A 激活 → JC_ProductAuthority.MachineCode = M-A → GovProject.MachineCode = M-A
    → PC_A 获得 JWT_A(machineCode=M-A)

T2: PC_B 激活 → JC_ProductAuthority.MachineCode = M-B → GovProject.MachineCode = M-B
    → PC_B 获得 JWT_B(machineCode=M-B)

T3: PC_A 持 JWT_A 连接 UrbanManagement
    → VerifyJwtAsync → VerifyAndCompareAsync
    → 提取 JWT_A claims: machineCode = M-A
    → 查 GovProject.MachineCode = M-B
    → M-A != M-B → Passed=false, Reason="授权设备已变更"
    → 客户端清除 LatestJwtToken → 终止运行 ✅

T3': PC_B 持 JWT_B 连接
    → VerifyJwtAsync → machineCode = M-B == GovProject.MachineCode = M-B → Passed=true
    → 返回 ServerJwt → 正常运行 ✅
```

#### Case 2 路径（同设备授权时间变更，正常更新）

```
T1: PC_A 激活 → MachineCode = M-A → JWT_A(exp=2026-12-31)

T2: 运营修改授权时间 → 重新生成授权码

T3: PC_A 重新激活 → MachineCode 仍为 M-A（同一设备）
    → JC_ProductAuthority.MachineCode = M-A（未变化）
    → 签发 JWT_B(exp=2027-12-31, machineCode=M-A, jti=新GUID)

T4: PC_A VerifyJwtAsync → machineCode = M-A == GovProject.MachineCode = M-A → Passed=true
    → 返回 ServerJwt = JWT_B → 正常更新 LatestJwtToken ✅
```

### 5.6 与「签名版本控制」的关系

> **结论：不需要签名版本控制机制。**
>
> 原始需求表述为「存储最新签名版本，旧签名连接时失效」，但通过分析实际代码发现：
>
> 1. **JWT 的 `jti` claim 已是唯一标识**（每次签发新 GUID），不需要额外版本号
> 2. `**GovProject.MachineCode` 已存储最新授权设备的机器码**，不需要额外字段
> 3. **Case 1 的核心是 machineCode 比对**，与密钥版本无关
> 4. **Case 2 天然由现有 `VerifyJwtAsync` → `ServerJwt` 流程支持**
>
> 若未来确实需要 RSA 密钥轮换能力，可作为独立需求另行设计（当前暂无密钥泄露或定期轮换的业务需求）。

### 5.7 涉及改动清单


| 项目                     | 文件 / 组件                                                       | 改动                                                           |
| ---------------------- | ------------------------------------------------------------- | ------------------------------------------------------------ |
| **UrbanManagement**    | `JwtAntiTamperService.VerifyAndCompareAsync`                  | 新增 machineCode 比对逻辑                                          |
| **UrbanManagement**    | `JwtAntiTamperResult`                                         | 可能新增 `RevocationReason` 枚举或约定 Reason 前缀（如 `DEVICE_CHANGED:`） |
| **MaterialClient**     | `DeviceStatusSignalRClient.SyncProjectLicenseFromServerAsync` | 处理设备变更失效 → 清除 JWT → 终止运行                                     |
| **MaterialClient**     | `urban-license-startup-gate` spec                             | 可能需要在启动 SignalR 连接后增加首次验签（若尚未实现）                             |
| **BasePlatform**（可选增强） | `LicenseFileAppService.BuildLicenseFileAsync`                 | machineCode 参数校验                                             |


### 5.8 规范影响


| 规范                           | 需修订的 Scenario                         |
| ---------------------------- | ------------------------------------- |
| `jwt-anti-tamper`            | 新增 Scenario：machineCode 不匹配时返回 Fail   |
| `jwt-anti-tamper-sync`       | 新增 Scenario：设备变更失败时清除 JWT + 终止运行      |
| `urban-license-startup-gate` | 新增 Scenario：启动后首次 SignalR 验签发现设备变更的处理 |


### 5.9 注意事项

- **首次连接窗口（已接受）**：客户端启动后、SignalR 首次 `VerifyJwtAsync` 之前，存在短暂窗口期（数秒），旧设备的 JWT 仍可通过本地验签。**决议：接受窗口期**，不消除。窗口期内上传的称重数据可通过 F2 新增的 `SubmitMachineCode` 字段追溯来源设备。
- **宽限期**：**无宽限期**。运营重新激活意味着明确的设备迁移意图，旧设备应立即失效。
- **离线场景**：F1 已将离线路径 UI 隐藏，离线客户端无法感知设备变更。若离线客户端重新联网后首次 SignalR 连接触发 `VerifyJwtAsync`，此时才会检测到 machineCode 不一致。

---

## 6. 实施优先级与阶段划分

### 阶段一（P0，立即）


| 编号   | 工作                                                             | 归属              |
| ---- | -------------------------------------------------------------- | --------------- |
| F1-1 | MaterialClient.Urban 离线导入 UI 移除                                | MaterialClient  |
| F1-2 | BasePlatform 下载授权管理页面 UI 隐藏                                    | BasePlatform    |
| F1-3 | UnauthorizedNoticeWindow 裁剪（仅保留在线激活）                           | MaterialClient  |
| F4-1 | `JwtAntiTamperService.VerifyAndCompareAsync` 增加 machineCode 比对 | UrbanManagement |
| F4-2 | MaterialClient 处理设备变更失效 → 清除 JWT → 终止运行                        | MaterialClient  |


### 阶段二（P1，短中期）


| 编号   | 工作                                                     | 归属                               |
| ---- | ------------------------------------------------------ | -------------------------------- |
| F2-1 | UrbanWeighingRecord / Extension 新增 `SubmitMachineCode` | UrbanManagement + MaterialClient |
| F2-2 | 上传 DTO 新增字段 + 数据库迁移                                    | UrbanManagement + MaterialClient |
| F3-1 | UrbanManagement 实体实现 ABP 审计接口                          | UrbanManagement                  |
| F3-2 | 移除领域方法中手动审计赋值                                          | UrbanManagement                  |
| F3-3 | 数据库列名标准化迁移（`AddTime` → `CreationTime`）                 | UrbanManagement                  |
| F3-4 | MaterialClient `SaveChangesInterceptor`                | MaterialClient                   |


### 阶段三（P2，可选增强）


| 编号       | 工作                                        | 归属        |
| -------- | ----------------------------------------- | ---------- |
| F4-3（可选） | BasePlatform `license-file` API 增加 machineCode 参数校验 | BasePlatform |


---

## 7. 跨项目依赖与发版协调

### 依赖关系

```
F1（在线验证裁剪）
  ├─ MaterialClient 可独立发版（纯 UI 裁剪）
  └─ BasePlatform 可独立发版（UI 隐藏）

F4（设备失效机制）
  ├─ UrbanManagement F4-1 可独立发版（VerifyAndCompareAsync 增加 machineCode 比对）
  └─ MaterialClient F4-2 需与 UrbanManagement 联调（处理设备变更失效逻辑）
  └─ F1 完成后 MaterialClient F4-2 的"终止运行 → 弹出仅在线激活"体验更完整

F2（机器码字段）
  ├─ MaterialClient 先发版（DTO 新增字段）
  └─ UrbanManagement 同步或延后一版本（接收新字段）

F3（ABP 审计）
  ├─ UrbanManagement 可独立发版
  └─ MaterialClient 可独立发版
  └─ 注意：若 UrbanManagement 对外 API 响应中审计字段名变更，需与 MaterialClient 联调
```

### 建议发版顺序


| 版本          | 内容                           | 涉及项目                                            |
| ----------- | ---------------------------- | ----------------------------------------------- |
| V2.1        | F1（UI 裁剪） + F4（设备失效，P0）      | MaterialClient + UrbanManagement + BasePlatform |
| V2.1 或 V2.2 | F2（SubmitMachineCode）        | UrbanManagement + MaterialClient                |
| V2.2        | F3（ABP 审计，UrbanManagement 侧） | UrbanManagement                                 |
| V2.2 或 V2.3 | F3（ABP 审计，MaterialClient 侧）  | MaterialClient                                  |


---

## 8. 与 V1 EPIC 的对照


| V1 EPIC 设计                                      | V2 修正                        | 理由                  |
| ----------------------------------------------- | ---------------------------- | ------------------- |
| 离线 + 在线双路径（§2 双路径表）                             | **在线为主，离线 UI 隐藏**            | 运营反馈：离线路径未使用，增加维护负担 |
| `StaticLicenseChecker` + `.urban` bootstrap     | **代码保留，UI 移除**               | 离线能力保留为应急，但不暴露给用户   |
| `GET /api/auth/license-file`（BasePlatform 离线下载） | **API 保留，管理后台 UI 隐藏**        | 运维应急路径保留            |
| `UrbanWeighingRecord` 无机器码记录                    | **新增 `SubmitMachineCode`**   | 数据溯源需求              |
| 审计字段手动赋值                                        | **改由 ABP 框架自动管理**            | 规范化、减少遗漏            |
| JWT 签名固定密钥                                      | **新增设备失效机制**（machineCode 比对） | 跨设备重新激活后旧设备须失效      |


---

## 9. 风险与缓解


| 风险                                              | 影响                                     | 缓解                                          |
| ----------------------------------------------- | -------------------------------------- | ------------------------------------------- |
| 离线 UI 移除后应急场景无法操作                               | 极端情况下无法离线导入                            | 保留代码路径；可通过命令行或手动文件拷贝应急                      |
| ABP 审计字段名变更导致政府出站数据格式变化                         | 政府平台对接异常                               | 政府出站 DTO 做独立映射，不直接暴露审计字段                    |
| `AddTime → CreationTime` 数据迁移失败                 | 历史数据审计断裂                               | 迁移脚本充分测试；保留旧列做过渡                            |
| 启动至首次 SignalR 连接的窗口期                            | 旧设备在首次 VerifyJwtAsync 前短暂可用            | **已接受窗口期**；F2 `SubmitMachineCode` 可追溯窗口期内数据来源 |
| GovProject.MachineCode 更新延迟                     | Urban 代理 activate 后 GovProject 副本未及时同步 | activate 代理中同步更新 GovProject（见 V1 EPIC §2.4） |
| BasePlatform `activate-urban` 无条件覆盖 MachineCode | 误操作授权码会导致合法设备被挤下线                      | 可考虑增加确认步骤或审计日志                              |


---

## 10. 非目标

- V1 EPIC 架构设计调整（JWT 唯一权威、仅 5001、iss=BasePlatform 不变）
- 非 5001 产品改动
- `POST /api/urban/auth/activate` 代理实现（属于 V1 P0 遗留，见 [07 拟稿 §4.2](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md)）
- SignalR `UpdateClientLicense` 推送实现（V1 P1 可选）

---

## 11. 文档索引


| 编号      | 文档                                                                                                                   |
| ------- | -------------------------------------------------------------------------------------------------------------------- |
| V1-EPIC | [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)                                         |
| V1-缺口   | [07-客户端服务端EPIC缺口对齐拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/07-客户端服务端EPIC缺口对齐拟稿提案.md)                 |
| V1-客户端  | [06-MaterialClient.Urban迁移拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/06-MaterialClient.Urban迁移拟稿提案.md) |
| V2-本文档  | **00-EPIC-授权修正迭代V2-拟稿提案**                                                                                            |


---

## 12. 已确认项 & 待决策项

### 已确认


| #   | 问题                                                     | 决议                                                                            |
| --- | ------------------------------------------------------ | ----------------------------------------------------------------------------- |
| 1   | UrbanManagement `AddTime` 是否所有实体均有此字段？                 | **是**，所有实体均有 `AddTime`                                                        |
| 2   | `UrbanWeighingRecord.CreatorUserId` 在 API 无认证上下文时如何填充？ | **仅实现 `IHasCreationTime`**（不实现 `ICreationAudited`），数据来源项目用独立 `ProjectId` 字段记录 |
| 3   | BasePlatform 下载授权 UI 隐藏方式？                             | **CSS `display:none`**                                                        |
| 4   | F4 设备失效是否需要宽限期？                                        | **无宽限期**                                                                      |


### 已确认（续）


| #   | 问题                                                 | 决议         |
| --- | -------------------------------------------------- | ---------- |
| 5   | 政府出站 DTO 是否直接使用实体审计字段？字段名变更的影响范围？     | **方案 A：出站 DTO 独立映射** |
| 6   | F4-4 是否需要消除"启动至首次 SignalR 连接"的窗口期？ | **不消除（接受窗口期）** |

**§12.1 决议说明**：F3 实体审计字段标准化（`AddTime` → `CreationTime`）时，政府出站 DTO 通过手动映射或 AutoMapper Profile 显式配置，不直接引用实体属性。出站字段名保持政府协议要求不变（如 `addTime`），实体侧使用标准 ABP 属性名（`CreationTime`）。

**§12.2 决议说明**：客户端启动后至首次 SignalR `VerifyJwtAsync` 之间存在短暂窗口期（数秒），旧设备 JWT 在此期间可通过本地验签。接受此窗口期——旧设备短暂运行期间上传的称重数据可通过 F2 新增的 `SubmitMachineCode` 字段追溯来源设备。


---

## 13. 代码库参考

> F4 方案设计基于以下代码仓库的实际实现。

### BasePlatform（JWT 签发侧）


| 文件                              | 路径                                                                          | 用途                                                        |
| ------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------- |
| `AuthController`                | `FdSoft.BasePlatform.PublicApi/Controllers/AuthController.cs`               | `activate-urban`：验 Redis 码 → 回写 MachineCode → 签发 JWT      |
| `ActivateUrbanRequest`          | `FdSoft.BasePlatform.PublicApi/Models/ActivateUrbanRequest.cs`              | 请求 DTO：ProductCode, Code, MachineCode                     |
| `BasePlatformJwtTokenGenerator` | `FdSoft.BasePlatform.Service/BasePlatform/BasePlatformJwtTokenGenerator.cs` | RS256 JWT 签发器（iss=BasePlatform, aud=MaterialClient.Urban） |
| `LicenseFileAppService`         | `FdSoft.BasePlatform.Service/BasePlatform/LicenseFileAppService.cs`         | 离线 license-file 签发服务                                      |
| `LicenseClaimsInput`            | `FdSoft.BasePlatform.Model/Dto/BasePlatform/LicenseClaimsInput.cs`          | JWT Claims 输入模型                                           |
| `JCProductAuthority`            | `FdSoft.BasePlatform.Model/Models/BasePlatform/JCProductAuthority.cs`       | 授权表实体（MachineCode, AuthToken, AccessCode 等字段）             |


### MaterialMonospec（规范侧）


| 规范                                  | 路径                                                         | 用途                                   |
| ----------------------------------- | ---------------------------------------------------------- | ------------------------------------ |
| `jwt-anti-tamper`                   | `openspec/specs/jwt-anti-tamper/spec.md`                   | UrbanManagement JWT 防篡改服务规范（F4 改动目标） |
| `jwt-anti-tamper-sync`              | `openspec/specs/jwt-anti-tamper-sync/spec.md`              | 客户端 JWT 同步集成规范（F4 改动目标）              |
| `urban-jwt-delegation`              | `openspec/specs/urban-jwt-delegation/spec.md`              | Urban JWT 委托规范                       |
| `urban-license-startup-gate`        | `openspec/specs/urban-license-startup-gate/spec.md`        | 客户端启动门禁规范                            |
| `materialclient-urban-activation`   | `openspec/specs/materialclient-urban-activation/spec.md`   | 在线激活规范                               |
| `materialclient-license-accesscode` | `openspec/specs/materialclient-license-accesscode/spec.md` | LicenseInfo AccessCode 规范            |
| `urban-weighing-extension`          | `openspec/specs/urban-weighing-extension/spec.md`          | UrbanWeighingExtension 实体规范（F2 改动目标） |
| `urban-weighing-record-reception`   | `openspec/specs/urban-weighing-record-reception/spec.md`   | 称重记录接收规范（F2/F3 改动目标）                 |
| `signalr-device-status-upload`      | `openspec/specs/signalr-device-status-upload/spec.md`      | SignalR 设备状态规范                       |


---

**文档版本**：0.4（所有决策项已确认；§12.1 出站 DTO 独立映射；§12.2 接受启动窗口期）
**创建日期**：2026-06-26
**最后更新**：2026-06-26
**状态**：拟稿 — 所有待决策项已确认，待团队评审