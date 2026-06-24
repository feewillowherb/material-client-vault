# BasePlatform JWT 签发迁移拟稿提案

> **文档类型**：拟稿提案（Draft Proposal）  
> **创建日期**：2026-06-24  
> **状态**：拟稿 — 待评审后进入 BasePlatform 仓库实施  
> **范围**：**仅** `FdSoft.BasePlatform.PublicApi`（及共享 Model / Service 层签发逻辑）  
> **不在范围**：AccessCode 库表与 `ListProjects`（→ [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)）、Urban 代理与下线（→ [04](./04-UrbanManagement迁移拟稿提案.md)）、MaterialClient.Urban

**前置**：[01-解决方案.md](./01-解决方案.md) · [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)（`AccessCode` 列为 Claims 数据源）· [05-联合发版说明.md](./05-联合发版说明.md)

**关联 EPIC**：[00-EPIC-项目改动总览.md](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) §1.3

---

## 1. 提案摘要

将 UrbanManagement 中的 **`UrbanLicenseGenerator`** 签发能力迁入 BasePlatform，使 **BasePlatform.PublicApi** 成为城管授权文件（`.urban` / JWT）的**唯一签发中心**。

| 项 | 决议 |
|----|------|
| 新 API | `GET /api/auth/license-file` |
| 签名算法 | RS256（与现网 Urban 一致） |
| Issuer / Audience | `UrbanManagement` / `MaterialClient.Urban`（**保持不变**，避免客户端验签变更） |
| Claims | `proId`, `proName`, **`accessCode`**, `fdBuildLicenseNo`, `machineCode`, `exp`, `jti` |
| 数据源 | `JC_ProductAuthority` + `JC_Project`（`AccessCode` 来自 02） |

**本提案不包含** Urban 侧代理、Hub 推送、旧签发下线（见 04）。

---

## 2. 目标与非目标

### 2.1 目标

1. 新增 `BasePlatformJwtTokenGenerator`（自 `UrbanLicenseGenerator` 移植），配置 `Jwt:PrivateKey`。
2. 实现 `AuthController`（或等价）`GET /api/auth/license-file`：按 `machineCode` + `proId` 查授权，组装 Claims 并返回 JWT 文件流或 JSON 包装。
3. Claims 中 **`accessCode`** 取自 `JC_ProductAuthority.AccessCode`（非 `MachineCode`，非 `buildLicenseNo`）。
4. `fdBuildLicenseNo` 保持 `MD5(ProId + "findongCode")` 计算规则。
5. 提供集成测试：与 Urban 现网签发样本对比 Issuer、算法、Claim 键名（值因环境而异）。

### 2.2 非目标

| 主题 | 文档 |
|------|------|
| `JC_ProductAuthority.AccessCode` DDL / 迁移 | [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md) |
| Urban `GET /api/urban/auth/license-file` 代理 | [04](./04-UrbanManagement迁移拟稿提案.md) |
| 客户端公钥分发、`LicenseInfo` 重命名 | [EPIC](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md) |
| 在线激活 `POST /api/auth/activate` | EPIC 其他条目（可后续单独立项） |

### 2.3 与 02 的接口约定

- **硬依赖**：P2 上线前须完成 02 的 P0（`AccessCode` 列有值）；否则 Claims 中 `accessCode` 为空。
- **软并行**：03 代码可与 02 同仓开发；JWT 联调须运营已在后台维护 `AccessCode`（或测试数据手工写入）。

---

## 3. 涉及仓库与项目

| 路径 | 项目 | 改动类型 |
|------|------|----------|
| `FdSoft.BasePlatform.PublicApi` | PublicApi | 新 Controller / 端点 |
| `FdSoft.BasePlatform.Service` 或独立 Auth 模块 | Service | `BasePlatformJwtTokenGenerator` |
| `FdSoft.BasePlatform.Model` | Model | `LicenseFileRequestDto` 等（若尚未存在） |
| `appsettings` / 密钥管理 | 配置 | `Jwt:PrivateKey`（自 Urban 迁移） |

---

## 4. API 设计

### 4.1 端点

**`GET /api/auth/license-file`**

**Query / Body**（与 EPIC 对齐，实现时二选一并文档化）：

```json
{
  "productCode": "5001",
  "machineCode": "MACHINE-CODE-12345",
  "proId": "project-guid",
  "authEndDate": "2026-12-31T23:59:59"
}
```

**响应（方案 A — 文件流，与 Urban 现行为近）**：

```
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="license.urban"

<JWT_TOKEN_UTF8>
```

**响应（方案 B — JSON 包装，便于 Urban 代理解析）**：

```json
{
  "success": true,
  "data": {
    "jwtToken": "<JWT>",
    "proId": "...",
    "proName": "...",
    "authEndDate": "2026-12-31T23:59:59"
  }
}
```

> **拟稿决议**：PublicApi **同时支持** B（Urban 代理默认）+ 可选 `?format=stream` 走 A；评审时确认一种为主。

### 4.2 鉴权

- 服务间调用：API Key / 内网 IP 白名单（与现有 PublicApi 惯例一致）。
- **不**对终端设备开放直连接口（设备仍经 Urban 或离线文件）。

### 4.3 业务校验

1. `JC_ProductAuthority` 存在且 `ProId` + `ProductCode` 匹配。
2. `MachineCode` 与请求一致（或允许运营预绑后首次激活场景 — 与现 Urban 规则对齐）。
3. `AuthEndDate` 不晚于库中 `AuthEndTime`。
4. `AccessCode` 非空（02 迁移后）；为空时返回 4xx 并记日志。

---

## 5. JWT 签发实现（拟稿）

### 5.1 Generator

```csharp
public sealed class BasePlatformJwtTokenGenerator
{
    private readonly RsaSecurityKey _rsaSecurityKey;

    public BasePlatformJwtTokenGenerator(IConfiguration configuration)
    {
        var privateKeyPem = configuration["Jwt:PrivateKey"]
            ?? throw new InvalidOperationException("JWT 私钥未配置");
        using var rsa = RSA.Create();
        rsa.ImportFromPem(privateKeyPem);
        _rsaSecurityKey = new RsaSecurityKey(rsa.ExportParameters(true));
    }

    public string GenerateToken(LicenseClaimsInput input)
    {
        var descriptor = new SecurityTokenDescriptor
        {
            Issuer = "UrbanManagement",
            Audience = "MaterialClient.Urban",
            Expires = input.AuthEndDate,
            SigningCredentials = new SigningCredentials(
                _rsaSecurityKey, SecurityAlgorithms.RsaSha256),
            Subject = new ClaimsIdentity([
                new Claim("proId", input.ProId.ToString()),
                new Claim("proName", input.ProName ?? ""),
                new Claim("accessCode", input.AccessCode ?? ""),
                new Claim("fdBuildLicenseNo", input.FdBuildLicenseNo ?? ""),
                new Claim("machineCode", input.MachineCode ?? ""),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            ])
        };
        var handler = new JwtSecurityTokenHandler();
        return handler.WriteToken(handler.CreateToken(descriptor));
    }
}
```

### 5.2 Claims 输入组装

| Claim | 来源 |
|-------|------|
| `proId` | 请求 / `JC_Project` |
| `proName` | `JC_Project.ProName` |
| `accessCode` | `JC_ProductAuthority.AccessCode` |
| `fdBuildLicenseNo` | `CommonHelper.GetFdBuildLicenseNo(proId)` |
| `machineCode` | 请求 `machineCode` |
| `exp` | `AuthEndDate` |

**禁止**再写入 `buildLicenseNo` claim（迁移期若客户端仍读旧键，由 EPIC 客户端改造或短期双写 — **不在本提案默认范围**）。

### 5.3 私钥迁移

1. 从 Urban `appsettings` 导出 `Jwt:PrivateKey`（运维安全通道）。
2. 写入 BasePlatform 密钥库 / 环境变量；Urban P4 后删除私钥。
3. **公钥**不变则 MaterialClient.Urban 无需换包即可验签新签发 JWT。

---

## 6. 实施步骤

| 步骤 | 工作 | 预估 |
|------|------|------|
| 1 | 移植 `UrbanLicenseGenerator` → `BasePlatformJwtTokenGenerator` + 单元测试 | 1d |
| 2 | `LicenseFile` 查询服务（授权 + 项目信息） | 0.5d |
| 3 | `GET /api/auth/license-file` + 鉴权 | 0.5d |
| 4 | 与 02 联调：`AccessCode` 入 Claims | 0.5d |
| 5 | 预发对比 Urban 旧签发 JWT 结构 | 0.5d |
| **合计** | | **~3d** |

可与 02 并行；**上线**见 [05](./05-联合发版说明.md) **P2**（Urban 仍可暂用旧签发直至 P4）。

---

## 7. 测试清单

| # | 场景 | 预期 |
|---|------|------|
| 1 | 合法 machineCode + proId | 200，JWT 可解析 |
| 2 | Claims 含 `accessCode` | 值 = DB `AccessCode` |
| 3 | `accessCode` 为空 | 4xx（02 未迁移时） |
| 4 | 错误 machineCode | 4xx |
| 5 | 过期 `AuthEndDate` | 4xx |
| 6 | RS256 验签（公钥） | 通过 |
| 7 | Issuer / Audience | 与现网 Urban 一致 |

---

## 8. 回滚

| 故障点 | 动作 |
|--------|------|
| 新 API 行为异常 | 关闭 PublicApi 端点或 feature flag；Urban 保持 `UseBasePlatformJwtIssuer=false`（04） |
| 私钥配置错误 | 回滚配置；勿在 Urban 与 BasePlatform 同时使用不同私钥签发 |
| Claims 错误 | 热修 Generator；已下发 JWT 靠 `exp` 自然过期 |

---

## 9. 风险与依赖

| 风险 | 缓解 |
|------|------|
| 02 未完成导致 `accessCode` 空 | P2 与 P0 绑定；签发前校验 |
| Issuer/Audience 变更导致客户端验签失败 | 严格保持现网值 |
| 私钥泄露 | 密钥仅驻留 BasePlatform；审计访问 |
| Urban 与 BasePlatform 双签发并存 | 04 feature flag + P4 统一切换 |

**下游依赖**：[04](./04-UrbanManagement迁移拟稿提案.md) P4 前须保留 Urban 旧签发回退路径。

---

## 10. 拟稿评审检查项

- [ ] 安全：私钥存储方式与轮换预案
- [ ] 与 Urban 现网 JWT 样本逐 Claim 对比
- [ ] `productCode` 枚举（5001 / UrbanManagement 字符串）与调用方一致
- [ ] 响应格式（stream vs JSON）与 04 代理实现对齐
- [ ] P2 可与 P1 同发版还是必须晚于 P0

---

## 11. 文档索引

| 编号 | 文档 | 说明 |
|------|------|------|
| 00 | [00-问题分析.md](./00-问题分析.md) | 问题发现 |
| 01 | [01-解决方案.md](./01-解决方案.md) | 全链路总方案 |
| 02 | [02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md) | AccessCode 分列与 ListProjects |
| 03 | [03-BasePlatform-JWT签发迁移拟稿提案.md](./03-BasePlatform-JWT签发迁移拟稿提案.md) | 本文档 |
| 04 | [04-UrbanManagement迁移拟稿提案.md](./04-UrbanManagement迁移拟稿提案.md) | Urban 配合 |
| 05 | [05-联合发版说明.md](./05-联合发版说明.md) | 联合发版 |

---

**文档版本**：0.1（拟稿）  
**最后更新**：2026-06-24
