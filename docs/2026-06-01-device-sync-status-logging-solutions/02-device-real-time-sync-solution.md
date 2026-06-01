# 设备实时同步方案

## 方案概述

本方案设计 MaterialClient.Urban 到 UrbanManagement 的实时设备信息同步机制，解决当前批量同步延迟问题，实现设备状态的实时推送和监控。

## 现状分析

### 当前同步机制
- **同步方式**: 后台定时任务批量查询待上传记录
- **同步频率**: 定时轮询（默认 5 分钟间隔）
- **同步内容**: 仅称重记录数据
- **同步延迟**: 最长可达 5 分钟
- **同步状态**: 通过 `UrbanWeighingExtension` 表跟踪

### 现有限制
- 称重记录创建到服务端接收存在延迟
- 缺乏设备状态变化的实时通知
- 无法实时反映客户端工作状态
- 服务端无法主动查询客户端状态

## 解决方案设计

### 方案一：WebSocket 实时推送（推荐）

#### 架构设计
```
MaterialClient.Urban                     UrbanManagement
┌─────────────────┐                      ┌──────────────────────┐
│ DeviceMonitor   │◄──── WebSocket ──────┤│ DeviceStatusHub      │
│ Service         │      实时推送         ││                      │
│                 │                      ││ - BroadcastStatus()  │
│ ┌─────────────┐ │                      ││ - UpdateDevice()     │
│ │ DeviceState  │─┤                     ││ - QueryDevice()      │
│ │ Manager      │ │                      │└──────────────────────┘
│ └─────────────┘ │                               │
│                 │                               │
│ ┌─────────────┐ │                      ┌──────────────────────────┐
│ │ Weighing    │ │                      ││ DeviceStatusService     │
│ │ Pipeline    │ │                      ││ - StoreDeviceStatus()   │
│ └─────────────┘ │                      ││ - GetDeviceStatus()     │
└─────────────────┘                      │└──────────────────────────┘
```

#### 实施步骤

**1. MaterialClient.Urban 端实现**

创建设备监控服务：

```csharp
// src/MaterialClient.Urban/Services/DeviceStatusWebSocketService.cs
public class DeviceStatusWebSocketService : IDeviceStatusWebSocketService
{
    private readonly ILogger<DeviceStatusWebSocketService> _logger;
    private readonly IDeviceStateManager _stateManager;
    private readonly HttpClient _httpClient;
    private WebSocket? _webSocket;
    private readonly CancellationTokenSource _cts = new();

    public async Task ConnectAsync(string serverUrl)
    {
        var wsUrl = serverUrl.Replace("http", "ws") + "/ws/devicestatus";
        _webSocket = new ClientWebSocket();
        
        try
        {
            await _webSocket.ConnectAsync(new Uri(wsUrl), _cts.Token);
            _logger.LogInformation("Device status WebSocket connected to {Url}", wsUrl);
            
            // Start listening for server commands
            _ = Task.Run(() => ListenForServerCommandsAsync());
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to connect to device status WebSocket");
        }
    }

    public async Task SendDeviceStatusAsync(DeviceStatusDto status)
    {
        if (_webSocket?.State == WebSocketState.Open)
        {
            var json = JsonSerializer.Serialize(status);
            var buffer = Encoding.UTF8.GetBytes(json);
            
            await _webSocket.SendAsync(
                new ArraySegment<byte>(buffer),
                WebSocketMessageType.Text,
                true,
                _cts.Token);
        }
    }
}
```

**2. UrbanManagement 端实现**

创建 WebSocket Hub：

```csharp
// src/UrbanManagement.App/Hubs/DeviceStatusHub.cs
[HubPath("/ws/devicestatus")]
public class DeviceStatusHub : Hub
{
    private readonly IDeviceStatusService _deviceStatusService;
    private readonly ILogger<DeviceStatusHub> _logger;

    public override async Task OnConnectedAsync()
    {
        var clientId = Context.GetHttpContext()?.Request.Query["clientId"];
        _logger.LogInformation("Device {ClientId} connected", clientId);
        
        // Send current server time for sync
        await Clients.Caller.SendAsync("ServerTime", DateTime.UtcNow);
        
        await base.OnConnectedAsync();
    }

    public async Task ReportStatus(DeviceStatusReportDto report)
    {
        await _deviceStatusService.UpdateDeviceStatusAsync(report);
        
        // Broadcast to all connected admin clients
        await Clients.Others.SendAsync("DeviceStatusUpdate", report);
    }

    public async Task Heartbeat(string clientId)
    {
        await _deviceStatusService.UpdateHeartbeatAsync(clientId);
        await Clients.Caller.SendAsync("HeartbeatAck", DateTime.UtcNow);
    }
}
```

**3. 数据传输模型**

```csharp
// src/MaterialClient.Urban/Dtos/DeviceStatusDto.cs
public class DeviceStatusDto
{
    public string ClientId { get; set; } = string.Empty;
    public DateTime ReportTime { get; set; }
    public ClientStatusInfo ClientInfo { get; set; } = new();
    public List<DeviceStatusInfo> Devices { get; set; } = new();
}

public class ClientStatusInfo
{
    public string MachineName { get; set; } = string.Empty;
    public string IpAddress { get; set; } = string.Empty;
    public string AppVersion { get; set; } = string.Empty;
    public DateTime StartTime { get; set; }
    public bool IsOnline { get; set; }
    public double CpuUsage { get; set; }
    public double MemoryUsage { get; set; }
}

public class DeviceStatusInfo
{
    public string DeviceType { get; set; } = string.Empty; // "Camera", "Scale", "LPR", "GateIO"
    public string DeviceId { get; set; } = string.Empty;
    public string DeviceName { get; set; } = string.Empty;
    public bool IsOnline { get; set; }
    public string? StatusMessage { get; set; }
    public DateTime? LastCommunication { get; set; }
}
```

#### 优势
- ✅ **实时性强**: 毫秒级状态更新延迟
- ✅ **双向通信**: 服务端可主动查询客户端状态
- ✅ **带宽效率**: 相比 HTTP 轮询减少 90% 网络流量
- ✅ **连接复用**: 单个连接处理多种消息类型

#### 风险与缓解
- ⚠️ **防火墙限制**: 部分网络可能阻断 WebSocket
  - 缓解: 提供 HTTP 长轮询备选方案
- ⚠️ **连接稳定性**: 长时间连接可能断开
  - 缓解: 实现自动重连和心跳检测
- ⚠️ **服务端资源**: 大量并发连接占用内存
  - 缓解: 设置连接超时和负载均衡

---

### 方案二：HTTP 长轮询（备选）

#### 实施步骤

**1. 客户端实现**

```csharp
// src/MaterialClient.Urban/Services/DeviceStatusPollingService.cs
public class DeviceStatusPollingService : IDeviceStatusPollingService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IDeviceStateManager _stateManager;
    private readonly ILogger<DeviceStatusPollingService> _logger;
    
    public async Task StartPollingAsync(string serverUrl)
    {
        while (!_cts.IsCancellationRequested)
        {
            try
            {
                var client = _httpClientFactory.CreateClient();
                var status = await _stateManager.GetCurrentStatusAsync();
                
                var response = await client.PostAsJsonAsync(
                    $"{serverUrl}/api/device-status/report", 
                    status);
                    
                if (response.IsSuccessStatusCode)
                {
                    // Check for server commands
                    var commands = await response.Content.ReadFromJsonAsync<ServerCommandsDto>();
                    await ProcessServerCommandsAsync(commands);
                }
                
                // Adaptive delay based on activity
                var delay = status.HasRecentChanges ? TimeSpan.FromSeconds(5) : TimeSpan.FromSeconds(30);
                await Task.Delay(delay, _cts.Token);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Device status polling failed");
                await Task.Delay(TimeSpan.FromMinutes(1), _cts.Token);
            }
        }
    }
}
```

**2. 服务端 API**

```csharp
// src/UrbanManagement.App/Controllers/DeviceStatusController.cs
[ApiController]
[Route("api/[controller]")]
public class DeviceStatusController : ControllerBase
{
    private readonly IDeviceStatusService _deviceStatusService;
    
    [HttpPost("report")]
    public async Task<IActionResult> ReportDeviceStatus([FromBody] DeviceStatusDto status)
    {
        await _deviceStatusService.UpdateDeviceStatusAsync(status);
        
        // Return any pending commands for the client
        var commands = await _deviceStatusService.GetPendingCommandsAsync(status.ClientId);
        return Ok(new { success = true, commands });
    }
    
    [HttpGet("active-clients")]
    public async Task<IActionResult> GetActiveClients()
    {
        var clients = await _deviceStatusService.GetActiveClientsAsync();
        return Ok(clients);
    }
}
```

#### 优势
- ✅ **网络兼容性**: 适用于所有网络环境
- ✅ **实施简单**: 基于现有 HTTP 基础设施
- ✅ **无状态设计**: 服务端无需维持连接状态

#### 劣势
- ⚠️ **实时性差**: 轮询间隔导致延迟（5-30秒）
- ⚠️ **资源浪费**: 无数据变化时仍产生网络流量
- ⚠️ **服务端压力大**: 高频轮询增加服务器负载

---

### 方案三：混合方案（生产推荐）

结合 WebSocket 和 HTTP 轮询的优势，提供最佳用户体验。

#### 架构设计
```
优先 WebSocket，降级到 HTTP 长轮询
┌─────────────────────┐
│ MaterialClient      │
│                     │
│ 1. 尝试 WebSocket  │─────┐
│    连接              │     │ 成功
│                     │◄────┘ 使用 WebSocket
│ 2. 失败则使用 HTTP   │
│    长轮询            │─────── HTTP 备用
└─────────────────────┘
```

#### 实施策略

```csharp
// src/MaterialClient.Urban/Services/HybridDeviceSyncService.cs
public class HybridDeviceSyncService : IHybridDeviceSyncService
{
    public async Task StartSyncAsync(string serverUrl)
    {
        // Try WebSocket first
        try
        {
            await _webSocketService.ConnectAsync(serverUrl);
            await _webSocketService.StartStatusReportingAsync();
            _logger.LogInformation("Using WebSocket for device sync");
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "WebSocket failed, falling back to HTTP polling");
            await _pollingService.StartPollingAsync(serverUrl);
        }
    }
    
    public async Task SendDeviceStatusAsync(DeviceStatusDto status)
    {
        if (_webSocketService.IsConnected)
        {
            await _webSocketService.SendDeviceStatusAsync(status);
        }
        else
        {
            await _pollingService.SendDeviceStatusAsync(status);
        }
    }
}
```

## 数据模型设计

### 服务端设备状态表

```csharp
// src/UrbanManagement.Core/Entities/DeviceStatus.cs
public class DeviceStatus : Entity<Guid>
{
    public string ClientId { get; set; } = string.Empty;
    public string MachineName { get; set; } = string.Empty;
    public string IpAddress { get; set; } = string.Empty;
    public string AppVersion { get; set; } = string.Empty;
    public DateTime LastHeartbeat { get; set; }
    public DateTime? LastOffline { get; set; }
    public bool IsOnline { get; set; }
    public double? CpuUsage { get; set; }
    public double? MemoryUsage { get; set; }
    public DateTime StatusUpdateTime { get; set; }
}

public class DeviceComponentStatus : Entity<Guid>
{
    public string DeviceClientId { get; set; } = string.Empty; // FK to DeviceStatus
    public string DeviceType { get; set; } = string.Empty; // Camera, Scale, LPR, GateIO
    public string DeviceId { get; set; } = string.Empty;
    public string DeviceName { get; set; } = string.Empty;
    public bool IsOnline { get; set; }
    public string? StatusMessage { get; set; }
    public DateTime? LastCommunication { get; set; }
    public DateTime StatusUpdateTime { get; set; }
}
```

### 在线状态计算逻辑

```csharp
public class OnlineStatusCalculator
{
    public static bool IsOnline(DateTime lastHeartbeat, TimeSpan timeout)
    {
        return DateTime.UtcNow - lastHeartbeat < timeout;
    }
    
    public static OnlineStatus GetStatus(TimeSpan timeSinceLastHeartbeat)
    {
        return timeSinceLastHeartbeat switch
        {
            { TotalSeconds: < 30 } => OnlineStatus.Online,
            { TotalSeconds: < 300 } => OnlineStatus.Idle,
            { TotalSeconds: < 900 } => OnlineStatus.Unstable,
            _ => OnlineStatus.Offline
        };
    }
}
```

## 性能优化建议

### 1. 数据压缩
```csharp
// 使用 MessagePack 代替 JSON 减少数据大小
var messagePackData = MessagePackSerializer.Serialize(status);
await webSocket.SendAsync(messagePackData, WebSocketMessageType.Binary, true, token);
```

### 2. 批量状态更新
```csharp
// 每 10 秒批量提交一次，而非每次变化都提交
public class DeviceStatusBatcher
{
    private readonly List<DeviceStatusDto> _buffer = new();
    private readonly Timer _timer;
    
    public DeviceStatusBatcher()
    {
        _timer = new Timer(async _ => await FlushBufferAsync(), 
                          null, TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(10));
    }
    
    public async Task AddStatusAsync(DeviceStatusDto status)
    {
        _buffer.Add(status);
        if (_buffer.Count >= 50) // 达到阈值立即发送
        {
            await FlushBufferAsync();
        }
    }
}
```

### 3. 服务端缓存
```csharp
// Redis 缓存设备状态，减少数据库查询
public class CachedDeviceStatusService : IDeviceStatusService
{
    private readonly IDistributedCache _cache;
    
    public async Task<DeviceStatusDto?> GetDeviceStatusAsync(string clientId)
    {
        var cacheKey = $"device_status:{clientId}";
        var cachedData = await _cache.GetStringAsync(cacheKey);
        
        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<DeviceStatusDto>(cachedData);
        }
        
        // 缓存未命中，从数据库查询
        var status = await _repository.GetAsync(clientId);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(status), 
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
        
        return status;
    }
}
```

## 监控和告警

### 关键指标监控

```csharp
// 监控指标收集
public class DeviceSyncMetrics
{
    public int ActiveConnections { get; set; }
    public double AverageMessageLatency { get; set; }
    public int FailedConnections { get; set; }
    public int MessagesPerSecond { get; set; }
    public double CpuUsage { get; set; }
    public double MemoryUsage { get; set; }
}
```

### 告警规则配置

```csharp
// 告警规则定义
public class AlertRule
{
    public string MetricName { get; set; } = string.Empty;
    public double Threshold { get; set; }
    public ComparisonOperator Operator { get; set; }
    public TimeSpan EvaluationPeriod { get; set; }
    public AlertSeverity Severity { get; set; }
    public string ActionUrl { get; set; } = string.Empty;
}

// 示例规则
var alertRules = new[]
{
    new AlertRule 
    { 
        MetricName = "DeviceOfflineTime", 
        Threshold = 15, // 15分钟
        Operator = ComparisonOperator.GreaterThan,
        Severity = AlertSeverity.High 
    },
    new AlertRule 
    { 
        MetricName = "ConnectionFailureRate", 
        Threshold = 0.1, // 10% 失败率
        Operator = ComparisonOperator.GreaterThan,
        Severity = AlertSeverity.Medium 
    }
};
```

---

**下一步**: 参考 [状态监控方案](./03-status-monitoring-solution.md) 了解具体的状态监控实现细节。