# MaterialClient.Urban 迁移拟稿提案

> **文档类型**：拟稿提案（Draft Proposal）  
> **创建日期**：2026-06-25  
> **状态**：拟稿（已对照 MaterialClient 现网 + UrbanManagement V2 归档提案）  
> **范围**：`MaterialClient.Urban` + 共用层 `MaterialClient.Common`（授权相关）  
> **不在范围**：BasePlatform 实现（→ [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)、[03](./03-BasePlatform-JWT签发迁移拟稿提案.md)）、MaterialClient 主程序（5000/5010）

**前置**：[01-解决方案.md](./01-解决方案.md) · [05-联合发版说明.md](./05-联合发版说明.md)  
**Urban 服务端对齐**：[MaterialMonospec `2026-06-25-urbanmanagement-migration-draft-proposal-v2`](../../../MaterialMonospec/openspec/changes/archive/2026-06-25-urbanmanagement-migration-draft-proposal-v2/proposal.md)（取代 vault [04](./04-UrbanManagement迁移拟稿提案.md) 作为 Urban 侧最新契约）  
**MaterialClient 代码基线**：[MaterialClient](https://github.com/feewillowherb/MaterialClient) `main`（2026-06-25）

> **前提（无旧 token 兼容）**：UrbanManagement **尚未上线**，不存在需兼容的 Urban 本地签发 JWT 或 `iss=UrbanManagement` 生产 token。客户端 **仅**接受 BasePlatform 签发、`iss=BasePlatform` 的新 JWT；**不**配置 `Jwt:LegacyIssuers`，**不**为旧 issuer / 旧 claim 组合留分支。

## 1. 提案摘要

| 部分 | 主题 | 要点 |
|------|------|------|
| **§A AccessCode** | 数据语义 | 本地 `LicenseInfo.BuildLicenseNo` → **`AccessCode`**；移除 **`FdBuildLicenseNo`**、**`AuthToken`** |
| **§B JWT 验权** | 改造现网 | `StaticLicenseChecker`：**仅** `iss=BasePlatform`；JWT claim **`accessCode`** → 本地 `AccessCode`；校验 **`machineCode`** |
| **§C 激活与导入** | 补齐缺口 | Refit **`POST /api/urban/auth/activate`**；保留启动读 **`license.urban`** |
| **§D SignalR** | 对齐 Urban V2 | 保留 **`VerifyJwtAsync`**；适配 **`JwtAntiTamperResult`** / **`GetClientProjectLicenseInfo`**；可选 **`UpdateClientLicense`** |

**与 Urban V2 的分歧（有意保留）**：Urban 提案写「客户端 `BuildLicenseNo` 暂不重命名」——本仓库仍执行 **`AccessCode` 重命名**（与 `GovProject.AccessCode` 语义一致）；Hub/JSON **wire 名**仍为 `buildLicenseNo`，客户端映射到 `AccessCode`。

---

## 2. UrbanManagement V2 契约（客户端须对齐）

来源：[`urban-jwt-delegation`](../../../MaterialMonospec/openspec/changes/archive/2026-06-25-urbanmanagement-migration-draft-proposal-v2/specs/urban-jwt-delegation/spec.md)、[`jwt-anti-tamper`](../../../MaterialMonospec/openspec/changes/archive/2026-06-25-urbanmanagement-migration-draft-proposal-v2/specs/jwt-anti-tamper/spec.md)。

### 2.1 Urban 代理 API（MaterialClient 调用）

| 方法 | Urban 路径 | 说明 |
|------|------------|------|
| POST | **`/api/urban/auth/activate`** | 代理 BasePlatform `POST /api/auth/activate-urban`；Body `{ productCode: 5001, code, machineCode }` |
| GET | `/api/urban/auth/license-file` | 代理离线下载（运维/Urban 管理端为主；客户端可选手动部署 `license.urban`） |

> **路径约定**：客户端 → Urban 使用 **`activate`**；Urban → BasePlatform 内部仍调 **`activate-urban`**（与 EPIC、[04](./04-UrbanManagement迁移拟稿提案.md) 一致）。

**激活成功响应**（Urban 透传 BasePlatform，字段以联调为准）须含 **`jwtToken`**，以及 `proId` / `proName` / `authEndDate`；接入码可在响应或 JWT claim 中提供。

### 2.2 JWT 签发与验签（Urban V2 变更 — 客户端必改）

| 项 | MaterialClient 现网 | Urban V2 / BasePlatform 委托后 |
|----|---------------------|--------------------------------|
| **`iss` claim** | `UrbanManagement` | **`BasePlatform`** |
| **`aud` claim** | `MaterialClient.Urban` | 不变 |
| 接入码 claim | `buildLicenseNo`（现网 MaterialClient 读取） | JWT：**`accessCode`**（与 [01](./01-解决方案.md) / BasePlatform 03 一致）；Hub JSON 仍为 **`buildLicenseNo`** → 映射到本地 **`AccessCode`** |
| `fdBuildLicenseNo` claim | 现网读取并落库 | **不读取、不落库** |
| 公钥 | `Jwt:PublicKey` | **BasePlatform 签发公钥** |

`StaticLicenseChecker`：**仅** `ValidIssuer = "BasePlatform"`；`iss=UrbanManagement` 或其它 issuer **一律拒绝**。

### 2.3 SignalR（现网已有，V2 改 JWT 来源）

| Hub 方法 | 方向 | V2 行为 | 客户端动作 |
|----------|------|---------|------------|
| **`VerifyJwtAsync(jwt, proId)`** | C→S→C | 验签通过后 **`ServerJwt` 来自 BasePlatform**（非 Urban 本地重签） | 现网 **`DeviceStatusSignalRClient`** 已调用 → 继续用 **`StoreServerJwtAsync`** |
| **`GetClientProjectLicenseInfo(proId)`** | C→S→C | 返回 **`buildLicenseNo`**（= `GovProject.AccessCode`）、`proName`、`authEndTime` 等 | 映射 **`buildLicenseNo` → `AccessCode`**；**不**同步 `fdBuildLicenseNo` |
| **`UpdateClientLicense`** | S→C | 推送 BasePlatform JWT（V2 新增能力） | **可选**：注册 handler → 验签 → `StoreServerJwtAsync` |

**不实施**：REST `POST /api/urban/auth/verify`（启动门禁仍纯本地 JWT）。

### 2.4 首发约束（无灰度兼容）

Urban 与 MaterialClient.Urban **同期首发**即可；无需为「Urban 旧签发 / 旧 JWT」预留客户端兼容逻辑。联调环境重新激活或重新下发 `.urban` 即可。

---

## 3. MaterialClient 现网基线

### 3.1 已有组件

| 组件 | 路径 | 现网行为 |
|------|------|----------|
| `LicenseInfo` | `MaterialClient.Common/Entities/LicenseInfo.cs` | `AuthToken`、`BuildLicenseNo`、`FdBuildLicenseNo`、`LatestJwtToken` |
| `StaticLicenseChecker` | `MaterialClient.Common/Services/StaticLicenseChecker.cs` | `iss=UrbanManagement`；读 `buildLicenseNo` / `fdBuildLicenseNo` |
| 启动验权 | `MaterialClientUrbanModule.TryExecuteStartupLicenseCheckAsync` | `LatestJwtToken` → 回退 `license.urban`；成功时**不写回** `LatestJwtToken` |
| 失败 UI | `App.axaml.cs` | `UnauthorizedNoticeWindow` |
| SignalR | `DeviceStatusSignalRClient.SyncProjectLicenseFromServerAsync` | `VerifyJwtAsync` + `GetClientProjectLicenseInfo` |
| 激活 | — | **无** Urban 在线激活；5000 走 `VerifyAuthorizationCodeAsync` → BasePlatform |

### 3.2 缺口（本提案补齐）

- `IUrbanAuthApi` 无 **`activate`**
- `iss` / `machineCode` claim 未对齐 Urban V2
- Hub DTO 仍用 `BuildLicenseNo` 字段名写入 `LicenseInfo`
- 无 `UpdateClientLicense` 订阅（V2 可选）

---

## 4. §A AccessCode 实体迁移

### 4.1 目标实体

```csharp
// MaterialClient.Common/Entities/LicenseInfo.cs（目标）
public class LicenseInfo : Entity<Guid>
{
    public Guid ProjectId { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string? ProName { get; set; }
    public string? AccessCode { get; set; }       // 原 BuildLicenseNo
    public string? LatestJwtToken { get; set; }
    public string? MachineCode { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

- **删除** `AuthToken`、`FdBuildLicenseNo`
- EF Core Migration 生成变更（拟稿不写 SQL）

### 4.2 Wire 名 vs 域内名

| 来源 | Wire / Claim 名 | 本地属性 |
|------|-------------------|----------|
| Hub `JwtAntiTamperResult.BuildLicenseNo` | `buildLicenseNo` | **`AccessCode`** |
| Hub `ClientProjectLicenseInfoDto.buildLicenseNo` | `buildLicenseNo` | **`AccessCode`** |
| JWT claim | `accessCode`（BasePlatform 目标契约） | **`AccessCode`** |
| 称重上传 DTO | 政府协议仍可能叫 `buildLicenseNo` | 值取自 **`LicenseInfo.AccessCode`** |

---

## 5. §B JWT 本地验权

### 5.1 `StaticLicenseChecker` 改造

```csharp
// 要点（非完整代码）
ValidIssuer = "BasePlatform";                    // 唯一接受；拒绝 UrbanManagement 等
ValidAudience = "MaterialClient.Urban";

// ExtractClaimsFromPrincipal：
// - proId, proName, exp, machineCode（machineCode 须与本机一致）
// - AccessCode ← claim "accessCode"（不读 buildLicenseNo / fdBuildLicenseNo claim）
```

`LicenseCheckResult`：`BuildLicenseNo` 属性改名为 **`AccessCode`**（或保留属性名仅改语义，推荐改名以减少混淆）。

### 5.2 启动流程（改造 `TryExecuteStartupLicenseCheckAsync`）

现网流程保留，**增补**：

1. 验签使用更新后的 `StaticLicenseChecker`（`iss=BasePlatform`）
2. 自 **`license.urban` bootstrap 成功**时，将 JWT 文本写入 **`LatestJwtToken`**
3. 持久化字段改用 **`AccessCode`**，不再写 `FdBuildLicenseNo` / `AuthToken`

---

## 6. §C 在线激活与离线导入

### 6.1 Refit（`MaterialClient.Urban/Api/IUrbanManagementApi.cs`）

```csharp
[Post("/api/urban/auth/activate")]
Task<ApiResponse<ActivateUrbanResponse>> ActivateUrbanAsync(
    [Body] ActivateUrbanRequest request,
    CancellationToken ct = default);
```

`ActivateUrbanRequest`：`ProductCode = 5001`、`Code`、`MachineCode`（**不传 ProId**）。

### 6.2 `ILicenseService.ActivateUrbanAsync`（Common 层）

1. 调 Urban **`activate`**
2. `CheckLicenseFromTokenAsync(jwtToken)`（含 machineCode）
3. Insert/Update `LicenseInfo`：`LatestJwtToken` + Claims 元数据；**无 AuthToken**

Urban 授权 UI：**新增** Urban 专用对话框（现网无 `AuthCodeWindow`）；**禁止** 5001 走 `VerifyAuthorizationCodeAsync` 直连 BasePlatform。

### 6.3 离线 `.urban`

| 方式 | 说明 |
|------|------|
| 启动读 `SystemSettings.LicenseFilePath`（默认 `license.urban`） | 保留；成功后写 `LatestJwtToken` |
| Urban `GET /api/urban/auth/license-file` | 运维/工具链；客户端可选下载助手 |
| 用户文件导入 | 可选增强 |

---

## 7. §D SignalR 对齐 Urban V2

改造 **`DeviceStatusSignalRClient.SyncProjectLicenseFromServerAsync`**：

### 7.1 `VerifyJwtAsync` 路径（保留）

```
GetLocalJwtTokenAsync → Hub VerifyJwtAsync → JwtAntiTamperResult
  → StoreServerJwtAsync(serverJwt, proName, accessCodeFromBuildLicenseNo, authEndTime)
```

- `StoreServerJwtAsync` 签名：`buildLicenseNo` 参数改名为 **`accessCode`**（或内部映射）
- **`ServerJwt` 为 BasePlatform 签发**；客户端仍只存 `LatestJwtToken`

### 7.2 `GetClientProjectLicenseInfo` 路径（字段同步）

- `SyncProjectFieldsFromServerAsync(proName, accessCode, authEndTime)` — **移除 `fdBuildLicenseNo` 参数**
- JSON **`buildLicenseNo`** → 本地 **`AccessCode`**

### 7.3 `UpdateClientLicense`（V2 可选）

若 Urban Hub 实现推送（见 `urban-jwt-delegation` spec）：

```csharp
_connection.On<ClientLicenseUpdateDto>("UpdateClientLicense", async dto => { ... });
```

验签通过后 `StoreServerJwtAsync`；与 `VerifyJwtAsync` 共用验签逻辑。

---

## 8. 涉及文件清单

| 路径 | 改动 |
|------|------|
| `MaterialClient.Common/Entities/LicenseInfo.cs` | AccessCode；删 AuthToken / FdBuildLicenseNo |
| `MaterialClient.Common/Migrations/*` | 新 Migration |
| `MaterialClient.Common/Services/StaticLicenseChecker.cs` | iss=BasePlatform；machineCode；AccessCode |
| `MaterialClient.Common/Services/IStaticLicenseChecker.cs` | `LicenseCheckResult.AccessCode` |
| `MaterialClient.Common/Services/Authentication/LicenseService.cs` | `ActivateUrbanAsync`；改 Store/Sync 签名 |
| `MaterialClient.Common/Services/DeviceStatusSignalRClient.cs` | Hub DTO 映射；可选 UpdateClientLicense |
| `MaterialClient.Common/Models/JwtAntiTamperResult.cs` | 注释：`BuildLicenseNo` = 接入码 → 存 AccessCode |
| `MaterialClient.Common/Models/ClientProjectLicenseInfoDto.cs` | 映射到 AccessCode |
| `MaterialClient.Urban/MaterialClientUrbanModule.cs` | 启动写 LatestJwtToken；AccessCode |
| `MaterialClient.Common/Api/IUrbanAuthApi.cs` | activate |
| `MaterialClient.Urban/Services/UrbanServerUploadService.cs` | AccessCode 替代 BuildLicenseNo |
| `MaterialClient.Urban/Services/UrbanAttachmentSyncService.cs` | 同上 |
| Urban 授权 UI（新） | 在线激活 |
| `appsettings.json` | `Jwt:PublicKey`（BasePlatform 公钥） |

---

## 9. 与 Urban V2 / BasePlatform 依赖

| 依赖 | 来源 | 客户端前置条件 |
|------|------|----------------|
| `activate` 代理 | Urban V2 §B | Refit 路径与响应含 `jwtToken` |
| JWT `iss=BasePlatform` | Urban V2 `jwt-anti-tamper` | 更新 `StaticLicenseChecker` |
| Hub `VerifyJwtAsync` 返回 BasePlatform JWT | Urban V2 tasks §3 | `StoreServerJwtAsync` 不变 |
| `GetClientProjectLicenseInfo.buildLicenseNo` | Urban V2 §A | 映射 → `AccessCode` |
| BasePlatform 公钥 | 03 + Urban 配置同步 | `Jwt:PublicKey` 更新 |

**发版顺序**（与 [05](./05-联合发版说明.md) 一致）：

| 阶段 | 客户端交付 | 阻塞 |
|------|------------|------|
| P-Client-1 | AccessCode 实体 + `iss=BasePlatform` + machineCode 校验 | 可与 Urban 并行 |
| P-Client-2 | `activate` + UI | Urban JWT 委托已启用 |
| P-Client-3 | `UpdateClientLicense` handler | Urban Hub 推送就绪（可选） |

---

## 10. 实施步骤

| 步骤 | 工作 | 预估 |
|------|------|------|
| 1 | LicenseInfo + Migration + 引用清理 | 1d |
| 2 | StaticLicenseChecker（iss/claims/machineCode） | 0.5d |
| 3 | TryExecuteStartupLicenseCheckAsync 回写 LatestJwtToken | 0.25d |
| 4 | IUrbanAuthApi + ActivateUrbanAsync + UI | 1.5d |
| 5 | DeviceStatusSignalRClient + DTO 映射 | 0.5d |
| 6 | Urban 上传服务 AccessCode | 0.25d |
| 7 | 与 Urban V2 + BasePlatform 联调 | 1d |
| **合计** | | **~5d** |

---

## 11. 测试清单

| # | 场景 | 预期 |
|---|------|------|
| 1 | JWT `iss=BasePlatform`，claim 齐全 | 启动/SignalR 验签通过 |
| 2 | JWT `iss=UrbanManagement` 或其它 issuer | **拒绝**（无兼容分支） |
| 3 | JWT claim `accessCode` | 写入 `LicenseInfo.AccessCode` |
| 4 | JWT 含 `buildLicenseNo` / `fdBuildLicenseNo` claim 无 `accessCode` | **拒绝**（不读废弃 claim） |
| 5 | `machineCode` 不匹配 | 启动失败 |
| 6 | `activate` 成功 | `LatestJwtToken` 有值；无 AuthToken |
| 7 | Hub `VerifyJwtAsync` | `ServerJwt` 覆盖 `LatestJwtToken` |
| 8 | Hub `GetClientProjectLicenseInfo` | JSON `buildLicenseNo` → `AccessCode` |
| 9 | bootstrap `license.urban` | DB 含 `LatestJwtToken` |

---

## 12. 回滚

| 故障 | 回滚 |
|------|------|
| Migration | EF 回滚上一版本 |
| JWT/激活逻辑 | 回退客户端版本；联调环境重新下发新 JWT / `.urban` |
| 无旧 token 义务 | 不回滚到 `iss=UrbanManagement` 验签路径 |

---

## 13. 文档索引

| 文档 | 说明 |
|------|------|
| [04-UrbanManagement迁移拟稿提案.md](./04-UrbanManagement迁移拟稿提案.md) | vault 早期 Urban 拟稿（部分 API 名已过时） |
| [UrbanManagement V2 归档](../../../MaterialMonospec/openspec/changes/archive/2026-06-25-urbanmanagement-migration-draft-proposal-v2/proposal.md) | **Urban 侧最新契约** |
| [05-联合发版说明.md](./05-联合发版说明.md) | 跨仓发版顺序 |

---

**文档版本**：1.2（无旧 JWT 兼容）  
**最后更新**：2026-06-25
