# 客户端与服务端 EPIC 缺口对齐拟稿提案

> **文档类型**：拟稿提案（Draft Proposal）  
> **创建日期**：2026-06-26  
> **状态**：拟稿  
> **范围**：`MaterialClient.Urban` ↔ `UrbanManagement` ↔ `BasePlatform`（**仅 ProductCode 5001**）  
> **对照基线**：[00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)  
> **前置**：[01-解决方案](./01-解决方案.md) · [05-联合发版说明](./05-联合发版说明.md) · [06-MaterialClient.Urban迁移拟稿提案](./06-MaterialClient.Urban迁移拟稿提案.md)

---

## 0. 文档定位

| 维度 | 说明 |
|------|------|
| **盘点范围** | 三仓端到端契约（对照 EPIC） |
| **剩余开发主体** | **主要为 UrbanManagement**（`activate` 代理、Hub 推送等） |
| **BasePlatform** | `iss` 已对齐（见 §4.1）；其余项在 02/03 归档中已完成 |
| **MaterialClient** | tasks §1–6 已实现；无新增开发项，等待 Urban 对端 |

本拟稿 **不是** 仅 UrbanManagement 的单仓实施正文（那是 [04](./04-UrbanManagement迁移拟稿提案.md) 与 Monospec Urban V2 归档的职责），而是跨仓缺口索引；**当前阻塞在线激活闭环的 P0 仅剩 Urban 侧 `activate` 代理**。

---

## 1. 提案摘要

各仓库已按 [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)～[06](./06-MaterialClient.Urban迁移拟稿提案.md) 与 Urban V2 归档变更分头推进。本拟稿汇总 **2026-06-26 起** 的**实现缺口**（不含联调勾选状态）。

| 优先级 | 主题 | 归属 | 状态 |
|--------|------|------|------|
| ~~**P0**~~ | ~~JWT `iss` 三方不一致~~ | BasePlatform | **✅ 已修复**（§4.1） |
| **P0** | Urban 未暴露 `activate` 代理 | UrbanManagement | ❌ 待实现 |
| **P1** | REST 路径与 EPIC 不一致（`license-file`） | UrbanManagement | ❌ P2，非阻断 |
| **P1** | Hub `UpdateClientLicense` 仅客户端订阅 | UrbanManagement | ❌ 可选 |
| **P2** | `FdBuildLicenseNo` / Hub 字段未按 EPIC 废弃 | UrbanManagement | ⚠️ |
| **P2** | `GovProject.LastMachineCodeUpdate` 未落地 | UrbanManagement / EPIC | ❌ 可选 |

> **联调与测试**：端到端联调、OpenSpec tasks §7 勾选及发版验收 **由用户负责**；tasks 未勾选 **不作为**本拟稿的缺口判定依据（见 §7）。

---

## 2. 对照方法

- **设计基线**：EPIC + [01-解决方案](./01-解决方案.md)
- **对外 Urban 激活路径**：**`POST /api/urban/auth/activate`**（Urban 内部转发 BasePlatform `POST /api/auth/activate-urban`）
- **代码基线**：
  - `MaterialClient`：`update-materialclient-urban-accesscode-jwt`（tasks §1–6）
  - `UrbanManagement`：Urban V2 归档（JWT 委托等；**缺 `activate` 代理**）
  - `BasePlatform`：`BasePlatformJwtTokenGenerator` **`iss=BasePlatform`**（2026-06-26 已改）

图例：**✅ 已对齐** · **⚠️ 部分对齐** · **❌ 未实现**

---

## 3. 端到端契约对照矩阵

### 3.1 BasePlatform.PublicApi

| EPIC / 01 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `GET /api/auth/license-file`（5001） | `AuthController` 已实现 | ✅ | |
| `POST /api/auth/activate-urban`（Urban 内部调用） | `AuthController` 已实现 | ✅ | |
| `ListProjects` 输出 `accessCode` / `machineCode` | `ProjectCatalogController` 已改 | ✅ | |
| JWT Claims：`accessCode`, `machineCode`, 无 `fdBuildLicenseNo` | `BasePlatformJwtTokenGenerator` | ✅ | |
| JWT `iss = BasePlatform` | `JwtIssuer = "BasePlatform"` | ✅ | `FdSoft.BasePlatform.Service` |
| `SendAuthLicense` 5001 Redis 载荷含 `AccessCode` | 归档 tasks 已勾选 | ✅ | |
| `DownloadUrbanLicense`（Web 离线下载） | 归档 tasks 已勾选 | ✅ | 运维路径 |

**BasePlatform 侧：无剩余 P0 开发项。**

### 3.2 UrbanManagement（剩余工作集中于此）

| EPIC / V2 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `GovProject.AccessCode` / `MachineCode` / `AuthToken` | 已实现 | ✅ | |
| `GovProject.LastMachineCodeUpdate` | **无此字段** | ❌ | 可选补列或修订 EPIC |
| `FdBuildLicenseNo` 废弃（JWT/出站） | Legacy 仍保留 | ⚠️ | 另立 change |
| JWT 委托 `license-file`（内部） | `GovProjectLicenseAppService` | ✅ | |
| **`GET /api/urban/auth/license-file`** | ABP `gov-project-license/generate` | ❌ | 客户端不直连；P2 |
| **`POST /api/urban/auth/activate`** | **无** | ❌ | **P0**，见 §4.2 |
| 激活后更新 `GovProject.MachineCode` 等 | 未实现 | ❌ | 依赖 `activate` 代理 |
| Hub `VerifyJwtAsync` / `GetClientProjectLicenseInfo` | 已实现 | ✅ | |
| **`UpdateClientLicense` Hub 推送** | **无** | ❌ | P1 可选 |
| `IBasePlatformAuthHttpClient` 激活方法 | 仅 `GetLicenseFileAsync` | ❌ | 需 `ActivateAsync` |

### 3.3 MaterialClient.Urban

| EPIC / 06 设计 | 现网实现 | 状态 | 说明 |
|----------------|----------|------|------|
| `LicenseInfo.AccessCode`；删 `AuthToken` / `FdBuildLicenseNo` | 已实现 | ✅ | |
| `StaticLicenseChecker`：`iss=BasePlatform` | 已实现 | ✅ | 与 BasePlatform 签发一致 |
| Refit `POST /api/urban/auth/activate` | `IUrbanAuthApi` | ✅ | **对端 Urban 无路由** |
| 禁止 5001 直连 BasePlatform 激活 | 已拦截 | ✅ | |
| SignalR `UpdateClientLicense` handler | 已注册 | ✅ | 等待 Urban 推送 |
| Hub 不同步 `fdBuildLicenseNo` | 已移除 | ✅ | |

**MaterialClient 侧：无新增开发项；阻塞在 Urban `activate` 与联调。**

---

## 4. 关键缺口详述

### 4.1 ~~P0~~ JWT `iss` — **已解决**

| 来源 | `iss` |
|------|--------|
| 01 / 06 / 客户端 / Urban 验签 | **`BasePlatform`** |
| `BasePlatformJwtTokenGenerator`（2026-06-26 起） | **`BasePlatform`** |

**变更**：`FdSoft.BasePlatform.Service/BasePlatform/BasePlatformJwtTokenGenerator.cs` — `JwtIssuer = "BasePlatform"`。

**注意**：联调环境若仍有旧 JWT（`iss=UrbanManagement`），须重新签发 `.urban` 或重新在线激活。

---

### 4.2 P0 — Urban 未实现 `activate` 代理

**目标架构**：

```
MaterialClient.Urban --POST /api/urban/auth/activate--> UrbanManagement --POST /api/auth/activate-urban--> BasePlatform
```

**现网**：客户端 Refit 已就绪；Urban **无** `/api/urban/auth/activate`；BasePlatform `activate-urban` **已有**。

**建议实现（UrbanManagement）**：

```csharp
// IBasePlatformAuthHttpClient（调 BasePlatform，路径不变）
[Post("/api/auth/activate-urban")]
Task<BasePlatformApiResponse<ActivateUrbanResponse>> ActivateAsync(...);

// 对外：POST /api/urban/auth/activate
// Body: { productCode: 5001, code, machineCode }
// 逻辑：调 BasePlatform → 更新 GovProject.MachineCode 等 → 透传 data
```

> **命名**：Refit/Service 方法为 `ActivateAsync`；BasePlatform 控制器动作名仍为 `ActivateUrban`（路径 `/api/auth/activate-urban`）。

**归属**：UrbanManagement OpenSpec change（建议名：`add-urban-auth-activate-proxy`）。

---

### 4.3 P1 — `license-file` REST 路径（非阻断）

| 能力 | EPIC 文档 | Urban 现网 | 客户端调用 |
|------|-----------|------------|------------|
| 离线授权文件 | `GET /api/urban/auth/license-file` | `GET /api/app/gov-project-license/generate` | 否 |

---

### 4.4 P1 — `UpdateClientLicense`（可选）

Urban Hub **未实现**推送；不阻断首次在线激活。

---

### 4.5 P2 — `FdBuildLicenseNo` / `LastMachineCodeUpdate`

Legacy 与政府路径另立 change；本拟稿不扩大范围。

---

## 5. 统一接口契约（收口版）

### 5.1 MaterialClient.Urban → UrbanManagement

| 方法 | 路径 | 请求 | 成功响应 `data` |
|------|------|------|-----------------|
| POST | **`/api/urban/auth/activate`** | `{ productCode: 5001, code, machineCode }` | `jwtToken`, `proId`, `proName`, `accessCode`, `authEndDate` |

### 5.2 UrbanManagement → BasePlatform.PublicApi（内部）

| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/auth/activate-urban` | 在线激活（Urban 透传） |
| GET | `/api/auth/license-file` | 离线 JWT |

### 5.3 JWT（5001）

| 项 | 值 |
|----|-----|
| `iss` | **`BasePlatform`** |
| `aud` | `MaterialClient.Urban` |
| 必填 claims | `proId`, `proName`, `accessCode`, `machineCode`, `exp`, `jti` |

### 5.4 SignalR（可选）

`VerifyJwtAsync`、`GetClientProjectLicenseInfo` 已实现；`UpdateClientLicense` Urban 待实现。

---

## 6. 建议实施顺序（开发侧）

| 步骤 | 仓库 | 工作 | 状态 |
|------|------|------|------|
| ~~1~~ | ~~BasePlatform~~ | ~~`iss` → `BasePlatform`~~ | **✅ 已完成** |
| 2 | UrbanManagement | `IBasePlatformAuthHttpClient.ActivateAsync` | 待做 |
| 3 | UrbanManagement | `POST /api/urban/auth/activate` 代理 + `GovProject` 更新 | 待做 |
| 4 | UrbanManagement（可选） | `UpdateClientLicense` | 待做 |
| 5 | 文档 | 修订 EPIC `iss` 样例；归档本拟稿 | 待做 |

**联调与发版验收**：步骤 2–3 完成后由 **用户** 执行 §7；本拟稿不将 tasks 勾选状态列为缺口。

---

## 7. 用户验收参考（联调与测试）

> **说明**：供 **用户** 在目标环境联调使用。OpenSpec tasks §7 由用户勾选；**未勾选不代表代码未实现**。

| # | 场景 | 预期 |
|---|------|------|
| 1 | BasePlatform 签发 JWT | `iss=BasePlatform`；含 `accessCode`、`machineCode` |
| 2 | 客户端离线导入 `.urban` | 启动验签通过；`LatestJwtToken` 已写入 |
| 3 | 客户端 `POST /api/urban/auth/activate` | Urban 代理 200；`LatestJwtToken` 有值 |
| 4 | 激活后 Urban `GovProject.MachineCode` | 与请求一致或 Pull 后一致 |
| 5 | JWT `iss=UrbanManagement` | 客户端 **拒绝** |
| 6 | Hub `VerifyJwtAsync` | `ServerJwt` 覆盖本地 |
| 7 | Hub `GetClientProjectLicenseInfo` | `buildLicenseNo` → 客户端 `AccessCode` |
| 8 | `UpdateClientLicense`（若 Urban 已实现） | 推送后客户端更新成功 |
| 9 | 5000 产品 | 仍走 `GetAuthClientLicense`；无 `jwtToken` |

---

## 8. 与现有拟稿 / OpenSpec 的关系

| 文档 / Change | 关系 |
|---------------|------|
| [00-EPIC](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) | 需修订 §1.2 样例 `iss` 为 `BasePlatform` |
| [04-UrbanManagement](./04-UrbanManagement迁移拟稿提案.md) | `activate` 代理实施正文 |
| [06-MaterialClient.Urban](./06-MaterialClient.Urban迁移拟稿提案.md) | 客户端已对齐 |
| Urban V2 归档 | JWT 委托 ✅；**`activate` 代理未做** |
| `update-materialclient-urban-accesscode-jwt` | 客户端已完成 |

**建议 Monospec change（剩余）**：

1. `add-urban-auth-activate-proxy`（对外 `activate`，对内 `activate-urban`）
2. （可选）`add-urban-hub-update-client-license`

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
| 04 | [04-UrbanManagement迁移拟稿提案](./04-UrbanManagement迁移拟稿提案.md) |
| 06 | [06-MaterialClient.Urban迁移拟稿提案](./06-MaterialClient.Urban迁移拟稿提案.md) |
| 07 | **本文档** |
| EPIC | [00-EPIC-项目改动总览](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) |

---

**文档版本**：0.4（方法名 `ActivateAsync`，去除 Urban 后缀）  
**最后更新**：2026-06-26
