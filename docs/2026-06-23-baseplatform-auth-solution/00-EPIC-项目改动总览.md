# UrbanManagement 代理授权方案 - 项目改动 EPIC

## 概述

本文档作为 UrbanManagement 代理授权方案的总体 EPIC，描述所有涉及项目的改动内容。每个项目可基于此文档创建详细的 Proposal 和实施计划。

**方案架构**：`MaterialClient.Urban → UrbanManagement → BasePlatform.PublicApi`

**核心目标**：
- 在 GovProject 中扩展机器码授权字段
- UrbanManagement 作为代理层处理授权验证
- MaterialClient.Urban 保持现有 LicenseInfo 结构和 JWT 验证机制
- BasePlatform.PublicApi 提供授权验证能力

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

**注意**：当前 BasePlatform 的授权码验证 API **不支持回写 machineCode**。授权码验证只返回预设的授权信息，不保存机器码。机器码绑定需要在 UrbanManagement 侧通过 GovProject 扩展字段实现。

---

### 2. UrbanManagement

**项目路径**：`FdSoft.UrbanManagement` 或 `UrbanManagement`

**改动内容**：

#### 2.1 GovProject 实体扩展

```csharp
// UrbanManagement.GovProject 实体扩展
public class GovProject : Entity<Guid>
{
    public string ProName { get; set; } = default!;
    public string? BuildLicenseNo { get; set; }        // 建设许可证号（保留）

    // ===== 新增：机器码授权字段 =====
    public string? MachineCode { get; set; }           // 绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌（GUID）
    public DateTime? AuthBeginDate { get; set; }       // 授权开始时间
    public DateTime? AuthEndDate { get; set; }         // 授权结束时间
    public int? AuthStatus { get; set; }               // 授权状态 0=失效, 1=正常
    public int? AuthType { get; set; }                 // 授权类型 0=离线, 1=在线
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码最后更新时间
}
```

**数据库迁移**：
```sql
-- 添加新字段到 GovProject 表
ALTER TABLE GovProject ADD COLUMN MachineCode NVARCHAR(128) NULL;
ALTER TABLE GovProject ADD COLUMN AuthToken UNIQUEIDENTIFIER NULL;
ALTER TABLE GovProject ADD COLUMN AuthBeginDate DATETIME2 NULL;
ALTER TABLE GovProject ADD COLUMN AuthEndDate DATETIME2 NULL;
ALTER TABLE GovProject ADD COLUMN AuthStatus INT NULL;
ALTER TABLE GovProject ADD COLUMN AuthType INT NULL;
ALTER TABLE GovProject ADD COLUMN LastMachineCodeUpdate DATETIME2 NULL;

-- 添加索引
CREATE INDEX IDX_GovProject_MachineCode ON GovProject(MachineCode);
CREATE INDEX IDX_GovProject_AuthToken ON GovProject(AuthToken);
CREATE INDEX IDX_GovProject_AuthStatus ON GovProject(AuthStatus);
```

#### 2.2 新增授权代理 API

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
  "buildLicenseNo": "BUILD-LICENSE-NO",
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
1. 根据 BuildLicenseNo 查找 GovProject
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
    public string? BuildLicenseNo { get; set; }
    public string? FdBuildLicenseNo { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string JwtToken { get; set; }
}
```

---

### 3. MaterialClient.Urban

**项目路径**：`MaterialClient.Urban`

**改动内容**：

#### 3.1 保持现有 LicenseInfo 结构（无改动）

**现有结构**（保持不变）：
```csharp
[Table("LicenseInfo")]
public class LicenseInfo : Entity<Guid>
{
    public Guid ProjectId { get; set; }
    public Guid? AuthToken { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string? ProName { get; set; }
    public string? BuildLicenseNo { get; set; }
    public string? FdBuildLicenseNo { get; set; }
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
            response.Data.BuildLicenseNo ?? "",
            response.Data.FdBuildLicenseNo ?? "",
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

**现状**：BasePlatform.PublicApi 的授权码验证 API **不支持回写 machineCode**。

**解决方案**：
- UrbanManagement 在 GovProject 中自行管理 MachineCode
- 授权码验证时，BasePlatform 返回预设的授权信息
- UrbanManagement 收到响应后，在本地 GovProject 中记录 MachineCode

### 2. 客户端激活时不提供 ProId

**原因**：客户端在激活时只知道授权码，不知道 ProId。

**解决方案**：
- 激活请求只包含：`{ code, machineCode }`
- ProId 由服务器端根据授权码匹配后返回
- UrbanManagement 根据 BasePlatform 返回的 ProId 更新 GovProject

### 3. 授权验证使用 BuildLicenseNo 而非 ProId

**原因**：
- MaterialClient.Urban 当前使用 BuildLicenseNo 作为许可证密钥
- 需要保持向后兼容

**解决方案**：
- UrbanManagement 验证接口使用 `buildLicenseNo` 查找 GovProject
- 客户端调用时传递 `BuildLicenseNo` 而非 `ProId`

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
| MachineCode | `NVARCHAR(128)` | 绑定的机器码（新增） |
| AuthToken | `UNIQUEIDENTIFIER` | 授权令牌（新增） |
| AuthBeginDate | `DATETIME2` | 授权开始时间（新增） |
| AuthEndDate | `DATETIME2` | 授权结束时间（新增） |
| AuthStatus | `INT` | 授权状态（新增） |
| AuthType | `INT` | 授权类型（新增） |
| LastMachineCodeUpdate | `DATETIME2` | 机器码更新时间（新增） |

### MaterialClient.LicenseInfo

**无结构变更**，现有字段已满足需求。新增 LatestJwtToken 字段用于接收服务器推送。

---

## 风险与缓解措施

| 风险 | 影响 | 缓解措施 |
|-----|------|---------|
| BasePlatform 不回写 MachineCode | 无法在 BasePlatform 端统一管理机器码 | UrbanManagement 在 GovProject 中自行管理 |
| 客户端不知道 ProId | 无法在激活时指定项目 | ProId 由服务器端根据授权码匹配后返回 |
| BuildLicenseNo 唯一性 | 可能存在重复许可证号 | UrbanManagement 根据 BuildLicenseNo 唯一查找 GovProject |
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
**最后更新**：2026-06-23
**状态**：待各项目创建详细 Proposal
