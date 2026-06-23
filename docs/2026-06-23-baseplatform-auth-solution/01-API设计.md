# BasePlatform 授权 API 设计

## 1. 概述

本文档定义 Urban 客户端基于 BasePlatform 的双授权模式所需的 API 接口。

## 2. 新增 API 接口

### 2.1 获取授权文件 API（离线授权）

**接口**：`GET /api/auth/license-file`

**描述**：BasePlatform 管理员录入机器码后，生成并下载授权文件

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

### 2.2 机器码上报与激活 API（在线授权）

**接口**：`POST /api/auth/activate`

**描述**：客户端使用授权码激活，同时上报机器码

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
3. 插入/更新 `Material_MachineCode` 表
4. 返回授权信息

### 2.3 在线验证 API

**接口**：`POST /api/auth/verify`

**描述**：客户端在线验证授权有效性

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

### 2.4 机器码获取脚本 API（可选）

**接口**：`GET /api/auth/machine-code-script`

**描述**：获取机器码获取脚本（跨平台）

**响应**：返回对应平台的脚本文件

## 3. 数据库表设计

### 3.1 Material_MachineCode 表扩展

现有表已包含所需字段，需明确使用约定：

```sql
-- AuthType 字段约定
-- 0 = 离线授权（文件导入）
-- 1 = 在线授权（授权码激活）

-- 新增索引优化
CREATE INDEX IDX_MachineCode_ProId ON Material_MachineCode(ProId);
CREATE INDEX IDX_MachineCode_Code ON Material_MachineCode(MachineCode);
CREATE INDEX IDX_MachineCode_Token ON Material_MachineCode(AuthToken);
```

### 3.2 Redis 授权码存储约定

```
Key: AuthClientLicense:{productCode}:{code}
Value: JSON 授权信息
TTL: 86400（24小时，可配置）
```

## 4. 安全考虑

### 4.1 授权文件加密

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

### 4.2 数字签名

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

### 4.3 AuthToken 生成

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

## 5. 错误码定义

| 错误码 | 描述 | 处理建议 |
|-------|------|---------|
| AUTH_001 | 授权码不存在 | 提示用户检查授权码 |
| AUTH_002 | 授权码已过期 | 联系管理员重新生成 |
| AUTH_003 | 机器码不匹配 | 提示授权无效，重新激活 |
| AUTH_004 | 授权已过期 | 联系管理员续期 |
| AUTH_005 | 授权已被撤销 | 联系管理员 |
| AUTH_006 | 授权文件格式错误 | 重新下载授权文件 |
| AUTH_007 | 签名验证失败 | 授权文件被篡改 |

---

**文档版本**：1.0
**最后更新**：2026-06-23
