# UrbanManagement 代理授权方案 - 项目改动 EPIC

> **字段语义（已定）**：与 [AccessCode 分离方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md) 一致。  
> - **`AccessCode`**：城管接入码（`GovProject` 原 `BuildLicenseNo` 重命名）  
> - **`MachineCode`**：设备机器码  
> - ~~**`FdBuildLicenseNo`**~~：**已废弃**（不再计算、出站或写入 JWT）  
> - 政府 HTTP 出站 `buildLicenseNo` 协议名可保留，**值 = AccessCode**  
> - **JWT 适用范围**：**仅 ProductCode `5001`**（MaterialDxlt / 城管）；`5000`、`5010` 及其它产品 **不** 走 JWT，现网流程不变  
> - 详细拟稿：[03-BasePlatform-JWT签发迁移拟稿提案](../2026-06-24-buildlicenseno-machinecode-confusion/03-BasePlatform-JWT签发迁移拟稿提案.md)

## 概述

本文档作为 UrbanManagement 代理授权方案的总体 EPIC，描述所有涉及项目的改动内容。每个项目可基于此文档创建详细的 Proposal 和实施计划。

**方案架构**：`MaterialClient.Urban → UrbanManagement → BasePlatform.PublicApi`（**仅 5001 城管链路**）

**授权原则（JWT 唯一权威，仅 5001）**：

1. **离线与在线均以 JWT 为唯一权威凭证**（RS256，Claims：`proId`, `proName`, `accessCode`, `machineCode`, `exp`, `jti`）。
2. 客户端 `LicenseInfo.LatestJwtToken` 存储 JWT；`.urban` 仅为 JWT 的文件载体，导入后同样写入 `LatestJwtToken`。
3. **启动验权**：仅本地 `StaticLicenseChecker` 验签 JWT；**不实施** `POST /api/urban/auth/verify` 等平行查库验证 API。
4. **产品隔离**：JWT 签发、`jwtToken` 响应、`SendAuthLicense` 扩展载荷、离线下载 **仅 `productCode == 5001`**；其它产品 `SendAuthLicense` / `DownloadAuth` 等行为与现网一致。

**核心目标**：
- 在 GovProject 中扩展机器码授权字段
- UrbanManagement 作为代理层转发 **5001 激活与 JWT**（不签发、不 verify）
- MaterialClient.Urban 更新 LicenseInfo（`BuildLicenseNo` → `AccessCode`）与 JWT 验证机制（**5001**）
- BasePlatform.PublicApi：**5001** 在线激活签发 JWT + 离线 `license-file`
- **将 UrbanManagement 中的 JWT 签发流程移植到 BasePlatform**（**仅 5001**）
- **在 BasePlatform 管理后台提供 5001 离线授权文件下载**（见拟稿 03）

---

## 5001 双路径（轻量）

| 路径 | 运营/现场 | 服务端 | 客户端落库 |
|------|-----------|--------|------------|
| **离线** | 目标机脚本采码 → 授权页录入 `MachineCode` →「下载授权」 | `DownloadUrbanLicense` / `license-file` 签发 JWT | 导入 `.urban` → `LatestJwtToken` |
| **在线** |「生成授权码」`SendAuthLicense` → 用户输入码 | Urban `activate` 代理 → BasePlatform 验 Redis → **签发 JWT** | 激活响应 `jwtToken` → `LatestJwtToken` |

SignalR 推送 JWT 为 **可选增强**（续期/换发），**不是**在线激活获得 JWT 的前置条件。

---

## 涉及项目及改动清单

### 1. BasePlatform.PublicApi

**项目路径**：`FdSoft.BasePlatform.PublicApi`

**改动内容**：

#### 1.1 授权码验证 API（已有，5001 在线激活扩展）

- **接口**：`POST /api/AuthClientLicense/GetAuthClientLicense`（现网）；**5001 在线**建议扩展或新增 `POST /api/auth/activate-urban`（名称可评审）
- **现网能力**（**所有产品**）：验证 Redis 一次性授权码，返回授权信息 JSON 字符串
- **5001 扩展**（**仅 5001**）：验码成功后  
  1. 以请求 `machineCode` 回写 `JC_ProductAuthority.MachineCode`（在线激活绑定）  
  2. 调用共用 `ILicenseFileAppService` 签发 JWT  
  3. 响应增加 **`jwtToken`**
- **非 5001**：保持现网返回结构，**不**签发、**不**返回 `jwtToken`

**在线激活请求**（Urban 代理转发，仅 5001）：

```json
{
  "productCode": "5001",
  "code": "1234",
  "machineCode": "MACHINE-CODE-12345"
}
```

**5001 在线激活响应**：

```json
{
  "success": true,
  "data": {
    "jwtToken": "<JWT>",
    "proId": "project-guid",
    "proName": "项目名称",
    "accessCode": "...",
    "authEndDate": "2026-12-31T23:59:59"
  }
}
```

#### ~~1.2 机器码验证 API~~（不实施）

~~`POST /api/auth/verify`~~ — **废弃**。JWT 为准后，日常验权由客户端本地验签完成，无需平行 verify API。

#### 1.2 JWT 授权文件生成与下载 API（新增，仅 5001，从 UrbanManagement 移植）

**背景**：将 UrbanManagement 的 JWT 签发迁入 BasePlatform，**仅服务 ProductCode 5001**。

**接口**：`GET /api/auth/license-file`

**描述**：生成并下载 JWT 授权文件（`.urban`）

**请求参数**：
```json
{
  "productCode": "5001",
  "machineCode": "MACHINE-CODE-12345",
  "proId": "project-guid",
  "authEndDate": "2026-12-31T23:59:59"
}
```

**门禁**：`productCode != 5001` 时 **4xx**；库中 `MachineCode` 为空时 **4xx**（须先录入现场脚本机器码）。

**响应**：
```
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="license.urban"

<JWT_TOKEN_CONTENT>
```

**实现要点**：
- 使用 RS256 算法签名 JWT（与 UrbanManagement 保持一致）
- JWT Claims 包含：`proId`, `proName`, `accessCode`, `exp`, `machineCode`（**不含** `fdBuildLicenseNo`）
- 私钥配置：`Jwt:PrivateKey`（从 UrbanManagement 移植）
- 公钥分发给 MaterialClient.Urban 客户端（用于验证）
- 返回的 JWT 文件可直接用作 .urban 文件

**数据模型**：
```csharp
public class LicenseFileRequestDto
{
    public string ProductCode { get; set; }
    public string MachineCode { get; set; }
    public Guid ProId { get; set; }
    public DateTime AuthEndDate { get; set; }
}

public class LicenseFileResponseDto
{
    public string JwtToken { get; set; }
    public string ProId { get; set; }
    public string ProName { get; set; }
    public DateTime AuthEndDate { get; set; }
}
```

**JWT 签名服务**（从 UrbanManagement 移植）：
```csharp
public class BasePlatformJwtTokenGenerator
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

    public string GenerateLicenseToken(LicenseFileRequestDto request)
    {
        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Issuer = "UrbanManagement",
            Audience = "MaterialClient.Urban",
            Expires = request.AuthEndDate,
            SigningCredentials = new SigningCredentials(_rsaSecurityKey, SecurityAlgorithms.RsaSha256),
            Subject = new ClaimsIdentity([
                new Claim("proId", request.ProId.ToString()),
                new Claim("proName", ""),
                new Claim("accessCode", ""),
                new Claim("machineCode", ""),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            ])
        };

        var handler = new JwtSecurityTokenHandler();
        var token = handler.CreateToken(tokenDescriptor);
        return handler.WriteToken(token);
    }
}
```

> **说明**：`ILicenseFileAppService` / `BasePlatformJwtTokenGenerator` 供 **离线下载**、**在线激活（5001）**、Urban `license-file` 代理共用。详见 [03 拟稿](../2026-06-24-buildlicenseno-machinecode-confusion/03-BasePlatform-JWT签发迁移拟稿提案.md)。

#### 1.3 其它产品（非 5001）

| ProductCode | 授权方式 | 本 EPIC JWT 改动 |
|-------------|----------|------------------|
| **5000** MaterialClient | `DownloadAuth` → `mlic.lic`；`SetCorpAuthMachineCode` | **不涉及** |
| **5010** 等其它 | 现网 `SendAuthLicense` / `DownloadAuth` | **不涉及 JWT** |
| **5001** MaterialDxlt | JWT 离线 + 在线 | **本 EPIC 范围** |

---

### 2. UrbanManagement

**项目路径**：`FdSoft.UrbanManagement` 或 `UrbanManagement`

**改动内容**：

#### 2.1 移除 JWT 签发功能（已移植到 BasePlatform）

**移除内容**：
- `UrbanLicenseGenerator` 服务（已移植到 BasePlatform）
- `GovProjectLicenseAppService` 中的授权文件生成功能
- JWT 私钥配置（已移到 BasePlatform）

**说明**：JWT 签发统一由 BasePlatform 负责，UrbanManagement 不再签发授权文件。

#### 2.2 新增授权文件下载 API（代理）

**接口**：`GET /api/urban/auth/license-file`

**描述**：代理从 BasePlatform 下载授权文件

**请求参数**：
```json
{
  "machineCode": "MACHINE-CODE-12345"
}
```

**实现逻辑**：
1. 调用 BasePlatform.PublicApi `/api/auth/license-file`
2. 转发响应给客户端
3. 缓存授权文件（可选）

#### 2.3 GovProject 实体扩展

```csharp
// UrbanManagement.GovProject 实体扩展
public class GovProject : Entity<Guid>
{
    // 现有字段（保留）
    public string ProName { get; set; } = default!;
    public string? AccessCode { get; set; }            // 城管接入码（原 BuildLicenseNo，列重命名）
    public DateTime? AuthEndTime { get; set; }         // 授权结束时间（已有）
    public DateTime? AddTime { get; set; }             // 添加时间（已有）

    // ===== 新增：机器码授权字段 =====
    public string? MachineCode { get; set; }           // 绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌（GUID）
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码最后更新时间
}
```

**字段说明**：
- `MachineCode` - 客户端机器码，用于绑定特定设备
- `AuthToken` - BasePlatform 授权令牌（GUID），由 BasePlatform 返回
- `LastMachineCodeUpdate` - 机器码最后更新时间，用于追踪变更
- `AuthEndTime` - 现有字段，表示授权结束时间，由 BasePlatform 返回

**不需要的字段**：
- ~~`AuthStatus`~~ - 授权状态应由 BasePlatform 的 JCProductAuthority 表管理，GovProject 不需要
- ~~`AuthBeginDate`~~ - 授权开始时间在 UrbanManagement 业务场景中不需要
- ~~`AuthType`~~ - 授权类型（离线/在线）应由 BasePlatform 管理，UrbanManagement 作为代理层不需要区分

**数据库迁移**：
```sql
-- 添加新字段到 GovProject 表
-- GovProject：BuildLicenseNo 列重命名为 AccessCode（SQLite/SQL Server 语法按环境调整）
-- ALTER TABLE GovProject RENAME COLUMN BuildLicenseNo TO AccessCode;

ALTER TABLE GovProject ADD COLUMN MachineCode NVARCHAR(128) NULL;
ALTER TABLE GovProject ADD COLUMN AuthToken UNIQUEIDENTIFIER NULL;
ALTER TABLE GovProject ADD COLUMN LastMachineCodeUpdate DATETIME2 NULL;

-- 添加索引
CREATE INDEX IDX_GovProject_MachineCode ON GovProject(MachineCode);
CREATE INDEX IDX_GovProject_AuthToken ON GovProject(AuthToken);
```

#### 2.4 授权码激活代理 API（仅 5001）

**接口**：`POST /api/urban/auth/activate`

**描述**：MaterialClient.Urban（**5001**）在线激活；Urban **纯代理**，不签发 JWT。

**请求参数**：
```json
{
  "code": "1234",
  "machineCode": "MACHINE-CODE-12345"
}
```

> **注意**：客户端在激活时不知道 ProId；`productCode` 固定为 **5001**（Urban 转发时写入）。

**响应**：
```json
{
  "success": true,
  "data": {
    "jwtToken": "<JWT>",
    "authEndDate": "2026-12-31T23:59:59",
    "proId": "project-guid",
    "proName": "项目名称",
    "accessCode": "..."
  }
}
```

**实现逻辑**：
1. 接收授权码、客户端上报的 `machineCode`
2. 转发 BasePlatform（`activate-urban` 或扩展后的验码接口），`productCode = 5001`
3. BasePlatform：验 Redis 一次性码 → 回写 `MachineCode` → **签发 JWT**
4. 更新本地 `GovProject`（`MachineCode`、`AuthToken` 等副本）
5. 将 **`jwtToken`** 原样返回客户端

#### ~~2.5 本地验证 API~~（不实施）

~~`POST /api/urban/auth/verify`~~ — **废弃**。启动与运行期以客户端 **本地 JWT 验签** 为准。

#### 2.5 SignalR DeviceStatusHub（可选，JWT 续期）

**Hub 方法**：`UpdateClientLicense`

**描述**：向客户端推送 **更新** 的 JWT（续期、换发）；**不替代**激活响应中的首次 `jwtToken`。

**优先级**：**中**（可选）；在线激活成功后客户端应已持有 `LatestJwtToken`。

**推送数据**：
```csharp
public class ClientLicenseUpdateDto
{
    public string ProId { get; set; }
    public string? ProName { get; set; }
    public string? AccessCode { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string JwtToken { get; set; }
}
```

---

### 3. MaterialClient.Urban

**项目路径**：`MaterialClient.Urban`

**改动内容**：

#### 3.1 LicenseInfo 结构（AccessCode 重命名）

**变更**：`BuildLicenseNo` 属性重命名为 **`AccessCode`**；**移除 `AuthToken`**（客户端不持久化服务端授权令牌）；须写入 **`LatestJwtToken`**。
```csharp
[Table("LicenseInfo")]
public class LicenseInfo : Entity<Guid>
{
    public Guid ProjectId { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string? ProName { get; set; }
    public string? AccessCode { get; set; }
    public string? LatestJwtToken { get; set; }  // 权威 JWT（在线激活或导入 .urban）
    public string MachineCode { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public bool IsExpired => DateTime.Now > AuthEndTime;
}
```

#### 3.2 授权激活流程

**新增/修改服务**：`UrbanAuthService`

```csharp
public class UrbanAuthService
{
    private readonly IUrbanManagementApi _urbanApi;
    private readonly ILicenseService _licenseService;

    /// <summary>
    /// 在线激活
    /// </summary>
    public async Task<bool> ActivateOnline(string authCode)
    {
        var machineCode = _machineCodeService.GetMachineCode();

        var response = await _urbanApi.ActivateProxy(new ActivateProxyRequest
        {
            Code = authCode,
            MachineCode = machineCode
        });

        if (!response.Success)
        {
            MessageBox.Show($"激活失败：{response.Message}", "错误",
                MessageBoxButton.OK, MessageBoxImage.Error);
            return false;
        }

        // 激活成功：持久化 JWT（权威凭证）及元数据
        await _licenseService.SaveLatestJwtTokenAsync(response.Data.JwtToken);
        await _licenseService.SyncProjectFieldsFromServerAsync(
            response.Data.ProId,
            response.Data.ProName,
            response.Data.AccessCode ?? "",
            response.Data.AuthEndDate
        );

        MessageBox.Show("激活成功！", "成功",
            MessageBoxButton.OK, MessageBoxImage.Information);
        return true;
    }
}
```

#### 3.3 启动时验证流程（JWT 唯一权威）

- 使用 `StaticLicenseChecker` 验证 **JWT** 签名与 Claims（`machineCode`、`exp` 等）
- 优先使用 `LatestJwtToken`；若为 null 则回退到已导入的 `.urban` 文件
- **不**调用 `POST /api/urban/auth/verify` 作为启动门禁

离线导入与在线激活 **共用同一套** JWT 验签逻辑。

---

### 4. BasePlatform 管理后台（FdSoft.BasePlatform）

**项目路径**：`FdSoft.BasePlatform`（Web 宿主）

**改动内容**（详见 [02](../2026-06-24-buildlicenseno-machinecode-confusion/02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)、[03](../2026-06-24-buildlicenseno-machinecode-confusion/03-BasePlatform-JWT签发迁移拟稿提案.md)）：

#### 4.1 生成授权码（`SendAuthLicense`，现网保留）

- **所有产品**：现网逻辑保留（Redis 一次性码、48h TTL）
- **仅 5001**：Redis 载荷 **可增加** `AccessCode`（供激活后 JWT Claims）；**非 5001 载荷结构与现网完全一致**

#### 4.2 离线下载（仅 5001，新增）

- `GET .../DownloadUrbanLicense`：签发 JWT → `license.urban`
- **前置**：库中 `MachineCode` 已录入（现场脚本）
- **5000 / 其它**：仍走 `DownloadAuth`（`mlic.lic` 等），**不变**

---

## 关键技术决策

### 1. 5001 在线激活时 BasePlatform 回写 MachineCode

**决议**（**仅 ProductCode 5001**）：在线激活验码成功后，BasePlatform **回写** `JC_ProductAuthority.MachineCode`（取客户端上报值），并签发 JWT。  
**非 5001** 产品：不改动现网 `GetAuthClientLicense` 行为。

UrbanManagement 仍在 `GovProject` 保留 `MachineCode` 副本，供 Pull / 政府出站等 Urban 域逻辑使用。

---

### 2. 客户端激活时不提供 ProId

**原因**：客户端在激活时只知道授权码，不知道 ProId。

**解决方案**：
- 激活请求只包含：`{ code, machineCode }`
- ProId 由服务器端根据授权码匹配后返回
- UrbanManagement 根据 BasePlatform 返回的 ProId 更新 GovProject

### 3. JWT 为唯一权威凭证（废弃平行 verify）

**决议**：
- 离线与在线均以 BasePlatform 签发的 JWT 为权威；客户端 `LatestJwtToken` / `.urban` 为存储载体
- **不实施** `POST /api/auth/verify`、`POST /api/urban/auth/verify`
- Claims 使用 **`accessCode`**（非废弃的 `fdBuildLicenseNo`）

### 4. JWT 推送机制（可选）

**现状**：MaterialClient 已有 `StaticLicenseChecker`。

**增强**（中优先级，可选）：
- UrbanManagement 通过 SignalR 推送 **更新** 的 JWT
- 客户端覆盖 `LicenseInfo.LatestJwtToken`
- **在线激活成功时须已写入 JWT**；Hub 仅用于续期，非首次激活前置

### 5. 产品隔离（非 5001 零改动）

| 能力 | 5001 | 5000 / 5010 / 其它 |
|------|------|----------------------|
| JWT 签发 | ✅ | ❌ |
| `SendAuthLicense` Redis 加 `AccessCode` | ✅ 可选 | ❌ 保持现网 JSON |
| `DownloadUrbanLicense` | ✅ | ❌ 仍 `DownloadAuth` |
| 在线激活返回 `jwtToken` | ✅ | ❌ 现网验码逻辑 |

---

## 实施优先级

### 高优先级（核心功能，5001）

1. **BasePlatform JWT 签发**（`ILicenseFileAppService`、`license-file`、`DownloadUrbanLicense`）— 见 [03 拟稿](../2026-06-24-buildlicenseno-machinecode-confusion/03-BasePlatform-JWT签发迁移拟稿提案.md)
2. **BasePlatform 5001 在线激活**（验码 + 回写 `MachineCode` + 响应 `jwtToken`）
3. **UrbanManagement**：`GovProject` 扩展 + `POST /api/urban/auth/activate` 代理（透传 `jwtToken`）
4. **MaterialClient.Urban**：激活 UI → 写入 `LatestJwtToken`；启动 JWT 验签
5. **AccessCode 分列**（02 拟稿 P0）— JWT Claims 数据源

### 中优先级（增强功能）

6. **UrbanManagement SignalR**（JWT 续期推送，可选）
7. **5000 等产品回归** — 确认 `SendAuthLicense` / `DownloadAuth` 无行为变化

### 低优先级（可选优化）

8. **机器码采集脚本**（5001 离线前置）

---

## 接口契约总结

### UrbanManagement → BasePlatform.PublicApi（5001）

| 接口 | 方法 | 用途 |
|-----|------|------|
| `/api/AuthClientLicense/GetAuthClientLicense` | POST | 验码（现网；非 5001） |
| `/api/auth/activate-urban`（或扩展验码接口） | POST | **5001** 在线激活：验码 + 写 `MachineCode` + **`jwtToken`** |
| `/api/auth/license-file` | GET | **5001** JWT 文件（离线 / Urban 代理） |

### MaterialClient.Urban → UrbanManagement（5001）

| 接口 | 方法 | 用途 |
|-----|------|------|
| `/api/urban/auth/activate` | POST | 授权码激活（响应含 **`jwtToken`**） |
| ~~`/api/urban/auth/verify`~~ | — | **不实施** |
| SignalR DeviceStatusHub | Hub | JWT **续期**推送（可选） |

---

## 数据模型变更总结

### UrbanManagement.GovProject

| 字段名 | 类型 | 说明 |
|-------|------|------|
| AccessCode | `NVARCHAR(200)` | 城管接入码（**原 BuildLicenseNo 列重命名**） |
| MachineCode | `NVARCHAR(128)` | 绑定的机器码（新增） |
| AuthToken | `UNIQUEIDENTIFIER` | 授权令牌（新增） |
| LastMachineCodeUpdate | `DATETIME2` | 机器码更新时间（新增） |
| AuthEndTime | `DATETIME2` | 授权结束时间（已有，保留） |

**说明**：
- 授权状态、过期等以 BasePlatform `JC_ProductAuthority` 与 JWT `exp` 为准
- UrbanManagement `GovProject` 存项目级副本（Pull、政府出站等）
- 客户端日常验权：**本地 JWT**，不依赖 Urban verify API

### MaterialClient.LicenseInfo

**变更**：`BuildLicenseNo` → **`AccessCode`**；**不存储 `AuthToken`**；激活/导入须写入 **`LatestJwtToken`**（权威 JWT）。

---

## 风险与缓解措施

| 风险 | 影响 | 缓解措施 |
|-----|------|---------|
| 非 5001 误走 JWT 路径 | 破坏其它产品授权 | 全链路 `productCode == 5001` 白名单；5000 回归测试 |
| 在线激活无 JWT | 启动无法验签 | 激活响应必须含 `jwtToken` 并写 `LatestJwtToken` |
| 客户端不知道 ProId | 无法在激活时指定项目 | ProId 由验码后服务端返回 |
| AccessCode 未维护 | JWT `accessCode` 为空 | 02 P0 + 签发前校验 |
| 机器码不稳定 | 授权失效 | 运营重新录入 / 在线重新激活 |

---

## 后续步骤

1. **各项目基于此 EPIC 创建详细 Proposal**
   - [02 / 03 / 04 / 05 拟稿](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md)（AccessCode + JWT + Urban）
   - MaterialClient.Urban：激活写 `LatestJwtToken`
   - **5000 等非 5001 产品回归清单**

2. **技术评审**
   - JWT 唯一权威 + 仅 5001 产品隔离
   - 废弃 verify API
   - 在线激活 `jwtToken` 闭环

3. **分阶段实施**
   - 阶段一：02 P0（`AccessCode`）+ 03 JWT 签发与离线下载
   - 阶段二：5001 在线激活 + Urban 代理 + 客户端 `LatestJwtToken`
   - 阶段三（可选）：SignalR JWT 续期

---

**文档版本**：1.1  
**创建日期**：2026-06-23  
**最后更新**：2026-05-29（JWT 唯一权威、仅 5001、废弃 verify、在线 jwtToken）  
**状态**：已与 confusion 系列 03 拟稿对齐；待各仓库实施
