# 05 - SignalR 技术选型与集成

## 1. 选型结论（已定）

Monitor 通道在**应用层**采用 **ASP.NET Core SignalR**；底层传输仍为 **WebSocket**，与 [00-调研总览](./00-调研总览.md)「仅长连接、不轮询降级」一致。

| 端 | 包 / 模块 | 说明 |
|----|-----------|------|
| **UrbanManagement（服务端）** | `Volo.Abp.AspNetCore.SignalR` | ABP 集成：自动 `AddSignalR`、端点映射；Hub 可继承 `AbpHub` |
| **MaterialClient / MaterialClient.Urban（客户端）** | `Microsoft.AspNetCore.SignalR.Client` | 标准 SignalR .NET 客户端；**无** `Volo.Abp.AspNetCore.SignalR`（该包仅用于 ASP.NET Core 宿主） |
| **MaterialClient 桌面宿主** | 已有 `Volo.Abp.Core` + `AbpModule` | 领域/DI 仍走 ABP；SignalR 连接在 Monitor 专用 `IHostedService` 中 |

**不采用**：手写 `ClientWebSocket` 收发循环（工作量大、重连需自研）。可选备选 `Websocket.Client` 仅在与非 SignalR 服务端对接时考虑。

---

## 2. 与「仅 WebSocket」的关系

SignalR 协商后默认走 WebSocket。需**禁止** Long Polling / SSE 降级：

```csharp
// 客户端 HubConnectionBuilder
.WithUrl(hubUrl, options =>
{
    options.Transports = HttpTransportType.WebSockets;
    options.SkipNegotiation = true; // 可选：固定 ws 端点时，需与服务端 MapHub 路径一致
})
.WithAutomaticReconnect()
```

服务端在 `Program` / 模块中保持常规 `MapHub`，不配置 SignalR 回退传输即可（勿启用仅用于降级的中间件策略）。

---

## 3. 服务端集成（UrbanManagement）

### 3.1 包与模块

```bash
# UrbanManagement.App 项目
dotnet add package Volo.Abp.AspNetCore.SignalR
```

```csharp
// UrbanManagementAppModule.cs
[DependsOn(
    typeof(AbpAspNetCoreMvcModule),
    typeof(AbpAspNetCoreSignalRModule),  // 新增
    typeof(UrbanManagementCoreModule)
)]
public class UrbanManagementAppModule : AbpModule { }
```

> `AbpAspNetCoreSignalRModule` 负责 SignalR 服务注册，无需手写 `services.AddSignalR()`。

### 3.2 Hub 设计

统一入口 Hub，按 Hub 方法或单一 `Dispatch` 方法承载多消息类型（与文档中 `heartbeat` / `device_status` / `error_log` 一致）：

```csharp
// src/UrbanManagement.App/Hubs/DeviceMonitorHub.cs
public class DeviceMonitorHub : AbpHub  // 或 Hub，按需 [Authorize]
{
    private readonly IDeviceStatusService _deviceStatusService;
    private readonly IErrorLogIngestionService _errorLogIngestionService;

    public async Task Heartbeat(HeartbeatDto dto)
    {
        await _deviceStatusService.UpdateHeartbeatAsync(dto);
    }

    public async Task ReportDeviceStatus(DeviceStatusDto dto)
    {
        await _deviceStatusService.UpdateDeviceStatusAsync(dto);
    }

    public async Task ReportCameraStatus(Dictionary<string, CameraStatusInfo> statuses)
    {
        await _deviceStatusService.UpdateCameraStatusAsync(statuses);
    }

    public async Task ReportErrorLog(ErrorLogEntry entry)
    {
        await _errorLogIngestionService.IngestAsync(entry);
    }

    public async Task ReportCriticalErrorLog(ErrorLogEntry entry)
    {
        await _errorLogIngestionService.IngestCriticalAsync(entry);
    }
}
```

路由映射（ABP 10 典型写法）：

```csharp
// 在 Configure 或模块 OnApplicationInitialization 中
app.UseEndpoints(endpoints =>
{
    endpoints.MapHub<DeviceMonitorHub>("/hubs/device-monitor");
});
```

管理端查询仍用 **HTTP API**（`MonitoringController`），与 SignalR 推送分离。

### 3.3 鉴权（可选）

工地 Monitor 若需对接 UrbanManagement 身份：

- Hub 方法或 Hub 类加 `[Authorize]`
- 客户端 `AccessTokenProvider` 返回 JWT（与 PFX/Bearer 方案对齐时见 `docs/2026-05-27-urban-management-pfx-auth-solution/`）

---

## 4. 客户端集成（MaterialClient.Urban）

### 4.1 包

```bash
# MaterialClient.Urban 或 MaterialClient.Common（Monitor 实现所在项目）
dotnet add package Microsoft.AspNetCore.SignalR.Client
```

**不要**在桌面项目引用 `Volo.Abp.AspNetCore.SignalR`。

### 4.2 连接服务示例

```csharp
// src/MaterialClient.Urban/Services/DeviceMonitorHubConnectionService.cs
public class DeviceMonitorHubConnectionService : IDeviceMonitorHubConnection, IAsyncDisposable
{
    private HubConnection? _connection;
    private readonly IConfiguration _configuration;
    private readonly ILogger<DeviceMonitorHubConnectionService> _logger;

    public bool IsConnected => _connection?.State == HubConnectionState.Connected;

    public async Task ConnectAsync(CancellationToken cancellationToken = default)
    {
        var baseUrl = _configuration["UrbanManagement:ServerUrl"]!.TrimEnd('/');
        var hubPath = _configuration["DeviceMonitoring:SignalRHubPath"] ?? "/hubs/device-monitor";
        var hubUrl = $"{baseUrl}{hubPath}";

        _connection = new HubConnectionBuilder()
            .WithUrl(hubUrl, options =>
            {
                options.Transports = HttpTransportType.WebSockets;
                // options.AccessTokenProvider = () => GetTokenAsync(); // 若启用鉴权
            })
            .WithAutomaticReconnect()
            .Build();

        _connection.Reconnected += async _ =>
        {
            _logger.LogInformation("SignalR reconnected");
            await Task.CompletedTask;
        };

        _connection.Closed += async ex =>
        {
            _logger.LogWarning(ex, "SignalR connection closed");
            await Task.CompletedTask;
        };

        await _connection.StartAsync(cancellationToken);
    }

    public Task SendHeartbeatAsync(HeartbeatDto dto, CancellationToken ct = default)
        => _connection!.InvokeAsync("Heartbeat", dto, ct);

    public Task SendDeviceStatusAsync(DeviceStatusDto dto, CancellationToken ct = default)
        => _connection!.InvokeAsync("ReportDeviceStatus", dto, ct);

    public Task SendErrorLogAsync(ErrorLogEntry entry, CancellationToken ct = default)
        => _connection!.InvokeAsync("ReportErrorLog", entry, ct);

    public async ValueTask DisposeAsync()
    {
        if (_connection != null)
            await _connection.DisposeAsync();
    }
}
```

### 4.3 与 Monitor 总开关

`DeviceMonitoring:Enabled == false` 时：

- 不创建 `HubConnection`
- 不调用 `StartAsync`
- 可注册 `IDeviceMonitorHubConnection` 的空实现

`IHostedService` 内 `try/catch` 包裹 `ConnectAsync` / `InvokeAsync`，**异常不得向上抛出到 UI 或称重服务**。

---

## 5. 配置项

```json
{
  "UrbanManagement": {
    "ServerUrl": "https://urban.example.com",
    "ClientId": "urban-client-001"
  },
  "DeviceMonitoring": {
    "Enabled": true,
    "SignalRHubPath": "/hubs/device-monitor",
    "EnableHeartbeat": true,
    "HeartbeatInterval": "00:00:30"
  }
}
```

| 配置键 | 说明 |
|--------|------|
| `DeviceMonitoring:Enabled` | Monitor 总开关（含 SignalR 连接） |
| `DeviceMonitoring:SignalRHubPath` | Hub 路径，需与服务端 `MapHub` 一致 |
| `UrbanManagement:ServerUrl` | HTTP 基地址；SignalR 在同源下使用该 URL + Hub 路径 |

---

## 6. 消息类型与文档映射

| 原设计消息 type | SignalR Hub 方法 |
|----------------|------------------|
| `heartbeat` | `Heartbeat` |
| `device_status` | `ReportDeviceStatus` |
| `camera_status` | `ReportCameraStatus` |
| `error_log` | `ReportErrorLog` |
| `error_log_critical` | `ReportCriticalErrorLog` |
| `error_log_replay` | `ReportErrorLog`（批量可增 `ReportErrorLogBatch`） |

---

## 7. 实施检查清单

- [ ] UrbanManagement.App 引用 `Volo.Abp.AspNetCore.SignalR` 并依赖 `AbpAspNetCoreSignalRModule`
- [ ] 实现 `DeviceMonitorHub` 并 `MapHub` 到 `/hubs/device-monitor`
- [ ] 客户端引用 `Microsoft.AspNetCore.SignalR.Client`，实现 `IDeviceMonitorHubConnection`
- [ ] 客户端 `Transports = WebSockets`，不依赖 Long Polling
- [ ] `DeviceMonitoring:Enabled` 控制是否 `StartAsync`
- [ ] 称重 `UrbanServerUploadService` 不引用 Hub 连接服务
- [ ] 断线重连由 `WithAutomaticReconnect` + 本地队列补传（错误日志）覆盖

---

## 8. 相关文档

| 文档 | 内容 |
|------|------|
| [01-设备实时同步方案.md](./01-设备实时同步方案.md) | 设备状态推送业务 |
| [02-状态监控方案.md](./02-状态监控方案.md) | 心跳与摄像头 |
| [03-错误日志方案.md](./03-错误日志方案.md) | 错误日志经 Hub 上报 |
| [04-实施建议.md](./04-实施建议.md) | 阶段实施与配置 |

---

**参考**：[ABP SignalR Integration](https://abp.io/docs/latest/framework/real-time/signalr) · [ASP.NET Core SignalR .NET Client](https://learn.microsoft.com/en-us/aspnet/core/signalr/dotnet-client)
