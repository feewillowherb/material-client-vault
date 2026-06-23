# Urban BasePlatform 授权方案调研

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
    public string? BuildLicenseNo { get; set; }
    public string? FdBuildLicenseNo { get; set; }

    // 新增机器码授权字段
    public string? MachineCode { get; set; }           // 绑定的机器码
    public string? AuthToken { get; set; }             // 授权令牌
    public DateTime? AuthBeginDate { get; set; }       // 授权开始时间
    public DateTime? AuthEndDate { get; set; }         // 授权结束时间
    public int? AuthStatus { get; set; }               // 授权状态
    public int? AuthType { get; set; }                 // 授权类型
    public DateTime? LastMachineCodeUpdate { get; set; } // 机器码更新时间
}
```

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

## 关键验证规则

**离线验证**：当前机器码 == SQLite 机器码

**在线验证**：当前机器码 == SQLite 机器码 == BasePlatform 机器码

**任一不匹配** → 授权失效 → 关闭程序

## 实施工期估算

- **BasePlatform 扩展**：2-3 周
- **Urban 客户端改造**：2-3 周
- **数据迁移与测试**：2 周
- **总计**：6-8 周

## 相关调研

- [Urban PFX 授权方案](../2026-05-27-urban-management-pfx-auth-solution/) - 另一种授权方案（证书）

---

**调研时间**：2026-06-23
**调研状态**：方案设计完成，待实施
