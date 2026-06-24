# UrbanManagement 代理授权方案 - 项目改动 EPIC

> **字段语义（已定）**：与 [AccessCode 分离方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md) 一致。  
> - **`AccessCode`**：城管接入码（`GovProject` 原 `BuildLicenseNo` 重命名）  
> - **`MachineCode`**：设备机器码  
> - ~~**`FdBuildLicenseNo`**~~：**已废弃**（不再计算、出站或写入 JWT）  
> - 政府 HTTP 出站 `buildLicenseNo` 协议名可保留，**值 = AccessCode**

## 概述

本文档作为 UrbanManagement 代理授权方案的总体 EPIC，描述所有涉及项目的改动内容。每个项目可基于此文档创建详细的 Proposal 和实施计划。

**方案架构**：`MaterialClient.Urban → UrbanManagement → BasePlatform.PublicApi`

**核心目标**：
- 在 GovProject 中扩展机器码授权字段
- UrbanManagement 作为代理层处理授权验证
- MaterialClient.Urban 更新 LicenseInfo（`BuildLicenseNo` → `AccessCode`）与 JWT 验证机制
- BasePlatform.PublicApi 提供授权验证能力
- **将 UrbanManagement 中的 JWT 授权文件签发流程移植到 BasePlatform**
- **在 BasePlatform 中提供授权文件下载功能**

---

## 涉及项目及改动清单

### 1. BasePlatform.PublicApi

**项目路径**：`FdSoft.BasePlatform.PublicApi`

**改动内容**：

#### 1.1 授权码验证 API（已有，无需改动）
- **接口**：`POST /api/AuthClientLicense/GetAuthClientLicense`
- **当前能力**：验证授权码，返回授权信息
- **数据模型**：
  ```csharp
  public class LicenseRequestDto
  {
      public string? ProductCode { get; set; }  // 产品代码
      public string? Code { get; set; }          // 授权码
  }
  ```

#### 1.2 机器码验证 API（可选新增）
- **接口**：`POST /api/auth/verify`
- **描述**：验证机器码是否与授权匹配
- **请求参数**：
  ```json
  {
    "productCode": "UrbanManagement",
    "authToken": "GUID-AUTH-TOKEN",
    "machineCode": "MACHINE-CODE-12345"
  }
  ```
- **响应**：
  ```json
  {
    "success": true,
    "data": {
      "isValid": true,
      "authStatus": 1,
      "authEndDate": "2026-12-31T23:59:59",
      "machineCodeMatch": true
    }
  }
  ```

#### 1.3 JWT 授权文件生成与下载 API（新增，从 UrbanManagement 移植）

**背景**：将 UrbanManagement 中的 JWT 签发流程移植到 BasePlatform，统一授权文件签发。

**接口**：`GET /api/auth/license-file`

**描述**：生成并下载 JWT 授权文件（.urban 文件）

**请求参数**：
```json
{
  "productCode": "UrbanManagement",
  "machineCode": "MACHINE-CODE-12345",
  "proId": "project-guid",
  "authEndDate": "2026-12-31T23:59:59"
}
```

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
    public string ProName { get; set; set; }
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

> **说明**：此功能将 UrbanManagement 的 `UrbanLicenseGenerator` 服务移植到 BasePlatform，使 BasePlatform 成为统一的授权文件签发中心。

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

#### 2.4 新增授权码激活代理 API

**接口**：`POST /api/urban/auth/activate`

**描述**：MaterialClient.Urban 通过此接口激活授权

**请求参数**：
```json
{
  "code": "ONE-TIME-AUTH-CODE",
  "machineCode": "MACHINE-CODE-12345"
}
```

> **注意**：客户端在激活时不知道 ProId，ProId 是服务器端根据授权码匹配后返回的。

**响应**：
```json
{
  "success": true,
  "data": {
    "authToken": "GUID-AUTH-TOKEN",
    "authEndDate": "2026-12-31T23:59:59",
    "proId": "project-guid",
    "proName": "项目名称"
  }
}
```

**实现逻辑**：
1. 接收授权码和机器码
2. 调用 BasePlatform.PublicApi `/api/AuthClientLicense/GetAuthClientLicense`
3. 从响应中获取 ProId
4. 更新本地 GovProject（写入 MachineCode、AuthToken 等）
5. 返回激活结果给客户端

#### 2.3 新增本地验证 API

**接口**：`POST /api/urban/auth/verify`

**描述**：MaterialClient.Urban 通过此接口验证授权

**请求参数**：
```json
{
  "accessCode": "ACCESS-CODE-VALUE",
  "machineCode": "MACHINE-CODE-12345"
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "isValid": true,
    "proId": "project-guid",
    "proName": "项目名称",
    "authEndDate": "2026-12-31T23:59:59"
  }
}
```

**实现逻辑**：
1. 根据 **AccessCode** 查找 GovProject
2. 检查授权状态和过期时间
3. 验证机器码是否匹配
4. 返回验证结果

#### 2.4 SignalR DeviceStatusHub 扩展（推送最新 JWT）

**Hub 方法**：`UpdateClientLicense`

**描述**：向客户端推送最新的授权 JWT

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

**变更**：`BuildLicenseNo` 属性重命名为 **`AccessCode`**；其余字段不变。
```csharp
[Table("LicenseInfo")]
public class LicenseInfo : Entity<Guid>
{
    public Guid ProjectId { get; set; }
    public Guid? AuthToken { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string? ProName { get; set; }
    public string? AccessCode { get; set; }
    public string? LatestJwtToken { get; set; }  // 服务器推送的最新 JWT
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

        // 从响应中获取 ProId 并更新本地 LicenseInfo
        await _licenseService.SyncProjectFieldsFromServerAsync(
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

#### 3.3 启动时验证流程（使用现有 JWT 验证）

**现有验证流程**（保持不变）：
- 使用 `StaticLicenseChecker` 验证 JWT 签名
- 优先使用 `LatestJwtToken`（服务器通过 SignalR 推送）
- 若为 null，则回退到 .urban 文件
- 验证过期时间 `AuthEndTime` 和机器码匹配

**无需改动**，现有验证机制已经满足需求。

---

### 4. BasePlatform.WebApi（可选）

**项目路径**：`FdSoft.BasePlatform.WebApi` 或 BasePlatform 管理后台

**改动内容**：

#### 4.1 授权码生成界面（新增或扩展现有功能）

**功能**：
- 管理员输入项目信息和机器码
- 生成一次性授权码
- 写入 Redis

**实现要点**：
- 授权码格式：`AuthClientLicense:{productCode}:{code}`
- 设置 TTL（如 24 小时）
- 支持重新生成

---

## 关键技术决策

### 1. BasePlatform 不回写 MachineCode

**核心决策**：BasePlatform.PublicApi 的授权码验证 API **不支持回写 MachineCode**，由 UrbanManagement 在 GovProject 中自行管理。

---

#### 1.1 决策背景

**技术约束**：
- BasePlatform 现有的 JC_ProductAuthority 表设计为授权中心的全局视图
- 该表由 BasePlatform.WebApi 通过管理界面操作，不暴露给外部 API 写入
- 若开放写权限，需重新设计 BasePlatform 的安全模型和 API 权限体系

**架构选择**：
- UrbanManagement 作为代理层，已有 GovProject 表存储项目级数据
- GovProject 是 Urban 领域的核心实体，天然适合管理 MachineCode
- 保持 BasePlatform 为只读验证服务，简化跨系统交互

---

#### 1.2 架构原理

**责任分离**：
```
┌─────────────────────────────────────────────────────────────────┐
│                      BasePlatform 平台                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  JC_ProductAuthority（授权中心全局视图）                    │ │
│  │  - 授权状态全局汇总                                         │ │
│  │  - AuthToken 管理                                          │ │
│  │  - 授权过期时间管理                                         │ │
│  │  - 由 BasePlatform.WebApi 管理（不暴露写 API）              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                ↑
                                │ 只读验证
                                │
┌─────────────────────────────────────────────────────────────────┐
│                    UrbanManagement 代理层                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  GovProject（项目级机器码权威来源）                          │ │
│  │  - MachineCode（唯一权威存储）                               │ │
│  │  - AuthToken（来自 BasePlatform 的副本）                     │ │
│  │  - LastMachineCodeUpdate（更新追踪）                         │ │
│  │  - 由 UrbanManagement 代理 API 写入                          │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**数据流向**：
1. **激活时**：UrbanManagement 收到客户端激活请求 → 调用 BasePlatform 验证 → BasePlatform 返回授权信息 → UrbanManagement 在本地 GovProject 写入 MachineCode
2. **验证时**：UrbanManagement 使用本地 GovProject.MachineCode 进行比对 → 不需要调用 BasePlatform

---

#### 1.3 权衡分析

**优点**：

| 优点 | 说明 |
|-----|------|
| **单一数据源** | GovProject.MachineCode 是唯一权威来源，避免数据同步冲突 |
| **简化 API** | BasePlatform API 保持只读，无需开放写权限 |
| **降低耦合** | UrbanManagement 可以独立管理项目数据，不依赖 BasePlatform 的写入接口 |
| **性能优化** | 验证时无需跨系统调用，直接比对本地数据 |
| **安全隔离** | BasePlatform 不暴露写 API，减少攻击面 |

**缺点与缓解**：

| 缺点 | 缓解措施 |
|-----|---------|
| **BasePlatform 无全局 MachineCode 视图** | UrbanManagement 可定期通过管理界面同步数据到 BasePlatform |
| **数据一致性风险** | GovProject 是权威来源，BasePlatform.JC_ProductAuthority 作为副本即可 |
| **运维复杂度增加** | 需明确 GovProject 为主要数据源，建立运维规范 |

---

#### 1.4 替代方案对比

**方案 A（当前）**：UrbanManagement 管理 MachineCode
- ✅ 单一数据源，无同步冲突
- ✅ API 简化，BasePlatform 只读
- ❌ BasePlatform 缺乏全局视图

**方案 B**：BasePlatform 回写 MachineCode
- ✅ BasePlatform 有全局机器码视图
- ✅ 授权中心数据完整
- ❌ 需开放 BasePlatform 写 API（安全风险）
- ❌ 两个系统都可写入同一数据（同步冲突）
- ❌ UrbanManagement 需等待 BasePlatform 写入完成（延迟增加）

**方案 C**：双向同步
- ✅ 两边都有完整数据
- ❌ 需要复杂的同步机制
- ❌ 数据冲突解决困难
- ❌ 运维成本高

---

#### 1.5 实施建议

**当前方案实施要点**：
1. **明确数据所有权**：在文档中明确 GovProject.MachineCode 为唯一权威来源
2. **API 设计**：BasePlatform.PublicApi 保持只读验证接口
3. **运维规范**：如需全局视图，通过 UrbanManagement → BasePlatform.WebApi 的管理界面同步

**未来扩展路径**：
- 如需 BasePlatform 拥有全局 MachineCode 视图，可通过 UrbanManagement 提供的查询接口定期同步
- 或由 UrbanManagement 定期通过管理界面更新 BasePlatform.JC_ProductAuthority 表

### 2. 客户端激活时不提供 ProId

**原因**：客户端在激活时只知道授权码，不知道 ProId。

**解决方案**：
- 激活请求只包含：`{ code, machineCode }`
- ProId 由服务器端根据授权码匹配后返回
- UrbanManagement 根据 BasePlatform 返回的 ProId 更新 GovProject

### 3. 授权验证使用 AccessCode 而非 ProId

**原因**：
- MaterialClient.Urban 以 **AccessCode**（原误称 BuildLicenseNo）作为项目接入标识
- 需要保持向后兼容（迁移期 JWT 可同时携带 `accessCode` 与废弃 claim）

**解决方案**：
- UrbanManagement 验证接口使用 **`accessCode`** 查找 GovProject
- 客户端调用时传递 **AccessCode** 而非 `ProId`

### 4. JWT 推送机制

**现状**：MaterialClient 已有完整的 JWT 验证机制（StaticLicenseChecker）。

**增强**：
- UrbanManagement 通过 SignalR DeviceStatusHub 推送最新 JWT
- 客户端更新 LicenseInfo.LatestJwtToken
- 启动时优先使用 LatestJwtToken，回退到 .urban 文件

---

## 实施优先级

### 高优先级（核心功能）

1. **UrbanManagement GovProject 扩展**
   - 数据库迁移脚本
   - 实体字段添加
   - 索引创建

2. **UrbanManagement 授权代理 API**
   - `/api/urban/auth/activate` - 授权激活
   - `/api/urban/auth/verify` - 本地验证

3. **MaterialClient.Urban 激活流程**
   - 授权码激活 UI
   - 调用 UrbanManagement API
   - 更新本地 LicenseInfo

### 中优先级（增强功能）

4. **UrbanManagement SignalR 推送**
   - DeviceStatusHub 推送最新 JWT
   - 客户端接收并更新 LatestJwtToken

5. **BasePlatform.WebApi 管理界面**
   - 授权码生成功能
   - Redis 管理

### 低优先级（可选优化）

6. **机器码获取脚本**
   - 跨平台机器码生成工具
   - 客户端部署工具

---

## 接口契约总结

### UrbanManagement → BasePlatform.PublicApi

| 接口 | 方法 | 用途 |
|-----|------|------|
| `/api/AuthClientLicense/GetAuthClientLicense` | POST | 验证授权码（已有） |

### MaterialClient.Urban → UrbanManagement

| 接口 | 方法 | 用途 |
|-----|------|------|
| `/api/urban/auth/activate` | POST | 授权码激活（新增） |
| `/api/urban/auth/verify` | POST | 本地验证（新增） |
| SignalR DeviceStatusHub | Hub | 推送最新 JWT（已有，需扩展） |

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
- 授权状态、授权类型等管理字段由 BasePlatform 的 JC_ProductAuthority 表负责
- UrbanManagement.GovProject 仅需存储机器码绑定信息和授权令牌
- 授权验证逻辑由 BasePlatform.PublicApi 处理，UrbanManagement 作为代理层转发

### MaterialClient.LicenseInfo

**变更**：属性 **`BuildLicenseNo` 重命名为 `AccessCode`**；`LatestJwtToken` 等其余字段不变。

---

## 风险与缓解措施

| 风险 | 影响 | 缓解措施 |
|-----|------|---------|
| BasePlatform 不回写 MachineCode | 无法在 BasePlatform 端统一管理机器码 | UrbanManagement 在 GovProject 中自行管理 |
| 客户端不知道 ProId | 无法在激活时指定项目 | ProId 由服务器端根据授权码匹配后返回 |
| AccessCode 唯一性 | 接入码应在 GovProject 内可唯一查找 | 为 AccessCode 建索引；Pull 同步与 BasePlatform 对齐 |
| 机器码不稳定 | 硬件变更导致授权失效 | 提供机器码重新绑定流程 |

---

## 后续步骤

1. **各项目基于此 EPIC 创建详细 Proposal**
   - BasePlatform.PublicApi：验证现有 API 能力
   - UrbanManagement：详细实施计划
   - MaterialClient.Urban：激活流程实现
   - BasePlatform.WebApi：管理界面设计

2. **技术评审**
   - 评审 UrbanManagement 代理架构设计
   - 评审 GovProject 字段扩展方案
   - 评审授权流程安全性

3. **分阶段实施**
   - 阶段一：UrbanManagement GovProject 扩展 + 代理 API
   - 阶段二：MaterialClient.Urban 激活流程
   - 阶段三：SignalR 推送机制增强
   - 阶段四：BasePlatform.WebApi 管理界面

---

**文档版本**：1.0
**创建日期**：2026-06-23
**最后更新**：2026-06-24（AccessCode 语义对齐）
**状态**：待各项目创建详细 Proposal
