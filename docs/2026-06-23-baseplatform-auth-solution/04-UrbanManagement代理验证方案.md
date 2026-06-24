# UrbanManagement 代理验证方案讨论

## 方案概述

### 核心思想

MaterialClient.Urban **不直接连接 BasePlatform**，而是通过 UrbanManagement 作为代理进行授权验证和机器码管理。

### 架构变化

**原方案**（直连 BasePlatform）：
```
MaterialClient.Urban ──直接连接──> BasePlatform.PublicApi
```

**新方案**（UrbanManagement 代理）：
```
MaterialClient.Urban ──连接──> UrbanManagement ──代理──> BasePlatform.PublicApi
```

> **说明**：UrbanManagement 通过 BasePlatform.PublicApi 与 BasePlatform 通信。BasePlatform.WebApi 用于 UI 相关操作。

## 方案优势

### 1. 简化 MaterialClient.Urban 依赖

| 对比项 | 直连方案 | 代理方案 |
|-------|---------|---------|
| 外部依赖 | BasePlatform.PublicApi | 仅 UrbanManagement |
| 网络配置 | 需配置 BasePlatform 地址 | 仅配置 UrbanManagement 地址 |
| 认证方式 | 需 BasePlatform 认证 | 使用 UrbanManagement 现有认证 |
| 部署复杂度 | 高 | 低 |

### 2. 统一授权管理

- UrbanManagement 已有 GovProject 实体
- 可以在 GovProject 中统一管理 MachineCode
- MaterialClient.Urban 作为 UrbanManagement 的客户端，复用现有认证

### 3. 减少网络暴露面

- BasePlatform 仅暴露给 UrbanManagement
- MaterialClient.Urban 只需访问内网的 UrbanManagement
- 提高系统安全性

## 数据模型设计

### 1. GovProject 表扩展

```csharp
// UrbanManagement.GovProject 实体扩展
public class GovProject : Entity<Guid>
{
    // 现有字段（保留）
    public string ProName { get; set; } = default!;
    public string? BuildLicenseNo { get; set; }        // 建设许可证号
    public string? FdBuildLicenseNo { get; set; }      // 对接码
    public DateTime? AuthEndTime { get; set; }         // 授权结束时间（已有）

    // ===== 新增：机器码授权字段 =====
    public string? MachineCode { get; set; }           // 当前绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码最后更新时间
}
```

**字段说明**：
- `MachineCode` - 客户端机器码，用于绑定特定设备
- `AuthToken` - BasePlatform 授权令牌（GUID），由 BasePlatform 返回
- `LastMachineCodeUpdate` - 机器码最后更新时间，用于追踪变更
- `AuthEndTime` - 现有字段，表示授权结束时间

**不需要的字段**：
- ~~`AuthStatus`~~ - 授权状态应由 BasePlatform 的 JCProductAuthority 表管理
- ~~`AuthBeginDate`~~ - 授权开始时间在 UrbanManagement 业务场景中不需要
- ~~`AuthType`~~ - 授权类型（离线/在线）应由 BasePlatform 管理

### 2. MaterialClient.Urban 授权结构（实际代码）

**当前 LicenseInfo 实体**（MaterialClient.Common.Entities.LicenseInfo）：

```csharp
/// <summary>
/// 授权许可信息实体
/// 存储软件授权信息，包括项目ID、授权令牌和有效期
/// </summary>
[Table("LicenseInfo")]
public class LicenseInfo : Entity<Guid>
{
    /// <summary>
    /// 项目ID（从基础平台获取）
    /// </summary>
    public Guid ProjectId { get; set; }

    /// <summary>
    /// 授权令牌（可选，从基础平台获取）
    /// </summary>
    public Guid? AuthToken { get; set; }

    /// <summary>
    /// 授权结束时间
    /// </summary>
    public DateTime AuthEndTime { get; set; }

    /// <summary>
    /// 项目名称
    /// </summary>
    public string? ProName { get; set; }

    /// <summary>
    /// 施工许可证号（接入码）
    /// </summary>
    public string? BuildLicenseNo { get; set; }

    /// <summary>
    /// 对接码
    /// </summary>
    public string? FdBuildLicenseNo { get; set; }

    /// <summary>
    /// 服务器最后一次提供的权威 JWT 原始文本。
    /// 在线更新时由服务器端推送，启动时优先使用此值验证授权。
    /// 若为 null，则回退到 .urban 文件。
    /// </summary>
    public string? LatestJwtToken { get; set; }

    /// <summary>
    /// 机器码（用于验证授权是否匹配当前机器）
    /// </summary>
    public string MachineCode { get; set; }

    /// <summary>
    /// 创建时间
    /// </summary>
    public DateTime CreatedAt { get; set; }

    /// <summary>
    /// 最后更新时间
    /// </summary>
    public DateTime UpdatedAt { get; set; }

    /// <summary>
    /// 检查授权是否已过期
    /// </summary>
    public bool IsExpired => DateTime.Now > AuthEndTime;

    /// <summary>
    /// 更新授权信息
    /// </summary>
    public void Update(Guid? authToken, DateTime authEndTime, string machineCode,
        string? proName = null, string? buildLicenseNo = null, string? fdBuildLicenseNo = null);
}
```

**授权验证机制**：
1. **JWT 验证**：使用 RSA 公钥验证 RS256 签名
2. **验证优先级**：LatestJwtToken → .urban 文件
3. **Claims 提取**：proId, proName, buildLicenseNo, fdBuildLicenseNo, exp
4. **过期检查**：AuthEndTime（从 JWT exp claim 获取）
5. **机器码验证**：当前机器码 == LicenseInfo.MachineCode

> **说明**：客户端使用 JWT 令牌验证授权。UrbanManagement 通过 SignalR DeviceStatusHub 推送最新的 JWT 到客户端的 `LatestJwtToken` 字段。
    ProId TEXT,                       -- 项目ID（可选）
    CreateDate TEXT NOT NULL
);
```

> **说明**：客户端 LicenseInfo 结构保持不变。新增字段（MachineCode、ExpireDate 等）为可选字段，确保向后兼容。UrbanManagement 负责授权验证和机器码管理。

## 流程设计

### 1. 离线授权流程

```
┌─────────────┐      ┌──────────────┐      ┌───────────────┐
│ BasePlatform│ ───> │UrbanManagement│ ───> │MaterialClient │
└─────────────┘      └──────────────┘      │    .Urban      │
                            │              └───────────────┘
                            ▼
     1. 生成授权文件
     （含 MachineCode）
                            │
                            ▼
     2. 写入 GovProject
     (MachineCode, AuthToken)
                            │
                            ▼
     3. 提供客户端接口
     ────────────────────────────────────────>
                            │
                            ▼
                   4. 调用 UrbanManagement API
                   获取授权信息（含 MachineCode）
                            │
                            ▼
                   5. 写入本地 SQLite
```

### 2. 在线授权与激活流程

```
┌─────────────┐      ┌──────────────┐      ┌───────────────┐
│ BasePlatform│      │UrbanManagement│      │MaterialClient │
│             │ <─── │    (代理)      │ <─── │    .Urban      │
└─────────────┘      └──────────────┘      └───────────────┘
      ▲                    ▲                      ▲
      │                    │                      │
      │                    │                      │
 1. 生成授权码         3. 转发请求          2. 输入授权码
    (管理员操作)          并上报机器码           并获取机器码
      │                    │                      │
      │                    ▼                      ▼
      │              4. 验证授权码             
      │              并持久化到               
  7. 返回授权    ───> JC_ProductAuthority <─── 5. 写入 GovProject
    信息给管理             │                    (MachineCode, AuthToken)
      员                   ▼
                      6. 更新 GovProject
                      (MachineCode, AuthToken)
                            │
                            ▼
                    8. 返回给客户端
                    <────────────────
                            │
                            ▼
                    9. 客户端写入 SQLite
```

#### 详细步骤说明

| 步骤 | 操作方 | 操作内容 | 数据流向 |
|-----|-------|---------|---------|
| 1 | BasePlatform 管理员 | 生成一次性授权码 | 写入 Redis |
| 2 | MaterialClient.Urban 用户 | 输入授权码 | - |
| 3 | MaterialClient.Urban | 获取本地机器码，发送请求到 UrbanManagement | → UrbanManagement |
| 4 | UrbanManagement | 转发到 BasePlatform 验证 | → BasePlatform |
| 5 | BasePlatform | 验证授权码，持久化机器码 | 写入 JC_ProductAuthority |
| 6 | UrbanManagement | 更新 GovProject | 写入 MachineCode, AuthToken |
| 7 | BasePlatform | 返回授权信息给管理员（确认） | - |
| 8 | UrbanManagement | 返回激活结果给客户端 | → MaterialClient.Urban |
| 9 | MaterialClient.Urban | 写入 ProId 到本地 SQLite | - |

#### 关键时序说明

```
时间线：

T0: 管理员在 BasePlatform 生成授权码 "AUTH-123456"
    → Redis: AuthClientLicense:UrbanManagement:AUTH-123456 = {...}

T1: 用户在 MaterialClient.Urban 输入授权码 "AUTH-123456"

T2: MaterialClient.Urban 获取机器码 "MACHINE-ABC-123"
    → POST /api/urban/auth/activate-proxy
    → Body: { code: "AUTH-123456", machineCode: "MACHINE-ABC-123" }
    → 注意：客户端此时不知道 ProId，ProId 在响应中返回

T3: UrbanManagement 收到请求，转发到 BasePlatform
    → POST /api/auth/activate
    → Body: { productCode: "UrbanManagement", code: "AUTH-123456", machineCode: "MACHINE-ABC-123" }

T4: BasePlatform 验证授权码
    → 从 Redis 读取并删除授权码
    → 写入 JC_ProductAuthority: { MachineCode: "MACHINE-ABC-123", AuthToken: "TOKEN-XYZ", ... }
    → 返回: { success: true, data: { authToken: "TOKEN-XYZ", authEndDate: "2026-12-31", ... } }

T5: UrbanManagement 收到响应，更新 GovProject
    → UPDATE GovProject SET MachineCode = "MACHINE-ABC-123", AuthToken = "TOKEN-XYZ", ...

T6: BasePlatform 向管理员确认激活成功（可选）

T7: UrbanManagement 返回结果给 MaterialClient.Urban
    → Response: { success: true, data: { authToken: "TOKEN-XYZ", ... } }

T8: MaterialClient.Urban 写入本地 SQLite（仅存储 ProId）
    → INSERT INTO UrbanAuth (ProId, CreateDate) VALUES (...)
```

### 3. 本地验证流程（极简）

```
MaterialClient.Urban 启动
        │
        ▼
1. 从 SQLite 读取 ProId
        │
        ▼
2. 获取当前机器码
        │
        ▼
3. 调用 UrbanManagement API 验证（ProId + MachineCode）
        │
        ▼
┌───────────────────────────────┐
│ UrbanManagement 验证逻辑：      │
│ 1. 根据 ProId 查找 GovProject  │
│ 2. 检查 GovProject.AuthEndTime │（是否过期）
│ 3. 比对：当前机器码 ==         │
│    GovProject.MachineCode      │
└───────────────────────────────┘
        │
        ▼
5. 返回验证结果
        │
        ▼
    验证成功 → 继续运行
    验证失败 → 关闭程序
```

## API 设计（UrbanManagement 侧）

### 1. 授权码激活代理接口

```csharp
// UrbanManagement.HttpApi.Host/Controllers/UrbanAuthProxyController.cs

[Route("api/urban/auth")]
public class UrbanAuthProxyController : AbpController
{
    /// <summary>
    /// 代理激活授权
    /// </summary>
    [HttpPost("activate")]
    public async Task<ApiResultDto<ActivationResultDto>> ActivateProxy(
        [FromBody] ActivateProxyRequest request)
    {
        // 1. 获取 BasePlatform 授权
        var basePlatformResponse = await _basePlatformAuthClient.ActivateAsync(
            new ActivateRequest
            {
                ProductCode = "UrbanManagement",
                Code = request.Code,
                MachineCode = request.MachineCode
            });

        if (!basePlatformResponse.Success)
        {
            return ApiResultDto<ActivationResultDto>.Fail(basePlatformResponse.Message);
        }

        // 2. 从 BasePlatform 响应中获取 ProId
        var proId = basePlatformResponse.Data.ProId;

        // 3. 更新本地 GovProject（根据 BasePlatform 返回的 ProId）
        var project = await _projectRepository.FirstOrDefaultAsync(
            p => p.ProId == proId);

        if (project != null)
        {
            project.MachineCode = request.MachineCode;
            project.AuthToken = basePlatformResponse.Data.AuthToken;
            project.AuthEndTime = basePlatformResponse.Data.AuthEndDate;
            project.LastMachineCodeUpdate = DateTime.UtcNow;

            await _projectRepository.UpdateAsync(project);
        }

        // 3. 返回结果
        return ApiResultDto<ActivationResultDto>.Success(new ActivationResultDto
        {
            AuthToken = basePlatformResponse.Data.AuthToken,
            AuthEndDate = basePlatformResponse.Data.AuthEndDate,
            ProId = project?.ProId,
            ProName = project?.ProName
        });
    }
}
```

### 2. 本地验证接口（基于 LicenseKey）

```csharp
/// <summary>
/// 验证客户端授权（UrbanManagement 内部验证）
/// </summary>
[HttpPost("verify")]
public async Task<ApiResultDto<VerifyResultDto>> VerifyLocal(
    [FromBody] VerifyLocalRequest request)
{
    // 1. 根据 LicenseKey（BuildLicenseNo）查找 GovProject
    var project = await _projectRepository.FirstOrDefaultAsync(
        p => p.BuildLicenseNo == request.LicenseKey);

    if (project == null)
    {
        return ApiResultDto<VerifyResultDto>.Fail("许可证不存在");
    }

    // 2. 检查授权是否过期
    if (project.AuthEndTime.HasValue && DateTime.UtcNow > project.AuthEndTime.Value)
    {
        return ApiResultDto<VerifyResultDto>.Fail("授权已过期");
    }

    // 3. 验证机器码（关键步骤）
    if (project.MachineCode != request.MachineCode)
    {
        _logger.LogWarning(
            "Machine code mismatch for project {ProId}. Expected: {Expected}, Actual: {Actual}",
            project.ProId, project.MachineCode, request.MachineCode);

        return ApiResultDto<VerifyResultDto>.Fail("机器码不匹配，授权无效");
    }

    // 5. 验证通过
    return ApiResultDto<VerifyResultDto>.Success(new VerifyResultDto
    {
        IsValid = true,
        ProId = project.ProId,
        ProName = project.ProName,
        AuthEndDate = project.AuthEndDate
    });
}
```

### 3. 授权文件获取代理接口

```csharp
/// <summary>
/// 获取离线授权文件（代理 BasePlatform）
/// </summary>
[HttpGet("license-file")]
public async Task<IActionResult> GetLicenseFileProxy([FromQuery] string machineCode)
{
    // 1. 调用 BasePlatform API 获取授权文件
    var response = await _basePlatformAuthClient.GetLicenseFileAsync(
        new GetLicenseFileRequest
        {
            ProductCode = "UrbanManagement",
            MachineCode = machineCode
        });

    if (!response.Success)
    {
        return BadRequest(response.Message);
    }

    // 2. 返回授权文件
    return File(
        Convert.FromBase64String(response.Data.LicenseFile),
        "application/octet-stream",
        response.Data.FileName);
}
```

## MaterialClient.Urban 验证实现（兼容现有 LicenseInfo）

### 1. 验证服务（基于 LicenseKey）

```csharp
public class UrbanAuthService
{
    private readonly IUrbanManagementApi _urbanApi;
    private readonly LicenseInfo _licenseInfo;  // 当前 LicenseInfo 结构

    /// <summary>
    /// 启动时验证（保持现有接口）
    /// </summary>
    public async Task<bool> VerifyOnStartup()
    {
        // 1. 使用当前 LicenseInfo 中的 LicenseKey（BuildLicenseNo）
        if (string.IsNullOrEmpty(_licenseInfo.LicenseKey))
        {
            MessageBox.Show("未配置许可证，请联系管理员", "授权验证",
                MessageBoxButton.OK, MessageBoxImage.Error);
            return false;
        }

        // 2. 获取当前机器码
        var machineCode = MachineCodeProvider.GetMachineCode();

        // 3. 调用 UrbanManagement 验证（LicenseKey + MachineCode）
        var response = await _urbanApi.VerifyLocal(new VerifyLocalRequest
        {
            LicenseKey = _licenseInfo.LicenseKey,  // BuildLicenseNo
            MachineCode = machineCode
        });

        // 4. 处理结果
        if (!response.Success)
        {
            MessageBox.Show($"授权验证失败：{response.Message}", "错误",
                MessageBoxButton.OK, MessageBoxImage.Error);
            Application.Current.Shutdown();
            return false;
        }

        // 5. 更新本地 LicenseInfo（如果返回了新字段）
        if (response.Data != null)
        {
            _licenseInfo.ProId = response.Data.ProId;
            _licenseInfo.ProName = response.Data.ProName;
            _licenseInfo.ExpireDate = response.Data.AuthEndDate;
        }

        return true;
    }

    /// <summary>
    /// 在线激活
    /// </summary>
    public async Task<bool> ActivateOnline(string authCode)
    {
        var machineCode = MachineCodeProvider.GetMachineCode();

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

        // 从响应中获取 ProId 并保存到本地
        var proId = response.Data.ProId;
        var existing = await _db.UrbanAuths.FirstOrDefaultAsync();
        if (existing != null)
        {
            existing.ProId = proId;
        }
        else
        {
            await _db.UrbanAuths.AddAsync(new UrbanAuth
            {
                ProId = proId,
                CreateDate = DateTime.UtcNow
            });
        }
        await _db.SaveChangesAsync();

        MessageBox.Show("激活成功！", "成功",
            MessageBoxButton.OK, MessageBoxImage.Information);
        return true;
    }
}
```

## 方案对比

### 对比表

| 维度 | 直连 BasePlatform | UrbanManagement 代理 |
|-----|------------------|---------------------|
| MaterialClient.Urban 复杂度 | 中等 | **低** |
| 外部依赖 | BasePlatform | **无（仅内网）** |
| 部署配置 | 需配置 BasePlatform 地址 | **仅需 UrbanManagement 地址** |
| 网络安全 | BasePlatform 需暴露 | **BasePlatform 仅暴露给 UrbanManagement** |
| 机器码存储位置 | BasePlatform + 客户端 | **GovProject（统一管理）** |
| 验证方式 | 三方比对 | **双方比对（客户端 <--> UrbanManagement）** |
| 扩展性 | 好 | **中（依赖 UrbanManagement）** |

## 风险与限制

### 1. UrbanManagement 单点故障

- UrbanManagement 不可用时，MaterialClient.Urban 无法验证
- **缓解**：本地缓存 + 离线宽限期

### 2. UrbanManagement 扩展性限制

- 多个项目需要通过 UrbanManagement 中转
- **缓解**：UrbanManagement 可作为网关，统一管理多个项目

### 3. 机器码同步延迟

- BasePlatform 更新机器码后，需先同步到 UrbanManagement
- **缓解**：UrbanManagement 定期同步 BasePlatform 数据

## 推荐结论

**推荐采用 UrbanManagement 代理方案**，原因如下：

1. ✅ **简化部署**：MaterialClient.Urban 只需配置 UrbanManagement 地址
2. ✅ **提高安全性**：BasePlatform 不直接暴露给客户端
3. ✅ **统一管理**：机器码在 GovProject 中统一管理
4. ✅ **降低复杂度**：MaterialClient.Urban 无需处理 BasePlatform 认证
5. ⚠️ **需注意**：确保 UrbanManagement 高可用

---

**创建时间**：2026-06-23
**方案状态**：讨论中，推荐采用
