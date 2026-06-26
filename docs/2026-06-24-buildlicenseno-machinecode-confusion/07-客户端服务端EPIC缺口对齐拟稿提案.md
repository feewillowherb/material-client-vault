# 客户端与服务端 EPIC 缺口对齐拟稿提案

> **文档类型**：拟稿提案（Draft Proposal）  
> **创建日期**：2026-06-26  
> **状态**：拟稿  
> **范围**：`MaterialClient.Urban` ↔ `UrbanManagement` ↔ `BasePlatform`（**仅 ProductCode 5001**）  
> **对照基线**：[00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)  
> **前置**：[01-解决方案](./01-解决方案.md) · [05-联合发版说明](./05-联合发版说明.md) · [06-MaterialClient.Urban迁移拟稿提案](./06-MaterialClient.Urban迁移拟稿提案.md)

---

## 1. 提案摘要

各仓库已按 [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)～[06](./06-MaterialClient.Urban迁移拟稿提案.md) 与 Urban V2 归档变更**分头推进**，但与 EPIC 定义的**端到端契约**仍存在多处未实现或未对齐项。本拟稿汇总 **2026-06-26 代码基线**下的**实现缺口**（不含联调勾选状态），并给出统一收口方案。

| 优先级 | 主题 | 影响 |
|--------|------|------|
| **P0** | JWT `iss` 三方不一致 | 任意路径签发的 JWT 在客户端验签失败 |
| **P0** | Urban 未暴露 `activate` 代理 | 在线激活闭环断裂 |
| **P1** | REST 路径与 EPIC 文档不一致（`license-file`） | 运维/文档与 ABP 常规路由不一致 |
| **P1** | Hub `UpdateClientLicense` 仅客户端订阅 | 续期推送无法触发 |
| **P2** | `FdBuildLicenseNo` / Hub 字段未按 EPIC 废弃 | 客户端已忽略，服务端仍输出 |
| **P2** | `GovProject.LastMachineCodeUpdate` 未落地 | EPIC 数据模型缺口 |

> **联调与测试**：端到端联调、OpenSpec tasks §7 勾选及发版验收 **由用户负责**；tasks 未勾选 **不作为**本拟稿的缺口判定依据（见 §7）。

---

## 2. 对照方法

- **设计基线**：EPIC + [01-解决方案](./01-解决方案.md)
- **对外 Urban 激活路径**：**`POST /api/urban/auth/activate`**（与 EPIC §2.4 一致；Urban 内部仍转发 BasePlatform `POST /api/auth/activate-urban`）
- **代码基线**（2026-06-26）：
  - `MaterialClient`：`update-materialclient-urban-accesscode-jwt`（实现 tasks §1–6）
  - `UrbanManagement`：Urban V2 归档（JWT 委托等）
  - `BasePlatform` / `PublicApi`：JWT 签发等归档实现

图例：**✅ 已对齐** · **⚠️ 部分对齐** · **❌ 未实现/冲突**

---

## 3. 端到端契约对照矩阵

### 3.1 BasePlatform.PublicApi

| EPIC / 01 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `GET /api/auth/license-file`（5001） | `AuthController` 已实现 | ✅ | |
| `POST /api/auth/activate-urban`（5001，Urban 内部调用） | `AuthController` 已实现 | ✅ | 验 Redis、回写 `MachineCode`、返回 `jwtToken` |
| `ListProjects` 输出 `accessCode` / `machineCode` | `ProjectCatalogController` 已改 | ✅ | |
| JWT Claims：`accessCode`, `machineCode`, 无 `fdBuildLicenseNo` | `BasePlatformJwtTokenGenerator` 符合 | ✅ | |
| JWT `iss = BasePlatform`（01 / 06 / 客户端） | **`iss = UrbanManagement`** | ❌ | 见 §4.1 |
| `SendAuthLicense` 5001 Redis 载荷含 `AccessCode` | 归档 tasks 已勾选 | ✅ | |
| `DownloadUrbanLicense`（Web 离线下载） | 归档 tasks 已勾选 | ✅ | 运维路径 |

### 3.2 UrbanManagement

| EPIC / V2 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `GovProject.BuildLicenseNo` → `AccessCode` | 实体 + EF 迁移已完成 | ✅ | |
| `GovProject.MachineCode` / `AuthToken` | 已新增 | ✅ | |
| `GovProject.LastMachineCodeUpdate` | **无此字段** | ❌ | EPIC §2.3；可选补列或修订 EPIC |
| `FdBuildLicenseNo` 废弃（JWT/出站） | 实体、Hub、Legacy API **仍保留** | ⚠️ | Legacy 政府路径仍用 |
| JWT 委托 `license-file`（内部） | `GovProjectLicenseAppService` | ✅ | 管理端下载 |
| **`GET /api/urban/auth/license-file`** | **无**；`GET /api/app/gov-project-license/generate` | ❌ | 客户端不直连；P2 |
| **`POST /api/urban/auth/activate`** | **无** Controller/AppService | ❌ | 见 §4.2 |
| 激活后更新本地 `GovProject.MachineCode` 等 | 无激活代理 → 未实现 | ❌ | EPIC §2.4 步骤 4 |
| `JwtAntiTamperService` / Hub `VerifyJwtAsync` | 已实现 | ✅ | |
| Hub `GetClientProjectLicenseInfo.buildLicenseNo` = `AccessCode` | 已实现 | ✅ | |
| Hub 仍返回 `FdBuildLicenseNo` | DTO 仍含字段 | ⚠️ | 客户端已不同步 |
| **`UpdateClientLicense` Hub 推送** | **Hub 无此方法** | ❌ | 客户端已注册 handler |
| `IBasePlatformAuthHttpClient` 激活方法 | **仅 `GetLicenseFileAsync`** | ❌ | 需 `ActivateUrbanAsync` 调 BasePlatform |

### 3.3 MaterialClient.Urban

| EPIC / 06 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `LicenseInfo.AccessCode`；删 `AuthToken` / `FdBuildLicenseNo` | 已实现 | ✅ | |
| `StaticLicenseChecker`：`iss=BasePlatform` | 已实现 | ✅ | 与 BasePlatform 实际 `iss` **冲突**（§4.1） |
| Refit `POST /api/urban/auth/activate` | `IUrbanAuthApi` + `ActivateUrbanAsync` | ✅ | **对端 Urban 无路由** |
| 禁止 5001 直连 BasePlatform 激活 | 已拦截 | ✅ | |
| SignalR `UpdateClientLicense` handler | 已注册 | ✅ | Urban 未推送 |
| Hub 不同步 `fdBuildLicenseNo` | 已移除同步 | ✅ | |

---

## 4. 关键缺口详述

### 4.1 P0 — JWT `iss` 三方不一致

| 来源 | `iss` |
|------|--------|
| EPIC 样例（过时） | `UrbanManagement` |
| 01 / 06 / 客户端 | **`BasePlatform`** |
| `BasePlatformJwtTokenGenerator` 现网 | **`UrbanManagement`** |

**建议**：BasePlatform 签发改为 `iss = "BasePlatform"`（OpenSpec：`fix-baseplatform-jwt-issuer`）。

---

### 4.2 P0 — Urban 未实现 `activate` 代理

**目标架构**：

```
MaterialClient.Urban --POST /api/urban/auth/activate--> UrbanManagement --POST /api/auth/activate-urban--> BasePlatform
```

**现网**：客户端 Refit 已指向 `/api/urban/auth/activate`；Urban **无**该路由；BasePlatform `activate-urban` **已有**。

**建议实现（UrbanManagement）**：

```csharp
// IBasePlatformAuthHttpClient（调 BasePlatform，路径不变）
[Post("/api/auth/activate-urban")]
Task<BasePlatformApiResponse<ActivateUrbanResponse>> ActivateUrbanAsync(...);

// 对外暴露（与 MaterialClient IUrbanAuthApi 一致）
// POST /api/urban/auth/activate
// Body: { productCode: 5001, code, machineCode }
// 逻辑：调 BasePlatform → 更新 GovProject.MachineCode 等 → 透传 data
```

**归属**：UrbanManagement OpenSpec change（建议名：`add-urban-auth-activate-proxy`）。

---

### 4.3 P1 — `license-file` REST 路径

| 能力 | EPIC 文档 | Urban 现网 | 客户端调用 |
|------|-----------|------------|------------|
| 离线授权文件 | `GET /api/urban/auth/license-file` | `GET /api/app/gov-project-license/generate` | 否 |

**建议**：P2 文档统一或增加别名路由；**不阻断** `activate` 闭环。

---

### 4.4 P1 — `UpdateClientLicense`（可选）

客户端已订阅；Urban Hub **未实现**推送。不阻断首次在线激活。

---

### 4.5 P2 — `FdBuildLicenseNo` / `LastMachineCodeUpdate`

见 v0.1 分析；Legacy 与政府路径另立 change，本拟稿不扩大范围。

---

## 5. 统一接口契约（收口版）

### 5.1 MaterialClient.Urban → UrbanManagement

| 方法 | 路径 | 请求 | 成功响应 `data` |
|------|------|------|-----------------|
| POST | **`/api/urban/auth/activate`** | `{ productCode: 5001, code, machineCode }` | `jwtToken`, `proId`, `proName`, `accessCode`, `authEndDate` |

- 客户端 **禁止** 直连 BasePlatform `/api/auth/activate-urban`。
- C# 方法名可仍为 `ActivateUrbanAsync`（实现细节，与 HTTP 路径无关）。

### 5.2 UrbanManagement → BasePlatform.PublicApi（内部）

| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/auth/activate-urban` | 在线激活（Urban 透传） |
| GET | `/api/auth/license-file` | 离线 JWT |

### 5.3 JWT（5001）

| 项 | 值 |
|----|-----|
| `iss` | **`BasePlatform`**（待 BasePlatform 代码修改） |
| `aud` | `MaterialClient.Urban` |
| 必填 claims | `proId`, `proName`, `accessCode`, `machineCode`, `exp`, `jti` |

### 5.4 SignalR（可选）

`VerifyJwtAsync`、`GetClientProjectLicenseInfo` 已实现；`UpdateClientLicense` Urban 待实现。

---

## 6. 建议实施顺序（开发侧）

| 步骤 | 仓库 | 工作 | 预估 |
|------|------|------|------|
| 1 | BasePlatform | `iss` → `BasePlatform` | 0.25d |
| 2 | UrbanManagement | `IBasePlatformAuthHttpClient.ActivateUrbanAsync` | 0.25d |
| 3 | UrbanManagement | `POST /api/urban/auth/activate` 代理 + `GovProject` 更新 | 1d |
| 4 | UrbanManagement（可选） | `UpdateClientLicense` | 0.5d |
| 5 | 文档 | 同步 06 / EPIC 路径表 | 0.25d |

**联调与发版验收**：步骤 1–3 完成后由 **用户** 执行 §7 清单并自行勾选 OpenSpec tasks §7；本拟稿不将勾选状态列为缺口。

---

## 7. 用户验收参考（联调与测试）

> **说明**：以下清单供 **用户** 在目标环境联调使用。OpenSpec `update-materialclient-urban-accesscode-jwt` tasks §7 由用户勾选；**未勾选不代表代码未实现**。

| # | 场景 | 预期 |
|---|------|------|
| 1 | BasePlatform 签发 JWT | `iss=BasePlatform`；含 `accessCode`、`machineCode` |
| 2 | 客户端离线导入 `.urban` | 启动验签通过；`LatestJwtToken` 已写入 |
| 3 | 客户端 `POST /api/urban/auth/activate` | Urban 代理 200；`LatestJwtToken` 有值 |
| 4 | 激活后 Urban `GovProject.MachineCode` | 与请求一致或 Pull 后一致 |
| 5 | JWT `iss=UrbanManagement` | 客户端拒绝 |
| 6 | Hub `VerifyJwtAsync` | `ServerJwt` 覆盖本地 |
| 7 | Hub `GetClientProjectLicenseInfo` | `buildLicenseNo` → 客户端 `AccessCode` |
| 8 | `UpdateClientLicense`（若 Urban 已实现） | 推送后客户端更新成功 |
| 9 | 5000 产品 | 仍走 `GetAuthClientLicense`；无 `jwtToken` |

---

## 8. 与现有拟稿 / OpenSpec 的关系

| 文档 / Change | 关系 |
|---------------|------|
| [00-EPIC](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) | Urban 对外路径 **`activate`** 与 EPIC 一致；需修订 `iss` 样例 |
| [04-UrbanManagement](./04-UrbanManagement迁移拟稿提案.md) | 已使用 `activate`；与本文一致 |
| [06-MaterialClient.Urban](./06-MaterialClient.Urban迁移拟稿提案.md) | 需将 Refit 路径从 `activate-urban` 同步为 **`activate`** |
| Urban V2 归档 | JWT 委托 ✅；**`activate` 代理未做** |
| `update-materialclient-urban-accesscode-jwt` | 客户端 Refit 已改为 `/api/urban/auth/activate` |

**建议 Monospec change**：

1. `fix-baseplatform-jwt-issuer`
2. `add-urban-auth-activate-proxy`（对外 `activate`，对内 `activate-urban`）

---

## 9. 非目标

- BasePlatform DDL / 授权后台 UI（02）
- MaterialClient 主程序 5000/5010
- `POST /api/urban/auth/verify`
- 本拟稿 **不包含** 联调执行与 tasks 勾选（用户负责）

---

## 10. 文档索引

| 编号 | 文档 |
|------|------|
| 06 | [06-MaterialClient.Urban迁移拟稿提案](./06-MaterialClient.Urban迁移拟稿提案.md) |
| 07 | **本文档** |
| EPIC | [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) |

---

**文档版本**：0.2（`activate` 路径；联调归用户）  
**最后更新**：2026-06-26
