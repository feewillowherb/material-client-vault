# UrbanManagement 授权方案设计

> **字段语义**：见 [AccessCode 分离方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md)。`GovProject.BuildLicenseNo` 重命名为 **`AccessCode`**。

## 1. 概述

本文档定义 UrbanManagement 基于当前授权机制的扩展方案，采用 UrbanManagement 代理架构：
- MaterialClient.Urban → UrbanManagement → BasePlatform.PublicApi
- UrbanManagement 在 GovProject 中统一管理 MachineCode
- 简化客户端实现，减少外部依赖

## 2. GovProject 表变更

### 2.1 当前字段

```csharp
// 当前 UrbanManagement.GovProject 实体
public class GovProject : Entity<Guid>
{
    public string ProName { get; set; } = default!;
    public string? AccessCode { get; set; }            // 城管接入码（原 BuildLicenseNo）
    public string? FdBuildLicenseNo { get; set; }      // 凡东对接码（MD5）
}
```

### 2.2 扩展后字段

```csharp
// 扩展后的 UrbanManagement.GovProject 实体
public class GovProject : Entity<Guid>
{
    // 现有字段（保留）
    public string ProName { get; set; } = default!;
    public string? AccessCode { get; set; }            // 城管接入码
    public string? FdBuildLicenseNo { get; set; }      // 凡东对接码
    public DateTime? AuthEndTime { get; set; }         // 授权结束时间（已有）
    public DateTime? AddTime { get; set; }             // 添加时间（已有）

    // ===== 新增：机器码授权字段 =====
    public string? MachineCode { get; set; }           // 当前绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌（GUID）
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码最后更新时间
}
```

### 2.3 字段变更总结

| 字段 | 变更 | 说明 |
|-----|------|------|
| AccessCode | **重命名**（原 BuildLicenseNo） | 城管接入码，PublicApi / 验证主键 |
| FdBuildLicenseNo | **保留** | 凡东 MD5 对接码 |
| AuthEndTime | **保留** | 授权结束时间（已有字段） |
| MachineCode | **新增** | 机器码绑定 |
| AuthToken | **新增** | BasePlatform 授权令牌 |
| LastMachineCodeUpdate | **新增** | 机器码更新时间 |

**不需要的字段**：
- ~~`AuthStatus`~~ - 授权状态应由 BasePlatform 的 JCProductAuthority 表管理
- ~~`AuthBeginDate`~~ - 授权开始时间在 UrbanManagement 业务场景中不需要
- ~~`AuthType`~~ - 授权类型（离线/在线）应由 BasePlatform 管理

## 3. BasePlatform.PublicApi 接口（UrbanManagement 调用）

UrbanManagement 通过 BasePlatform.PublicApi 调用以下接口。

### 3.1 获取授权文件 API（离线授权）

**接口**：`GET /api/auth/license-file`

**描述**：BasePlatform.WebApi 管理界面录入机器码后，生成并下载授权文件

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

```json
{
  "success": true,
  "data": {
    "licenseFile": "base64-encoded-license-file",
    "fileName": "urban-license-{machineCode}.json"
  }
}
```

**授权文件格式**（解密后）：

```json
{
  "version": "1.0",
  "productCode": "UrbanManagement",
  "machineCode": "MACHINE-CODE-12345",
  "proId": "project-guid",
  "proName": "项目名称",
  "authBeginDate": "2026-06-23T00:00:00",
  "authEndDate": "2026-12-31T23:59:59",
  "authType": 0,
  "authToken": "GUID-AUTH-TOKEN",
  "signature": "digital-signature"
}
```

### 3.2 机器码上报与激活 API（在线授权）

**接口**：`POST /api/auth/activate`

**描述**：UrbanManagement 使用授权码激活，同时上报机器码

**请求参数**：

```json
{
  "productCode": "UrbanManagement",
  "code": "ONE-TIME-AUTH-CODE",
  "machineCode": "MACHINE-CODE-12345"
}
```

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

**服务端处理逻辑**：

1. 验证 Redis 中的授权码
2. 删除授权码（一次性使用）
3. 插入/更新 `JCProductAuthority` 表（包含 MachineCode 字段）
4. 返回授权信息给 UrbanManagement
5. UrbanManagement 更新本地 GovProject

### 3.3 在线验证 API

**接口**：`POST /api/auth/verify`

**描述**：UrbanManagement 在线验证授权有效性

**请求参数**：

```json
{
  "productCode": "UrbanManagement",
  "authToken": "GUID-AUTH-TOKEN",
  "machineCode": "MACHINE-CODE-12345"
}
```

**响应**：

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

### 3.4 机器码获取脚本 API（可选）

**接口**：`GET /api/auth/machine-code-script`

**描述**：获取机器码获取脚本（跨平台）

**响应**：返回对应平台的脚本文件

## 4. BasePlatform.JCProductAuthority 表设计

### 4.1 表结构

```sql
-- BasePlatform.JC_ProductAuthority 表（实际使用）
CREATE TABLE JC_ProductAuthority (
    AuthId BIGINT PRIMARY KEY IDENTITY,
    ProductCode INT NOT NULL,              -- 产品标识（Urban=5001）
    CoId INT NOT NULL,                     -- 项目企业主键
    ProId NVARCHAR(50) NOT NULL,           -- 项目主键
    AuthStatus TINYINT NOT NULL,           -- 授权状态（0=未授权，1=已授权）
    AuthBeginTime DATETIME,                 -- 授权开始时间
    AuthEndTime DATETIME,                  -- 授权到期时间
    MachineCode NVARCHAR(100),              -- 机器码（仅设备）
    AccessCode NVARCHAR(200),               -- 接入码（新增）
    AuthToken NVARCHAR(100),                -- 授权码
    AuthTime DATETIME,                      -- 授权时间
    AuthUserId INT NOT NULL,                -- 授权人主键
    AuthUser NVARCHAR(50),                 -- 授权人
    CheckStatus TINYINT NOT NULL,           -- 审核状态（1=待审核，2=审核通过，3=审核不通过）
    CheckRemark NVARCHAR(500),              -- 审核备注
    CheckTime DATETIME,                     -- 审核时间
    CheckUserId INT NOT NULL,               -- 审核人主键
    CheckUser NVARCHAR(50),                 -- 审核人
    Remark NVARCHAR(500),                   -- 备注
    CreateTime DATETIME NOT NULL,           -- 创建时间
    CreateUserId INT NOT NULL,              -- 创建人主键
    CreateUser NVARCHAR(50),               -- 创建人
    UpdateTime DATETIME,                    -- 修改时间
    UpdateUserId INT NOT NULL,              -- 修改人主键
    UpdateUser NVARCHAR(50),               -- 修改人
    DeleteStatus TINYINT NOT NULL            -- 是否删除
);
```

### 4.2 使用约定

```sql
-- AuthStatus 字段约定
-- 0 = 未授权
-- 1 = 已授权

-- CheckStatus 字段约定
-- 1 = 待审核
-- 2 = 审核通过
-- 3 = 审核不通过

-- ProductCode 字段约定
-- 5001 = UrbanManagement 产品

-- 新增索引优化
CREATE INDEX IDX_JCProductAuthority_ProId ON JC_ProductAuthority(ProId);
CREATE INDEX IDX_JCProductAuthority_MachineCode ON JC_ProductAuthority(MachineCode);
CREATE INDEX IDX_JCProductAuthority_AuthToken ON JC_ProductAuthority(AuthToken);
CREATE INDEX IDX_JCProductAuthority_ProductCode ON JC_ProductAuthority(ProductCode);
CREATE INDEX IDX_JCProductAuthority_AccessCode ON JC_ProductAuthority(AccessCode);
```

## 5. Redis 授权码存储约定

```
Key: AuthClientLicense:{productCode}:{code}
Value: JSON 授权信息
TTL: 86400（24小时，可配置）
```

## 6. 安全考虑

### 6.1 授权文件加密

```csharp
// 使用 AES 加密授权文件
public class LicenseFileEncryption
{
    public static string Encrypt(LicenseData data, string password)
    {
        var json = JsonSerializer.Serialize(data);
        var encrypted = AesEncrypt(json, password);
        return Convert.ToBase64String(encrypted);
    }

    public static LicenseData Decrypt(string base64Content, string password)
    {
        var encrypted = Convert.FromBase64String(base64Content);
        var json = AesDecrypt(encrypted, password);
        return JsonSerializer.Deserialize<LicenseData>(json);
    }
}
```

### 6.2 数字签名

```csharp
// 使用 RSA 签名确保文件完整性
public class LicenseSignature
{
    public static string Sign(string content, RSA privateKey)
    {
        var data = Encoding.UTF8.GetBytes(content);
        var signature = privateKey.SignData(data, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
        return Convert.ToBase64String(signature);
    }

    public static bool Verify(string content, string signature, RSA publicKey)
    {
        var data = Encoding.UTF8.GetBytes(content);
        var sig = Convert.FromBase64String(signature);
        return publicKey.VerifyData(data, sig, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
    }
}
```

### 6.3 AuthToken 生成

```csharp
// 使用 GUID 生成唯一 Token
public class AuthTokenGenerator
{
    public static string Generate()
    {
        return Guid.NewGuid().ToString("N").ToUpper();
    }
}
```

## 7. 错误码定义

| 错误码 | 描述 | 处理建议 |
|-------|------|---------|
| AUTH_001 | 授权码不存在 | 提示用户检查授权码 |
| AUTH_002 | 授权码已过期 | 联系管理员重新生成 |
| AUTH_003 | 机器码不匹配 | 提示授权无效，重新激活 |
| AUTH_004 | 授权已过期 | 联系管理员续期 |
| AUTH_005 | 授权已被撤销 | 联系管理员 |
| AUTH_006 | 授权文件格式错误 | 重新下载授权文件 |
| AUTH_007 | 签名验证失败 | 授权文件被篡改 |

## 8. UrbanManagement 代理接口（MaterialClient.Urban 调用）

### 8.1 授权码激活代理接口

**接口**：`POST /api/urban/auth/activate`

**描述**：MaterialClient.Urban 通过 UrbanManagement 代理激活授权

**请求参数**：

```json
{
  "code": "ONE-TIME-AUTH-CODE",
  "machineCode": "MACHINE-CODE-12345"
}
```

> **说明**：客户端在激活时只知道授权码和机器码，不知道 ProId。ProId 是服务器端根据授权码匹配后返回的。

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

### 8.2 本地验证接口

**接口**：`POST /api/urban/auth/verify`

**描述**：MaterialClient.Urban 通过 UrbanManagement 验证授权

**请求参数**：

```json
{
  "authToken": "GUID-AUTH-TOKEN",
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

### 8.3 授权文件获取代理接口

**接口**：`GET /api/urban/auth/license-file?machineCode=xxx`

**描述**：MaterialClient.Urban 通过 UrbanManagement 获取授权文件

**响应**：返回加密的授权文件

---

**文档版本**：2.1  
**最后更新**：2026-06-24（AccessCode 语义对齐）
**变更说明**：采用 UrbanManagement 代理方案，简化 GovProject 字段变更
