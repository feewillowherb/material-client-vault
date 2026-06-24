# Urban BasePlatform 授权方案调研

> **字段语义**：与 [AccessCode 分离方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md) 一致。域内使用 **`AccessCode`**；~~`FdBuildLicenseNo`~~ **已废弃**。

## 调研文档索引

### 00-调研总览.md
- 调研背景与目标
- 当前 UrbanManagement 授权机制分析
- BasePlatform 现有能力分析
- 整体方案设计
- 实施建议与风险

### 01-API设计.md
- BasePlatform 新增 API 接口定义
- 授权文件格式与加密
- 数据库表设计
- 安全考虑与错误码

### 02-客户端验证设计.md
- SQLite 授权数据模型
- 机器码获取算法（跨平台）
- 离线验证机制
- 在线验证机制
- 授权失效处理

### 03-快速参考指南.md
- 核心概念速查
- 流程图
- API 速查
- 验证规则总结
- 实施检查清单

### 04-UrbanManagement代理验证方案.md
- **推荐方案**：UrbanManagement 作为代理
- MaterialClient.Urban 不直连 BasePlatform
- GovProject 扩展存储 MachineCode
- 简化客户端部署与配置

## 核心结论

### 推荐方案：UrbanManagement 代理验证

**采用 UrbanManagement 代理方案**（详见 04-UrbanManagement代理验证方案.md）

#### 核心优势
- MaterialClient.Urban 不直接连接 BasePlatform
- GovProject 统一管理 MachineCode
- 简化客户端部署和配置
- 提高安全性（BasePlatform 仅暴露给 UrbanManagement）

#### 架构对比

| 方案 | 架构 | 客户端复杂度 | 安全性 |
|-----|------|-------------|--------|
| 直连 | MaterialClient → BasePlatform.PublicApi | 中 | 中 |
| **代理（推荐）** | **MaterialClient → UrbanManagement → BasePlatform.PublicApi** | **低** | **高** |

> **说明**：UrbanManagement 通过 BasePlatform.PublicApi 通信。BasePlatform.WebApi 用于 UI 相关操作。

#### GovProject 扩展字段

```csharp
public class GovProject : Entity<Guid>
{
    // 现有字段...
    public string? AccessCode { get; set; }            // 城管接入码（原 BuildLicenseNo，待重命名）
    public DateTime? AuthEndTime { get; set; }        // 授权结束时间（已有）

    // 新增机器码授权字段
    public string? MachineCode { get; set; }           // 绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码更新时间
}
```

**字段说明**：
- `MachineCode` - 客户端机器码，用于绑定特定设备
- `AuthToken` - BasePlatform 授权令牌（GUID），由 BasePlatform 返回
- `LastMachineCodeUpdate` - 机器码最后更新时间，用于追踪变更
- `AuthEndTime` - 现有字段，表示授权结束时间

**不需要的字段**（由 BasePlatform 的 JCProductAuthority 表管理）：
- ~~`AuthStatus`~~ - 授权状态
- ~~`AuthBeginDate`~~ - 授权开始时间  
- ~~`AuthType`~~ - 授权类型（离线/在线）

**客户端 LicenseInfo 结构**（`AccessCode` 替代原 `BuildLicenseNo` 属性名）：
- 客户端使用 **`AccessCode`**（或过渡期 `LicenseKey`，值同为接入码）进行项目匹配验证
- 新增字段为可选，确保向后兼容

### 原直连方案摘要

BasePlatform 已具备基础授权能力，但需扩展以下功能以支持 Urban 客户端的双授权模式：

1. **机器码管理**：提供稳定的机器码生成算法
2. **授权文件**：支持离线授权的文件生成与下载
3. **客户端验证**：实现离线/在线双验证机制

## 授权模式

| 模式 | 激活方式 | 验证方式 | 适用场景 |
|-----|---------|---------|---------|
| 离线授权 | 导入授权文件 | 本地机器码比对 | 网络隔离环境 |
| 在线授权 | 输入授权码 | 定期服务器验证 | 有网络环境 |

## 关键验证规则（UrbanManagement 代理模式）

**启动时验证**：客户端使用 LicenseInfo.`AccessCode` + 当前机器码调用 UrbanManagement API
- UrbanManagement 根据 **`AccessCode`** 查找 GovProject
- UrbanManagement 验证：当前机器码 == GovProject.MachineCode

**任一不匹配** → 授权失效 → 关闭程序

> **说明**：客户端 JWT / LicenseInfo 以 **`AccessCode`** 作为项目接入标识。UrbanManagement 负责授权验证和机器码管理。详见 [01-解决方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md)。

## 实施工期估算

- **BasePlatform 扩展**：2-3 周
- **Urban 客户端改造**：2-3 周
- **数据迁移与测试**：2 周
- **总计**：6-8 周

## 相关调研

- [AccessCode 与 MachineCode 分离方案](../2026-06-24-buildlicenseno-machinecode-confusion/01-解决方案.md) - 字段语义与 UrbanManagement 对齐（**必读**）
- [Urban PFX 授权方案](../2026-05-27-urban-management-pfx-auth-solution/) - 另一种授权方案（证书）

---

**调研时间**：2026-06-23  
**语义对齐**：2026-06-24（AccessCode）  
**调研状态**：方案设计完成，待实施
