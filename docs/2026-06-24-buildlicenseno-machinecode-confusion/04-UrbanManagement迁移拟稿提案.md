# UrbanManagement 迁移拟稿提案

> **文档类型**：拟稿提案（Draft Proposal）  
> **创建日期**：2026-06-24  
> **状态**：拟稿  
> **范围**：**仅** `UrbanManagement`（ABP 城管服务端）  
> **不在范围**：BasePlatform 库表/UI（→ [02](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md)）、BasePlatform JWT 签发实现（→ [03](./03-BasePlatform-JWT签发迁移拟稿提案.md)）、MaterialClient.Urban

**前置**：[01-解决方案.md](./01-解决方案.md) · [05-联合发版说明.md](./05-联合发版说明.md)

---

## 1. 提案摘要

UrbanManagement 侧两类迁移，**同一发版窗口、可分 PR**：

| 部分 | 主题 | 要点 |
|------|------|------|
| **§A AccessCode** | 数据语义 | `GovProject.BuildLicenseNo` → **`AccessCode`**；Pull 消费 PublicApi 新字段 |
| **§B JWT** | 签发归属 | **移除**本地 JWT 签发；**代理** BasePlatform `/api/auth/license-file`；Hub 推送更新 |

---

## 2. 目标与非目标

### 2.1 目标

**§A AccessCode**

1. EF 实体 `GovProject`：`BuildLicenseNo` 重命名为 `AccessCode`（或新列迁移后删旧列）。
2. `GovProjectPullBackgroundWorker`：映射 `dto.AccessCode`、`dto.MachineCode`。
3. 脏数据修复：以 BasePlatform 拉取结果为准刷新本地 `AccessCode`。
4. 查询与政府出站：`GovProject.AccessCode`（**不**以 verify API 做客户端门禁）。
5. 政府出站：`payload.buildLicenseNo = govProject.AccessCode`（协议名保留）。

**§B JWT**

1. 删除或废弃：`UrbanLicenseGenerator`、`GovProjectLicenseAppService` 中本地签发逻辑。
2. 新增/改造：`GET /api/urban/auth/license-file` → 调用 BasePlatform PublicApi（[03](./03-BasePlatform-JWT签发迁移拟稿提案.md)）。
3. SignalR `DeviceStatusHub`：**可选** JWT 续期推送（非激活前置）。
4. 移除 Urban `appsettings` 中 **JWT 私钥**；**`POST /api/urban/auth/activate`** 代理 BasePlatform（**5001**，响应 `jwtToken`）。

### 2.2 非目标

- BasePlatform 表结构、授权后台 UI。  
- 客户端 `LicenseInfo` 属性重命名（可后续单独立项）。  
- 政府平台协议字段改名（仍叫 `buildLicenseNo`）。

---

## 3. §A AccessCode 数据迁移

### 3.1 实体

```csharp
public class GovProject : Entity<Guid>
{
    public string ProName { get; set; } = default!;

    /// <summary>城管接入码（原 BuildLicenseNo）</summary>
    public string AccessCode { get; set; } = string.Empty;
    public string? MachineCode { get; set; }
    public string? AuthToken { get; set; }
    public DateTime? AuthEndTime { get; set; }
    // ...
}
```

### 3.2 EF / SQLite 迁移

```sql
-- SQLite 示例
ALTER TABLE Gov_Project RENAME COLUMN BuildLicenseNo TO AccessCode;
-- 或：ADD AccessCode + UPDATE AccessCode = BuildLicenseNo + 弃用旧列
```

### 3.3 Pull Worker

```csharp
AccessCode = x.AccessCode,
MachineCode = x.MachineCode,
```

**依赖**：PublicApi 已输出 `accessCode`、`machineCode`（02 P1）；Pull 直接映射新字段，**不**读 `buildLicenseNo`。

### 3.4 脏数据修复

```sql
-- 全量刷新示意：以同步任务结果覆盖
UPDATE Gov_Project SET AccessCode = @SyncedAccessCode, ... WHERE Id = @ProId;
```

### 3.5 ~~验证 API~~（不实施）

~~`POST /api/urban/auth/verify`~~ — **废弃**。客户端以本地 JWT 验签为准（见 [EPIC](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)）。

### 3.6 在线激活代理（仅 5001）

**`POST /api/urban/auth/activate`** — 转发 BasePlatform `activate-urban`；响应须含 **`jwtToken`** 透传客户端。详见 [EPIC §2.4](../2026-06-23-baseplatform-auth-solution/00-EPIC-项目改动总览.md)。

---

## 4. §B JWT 下线与代理

### 4.1 移除清单（拟稿）

| 组件 | 动作 |
|------|------|
| `UrbanLicenseGenerator` | 删除或 `[Obsolete]` 后删 |
| `GovProjectLicenseAppService` 签发方法 | 改为调用 BasePlatform 客户端 |
| `Jwt:PrivateKey`（Urban 配置） | 删除 |
| 管理端「生成授权文件」直写本地 | 改调 BasePlatform 或 Urban 代理 API |

### 4.2 代理 API

**`GET /api/urban/auth/license-file`**

```csharp
public async Task<IActionResult> GetLicenseFileProxy(string machineCode, ...)
{
    var response = await _basePlatformClient.GetLicenseFileAsync(new {
        ProductCode = "5001",
        MachineCode = machineCode,
        ProId = govProject.Id,
        AuthEndDate = govProject.AuthEndTime
    });
    return File(Encoding.UTF8.GetBytes(response.JwtToken), "application/octet-stream", "license.urban");
}
```

### 4.3 SignalR 推送

`ClientLicenseUpdateDto`：

```csharp
public class ClientLicenseUpdateDto
{
    public string ProId { get; set; }
    public string? ProName { get; set; }
    public string? AccessCode { get; set; }
    public DateTime AuthEndTime { get; set; }
    public string JwtToken { get; set; }         // 来自 BasePlatform 签发
}
```

Hub 方法 `UpdateClientLicense`：JWT 文本须由 BasePlatform 签发，Urban **不**本地 `JwtSecurityTokenHandler` 签名。

### 4.4 Feature Flag（建议）

```json
"UrbanAuth": {
  "UseBasePlatformJwtIssuer": true
}
```

`false` 时回退旧 Urban 签发（仅灰度期）；P4 验证完成后移除旧代码。

---

## 5. 与 BasePlatform 提案的依赖

| 依赖项 | 来源文档 | Urban 动作前须满足 |
|--------|----------|-------------------|
| `AccessCode` 列与 PublicApi 字段 | 02 P1 | Pull 改映射 |
| JWT 签发 API | 03 P2 | 代理与 Hub 改调 BasePlatform |
| `JC_ProductAuthority.AccessCode` 有值 | 02 P0 | 签发 Claims 正确 |

见 [05-联合发版说明.md](./05-联合发版说明.md) 阶段 P0～P4。

---

## 6. 实施步骤

| 步骤 | § | 工作 | 预估 |
|------|---|------|------|
| 1 | A | EF 迁移 `AccessCode` | 0.5d |
| 2 | A | Pull Worker + 脏数据脚本 | 1d |
| 3 | A | Verify / 政府出站改用 `AccessCode` | 0.5d |
| 4 | B | BasePlatform HTTP 客户端 + 代理 API | 1d |
| 5 | B | 下线 `UrbanLicenseGenerator` + flag | 0.5d |
| 6 | B | Hub 推送联调 | 0.5d |
| **合计** | | | **~4d** |

可与 02、03 并行开发；**上线**须按 05 顺序。

---

## 7. 测试清单

| # | 场景 | 预期 |
|---|------|------|
| 1 | Pull 后 `GovProject.AccessCode` | 与 BasePlatform 一致 |
| 2 | Verify 用 AccessCode | 通过/失败正确 |
| 3 | 代理下载 license | 与 BasePlatform 直连接果一致 |
| 4 | Hub 推送 JWT | 客户端验签通过 |
| 5 | `UseBasePlatformJwtIssuer=false` | 旧路径仍可用（灰度） |
| 6 | 政府上传 buildLicenseNo | 值为 AccessCode |

---

## 8. 回滚

- §A：保留 `BuildLicenseNo` 列只读别名一版（EF 兼容属性）。  
- §B：`UseBasePlatformJwtIssuer=false` 恢复 Urban 本地签发（P4 前）。

---

## 9. 文档索引

| 编号 | 文档 |
|------|------|
| 02 | [02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md](./02-BasePlatform-AccessCode分列与ListProjects拟稿提案.md) | AccessCode 分列与 ListProjects |
| 03 | BasePlatform JWT 签发 |
| 04 | 本文档 |
| 05 | 联合发版 |

---

**文档版本**：0.1（拟稿）  
**最后更新**：2026-06-24
