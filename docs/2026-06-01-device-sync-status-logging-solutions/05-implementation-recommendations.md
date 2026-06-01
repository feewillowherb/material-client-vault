# 实施建议

## 实施优先级和路线图

### 第一阶段：基础设施搭建（1-2周）

#### 1. 数据模型和数据库设置

**步骤清单:**

1. **创建设备状态表**
```sql
-- 在 UrbanManagement 数据库中创建设备状态表
CREATE TABLE DeviceStatus (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    ClientId NVARCHAR(50) UNIQUE NOT NULL,
    MachineName NVARCHAR(100),
    IpAddress NVARCHAR(50),
    AppVersion NVARCHAR(20),
    LastHeartbeat DATETIME2 NOT NULL,
    LastOffline DATETIME2,
    IsOnline BIT NOT NULL DEFAULT 0,
    CpuUsage REAL,
    MemoryUsage REAL,
    StatusUpdateTime DATETIME2 NOT NULL,
    CreatedTime DATETIME2 NOT NULL,
    INDEX IX_DeviceStatus_ClientId (ClientId),
    INDEX IX_DeviceStatus_LastHeartbeat (LastHeartbeat DESC)
);

CREATE TABLE DeviceComponentStatus (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    DeviceClientId NVARCHAR(50) NOT NULL,
    DeviceType NVARCHAR(20) NOT NULL,
    DeviceId NVARCHAR(50) NOT NULL,
    DeviceName NVARCHAR(100),
    IsOnline BIT NOT NULL DEFAULT 0,
    StatusMessage NVARCHAR(500),
    LastCommunication DATETIME2,
    StatusUpdateTime DATETIME2 NOT NULL,
    INDEX IX_DeviceComponentStatus_ClientId (DeviceClientId),
    INDEX IX_DeviceComponentStatus_DeviceId (DeviceId)
);
```

2. **创建错误日志表**
```sql
CREATE TABLE ErrorLogs (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    Timestamp DATETIME2 NOT NULL,
    Level INT NOT NULL,
    Message NVARCHAR(MAX) NOT NULL,
    StackTrace NVARCHAR(MAX),
    Source NVARCHAR(100),
    ClientId NVARCHAR(50) NOT NULL,
    MachineName NVARCHAR(100),
    AppVersion NVARCHAR(20),
    AdditionalData NVARCHAR(MAX),
    
    -- 分析字段
    ErrorPattern NVARCHAR(100),
    ErrorCategory INT,
    Severity INT,
    IsProcessed BIT NOT NULL DEFAULT 0,
    ProcessedTime DATETIME2,
    
    -- 审计字段
    CreatedTime DATETIME2 NOT NULL,
    LastModifiedTime DATETIME2,
    
    INDEX IX_ErrorLogs_Timestamp (Timestamp DESC),
    INDEX IX_ErrorLogs_ClientId (ClientId),
    INDEX IX_ErrorLogs_Level (Level),
    INDEX IX_ErrorLogs_ErrorCategory (ErrorCategory)
);
```

3. **添加 ABP 实体类**
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

// 添加 DbContext 配置
public class UrbanManagementDbContext : AbpDbContext<UrbanManagementDbContext>
{
    public DbSet<DeviceStatus> DeviceStatus { get; set; }
    public DbSet<ErrorLog> ErrorLogs { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        builder.Entity<DeviceStatus>(b =>
        {
            b.ToTable("DeviceStatus");
            b.HasKey(x => x.Id);
            b.HasIndex(x => x.ClientId).IsUnique();
            b.HasIndex(x => x.LastHeartbeat);
        });
    }
}
```

#### 2. 核心服务接口定义

**客户端服务接口:**
```csharp
// src/MaterialClient.Urban/Services/IDeviceStatusReporter.cs
public interface IDeviceStatusReporter
{
    Task ReportDeviceStatusAsync(DeviceStatusDto status);
    Task ReportHeartbeatAsync(HeartbeatDto heartbeat);
    Task ReportCameraStatusAsync(Dictionary<string, CameraStatusInfo> cameraStatus);
}

// src/MaterialClient.Urban/Services/IErrorLogCaptureService.cs
public interface IErrorLogCaptureService
{
    void CaptureError(Exception exception, ErrorContext context);
    void CaptureWarning(string message, WarningContext context);
    Task<List<ErrorLogEntry>> GetPendingLogsAsync();
}
```

**服务端服务接口:**
```csharp
// src/UrbanManagement.Core/Services/IDeviceStatusService.cs
public interface IDeviceStatusService
{
    Task UpdateDeviceStatusAsync(DeviceStatusDto status);
    Task UpdateHeartbeatAsync(string clientId);
    Task<DeviceStatusDto?> GetDeviceStatusAsync(string clientId);
    Task<List<DeviceStatusDto>> GetActiveClientsAsync();
}

// src/UrbanManagement.Core/Services/IErrorLogIngestionService.cs
public interface IErrorLogIngestionService
{
    Task<ErrorLogIngestionResult> IngestAsync(ErrorLogEntry logEntry);
    Task<ErrorLogIngestionResult> IngestCriticalAsync(ErrorLogEntry logEntry);
}
```

### 第二阶段：核心功能实现（2-3周）

#### 1. WebSocket 实时通信

**客户端实现:**
```csharp
// src/MaterialClient.Urban/Services/DeviceStatusWebSocketService.cs
public class DeviceStatusWebSocketService : IDeviceStatusWebSocketService
{
    private readonly ILogger<DeviceStatusWebSocketService> _logger;
    private readonly IConfiguration _configuration;
    private ClientWebSocket? _webSocket;
    private readonly CancellationTokenSource _cts = new();
    
    public async Task ConnectAsync()
    {
        var serverUrl = _configuration["UrbanManagement:ServerUrl"];
        var wsUrl = serverUrl.Replace("http", "ws") + "/ws/devicestatus";
        
        _webSocket = new ClientWebSocket();
        
        try
        {
            await _webSocket.ConnectAsync(new Uri(wsUrl), _cts.Token);
            _logger.LogInformation("WebSocket connected to {Url}", wsUrl);
            
            // 启动心跳和状态报告
            _ = Task.Run(() => StartHeartbeatAsync());
            _ = Task.Run(() => StartStatusReportingAsync());
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "WebSocket connection failed");
            throw;
        }
    }
    
    private async Task StartHeartbeatAsync()
    {
        while (_webSocket?.State == WebSocketState.Open && !_cts.IsCancellationRequested)
        {
            try
            {
                var heartbeat = new HeartbeatDto
                {
                    ClientId = _configuration["Client:Id"],
                    Timestamp = DateTime.UtcNow,
                    Status = await _statusMonitor.GetCurrentStatusAsync()
                };
                
                await SendMessageAsync("heartbeat", heartbeat);
                await Task.Delay(TimeSpan.FromSeconds(30), _cts.Token);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Heartbeat failed");
                break;
            }
        }
    }
    
    private async Task StartStatusReportingAsync()
    {
        while (_webSocket?.State == WebSocketState.Open && !_cts.IsCancellationRequested)
        {
            try
            {
                var deviceStatus = await CollectDeviceStatusAsync();
                await SendMessageAsync("device_status", deviceStatus);
                await Task.Delay(TimeSpan.FromMinutes(1), _cts.Token);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Status reporting failed");
                break;
            }
        }
    }
}
```

**服务端实现:**
```csharp
// src/UrbanManagement.App/Hubs/DeviceStatusHub.cs
[HubPath("/ws/devicestatus")]
public class DeviceStatusHub : Hub
{
    private readonly IDeviceStatusService _deviceStatusService;
    private readonly IHubContext<DeviceStatusHub> _hubContext;
    private readonly ILogger<DeviceStatusHub> _logger;
    
    public override async Task OnConnectedAsync()
    {
        var clientId = Context.GetHttpContext()?.Request.Query["clientId"];
        _logger.LogInformation("Client {ClientId} connected", clientId);
        
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var clientId = Context.GetHttpContext()?.Request.Query["clientId"];
        _logger.LogInformation("Client {ClientId} disconnected", clientId);
        
        // 标记客户端为离线
        if (!string.IsNullOrEmpty(clientId))
        {
            await _deviceStatusService.MarkOfflineAsync(clientId);
        }
        
        await base.OnDisconnectedAsync(exception);
    }
    
    [HubMethodName("heartbeat")]
    public async Task HandleHeartbeat(HeartbeatDto heartbeat)
    {
        await _deviceStatusService.UpdateHeartbeatAsync(heartbeat.ClientId);
        
        // 广播给所有管理员
        await _hubContext.Clients.Group("admins").SendAsync("ClientHeartbeat", heartbeat);
    }
    
    [HubMethodName("device_status")]
    public async Task HandleDeviceStatus(DeviceStatusDto status)
    {
        await _deviceStatusService.UpdateDeviceStatusAsync(status);
        
        // 广播状态更新
        await _hubContext.Clients.Group("admins").SendAsync("DeviceStatusUpdate", status);
    }
}
```

#### 2. 错误日志收集和传输

**客户端错误捕获:**
```csharp
// src/MaterialClient.Urban/Services/ErrorLogCaptureService.cs
public class ErrorLogCaptureService : IErrorLogCaptureService
{
    private readonly ConcurrentQueue<ErrorLogEntry> _logQueue;
    private readonly IErrorLogTransmitter _transmitter;
    private readonly ILogger<ErrorLogCaptureService> _logger;
    
    public ErrorLogCaptureService(IErrorLogTransmitter transmitter)
    {
        _transmitter = transmitter;
        _logQueue = new ConcurrentQueue<ErrorLogEntry>();
        
        // 启动后台传输任务
        Task.Run(async () => await ProcessLogQueueAsync());
    }
    
    public void CaptureError(Exception exception, ErrorContext context)
    {
        var logEntry = new ErrorLogEntry
        {
            Id = Guid.NewGuid(),
            Timestamp = DateTime.UtcNow,
            Level = LogLevel.Error,
            Message = exception.Message,
            StackTrace = exception.StackTrace,
            Source = context.Source,
            ClientId = context.ClientId,
            MachineName = Environment.MachineName,
            AppVersion = context.AppVersion,
            AdditionalData = JsonSerializer.Serialize(context.AdditionalData)
        };
        
        _logQueue.Enqueue(logEntry);
        
        _logger.LogError(exception, "Error captured: {Message}", exception.Message);
    }
    
    private async Task ProcessLogQueueAsync()
    {
        while (!_cts.IsCancellationRequested)
        {
            try
            {
                if (_logQueue.TryDequeue(out var logEntry))
                {
                    await _transmitter.TransmitAsync(logEntry);
                }
                else
                {
                    await Task.Delay(TimeSpan.FromMilliseconds(100), _cts.Token);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process log queue");
                await Task.Delay(TimeSpan.FromSeconds(5), _cts.Token);
            }
        }
    }
}
```

**集成到现有应用:**
```csharp
// src/MaterialClient.Urban/App.axaml.cs
public partial class App : Application
{
    private void ConfigureServices(ServiceCollection services)
    {
        // 添加现有服务...
        
        // 添加新的监控和日志服务
        services.AddTransient<IDeviceStatusReporter, DeviceStatusReporter>();
        services.AddTransient<IErrorLogCaptureService, ErrorLogCaptureService>();
        services.AddSingleton<IDeviceStatusMonitor, DeviceStatusMonitor>();
        services.AddHostedService<DeviceStatusBackgroundService>();
    }
}

// 全局异常处理
public class GlobalExceptionHandler
{
    private readonly IErrorLogCaptureService _errorLogCapture;
    
    public async Task HandleExceptionAsync(Exception exception)
    {
        var context = new ErrorContext
        {
            Source = "GlobalExceptionHandler",
            ClientId = _clientId,
            AppVersion = _appVersion
        };
        
        _errorLogCapture.CaptureError(exception, context);
    }
}
```

### 第三阶段：监控界面和分析（1-2周）

#### 1. 创建监控 API 端点

```csharp
// src/UrbanManagement.App/Controllers/MonitoringController.cs
[ApiController]
[Route("api/[controller]")]
public class MonitoringController : AbpController
{
    [HttpGet("dashboard")]
    public async Task<IActionResult> GetDashboardData()
    {
        var dashboard = new MonitoringDashboardDto
        {
            Summary = await GetSummaryAsync(),
            ActiveClients = await GetActiveClientsAsync(),
            CameraStatus = await GetCameraStatusAsync(),
            RecentAlerts = await GetRecentAlertsAsync(),
            SystemHealth = await GetSystemHealthAsync()
        };
        
        return Ok(dashboard);
    }
    
    private async Task<SummaryStatistics> GetSummaryAsync()
    {
        var now = DateTime.UtcNow;
        var activeClients = await _deviceStatusService.GetActiveClientsAsync();
        
        return new SummaryStatistics
        {
            TotalClients = await _deviceStatusService.GetTotalClientCountAsync(),
            OnlineClients = activeClients.Count(c => c.IsOnline),
            OfflineClients = activeClients.Count(c => !c.IsOnline),
            TotalCameras = activeClients.Sum(c => c.TotalCameras),
            OnlineCameras = activeClients.Sum(c => c.OnlineCameras),
            OfflineCameras = activeClients.Sum(c => c.OfflineCameras),
            CriticalAlerts = await _alertService.GetCriticalAlertCountAsync(
                now.AddHours(-24), now)
        };
    }
}
```

#### 2. 创建前端监控界面

```html
<!-- Views/Monitoring/Dashboard.cshtml -->
@model MonitoringDashboardDto

<div class="monitoring-dashboard">
    <h1>设备监控仪表板</h1>
    
    <!-- 概览统计 -->
    <div class="summary-cards">
        <div class="card">
            <h3>客户端在线率</h3>
            <div class="metric">
                <span class="value">@Model.Summary.ClientOnlineRate.ToString("0.0")%</span>
                <span class="trend @(Model.Summary.ClientOnlineRate >= 90 ? "positive" : "negative")">
                    @((Model.Summary.ClientOnlineRate - 90).ToString("+0.0;-0.0")%)
                </span>
            </div>
            <div class="detail">
                @Model.Summary.OnlineClients / @Model.Summary.TotalClients 在线
            </div>
        </div>
        
        <div class="card">
            <h3>摄像头在线率</h3>
            <div class="metric">
                <span class="value">@Model.Summary.CameraOnlineRate.ToString("0.0")%</span>
            </div>
            <div class="detail">
                @Model.Summary.OnlineCameras / @Model.Summary.TotalCameras 在线
            </div>
        </div>
        
        <div class="card alert-card">
            <h3>严重告警</h3>
            <div class="metric alert @(Model.Summary.CriticalAlerts > 0 ? "critical" : "")">
                @Model.Summary.CriticalAlerts
            </div>
            <div class="detail">
                最近 24 小时
            </div>
        </div>
    </div>
    
    <!-- 客户端状态表格 -->
    <div class="client-status-section">
        <h2>客户端状态</h2>
        <table class="status-table">
            <thead>
                <tr>
                    <th>客户端名称</th>
                    <th>IP 地址</th>
                    <th>在线状态</th>
                    <th>最后心跳</th>
                    <th>摄像头状态</th>
                    <th>操作</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var client in Model.ActiveClients)
                {
                    <tr class="@(client.IsOnline ? "online" : "offline")">
                        <td>@client.MachineName</td>
                        <td>@client.IpAddress</td>
                        <td>
                            <span class="status-badge @(client.IsOnline ? "online" : "offline")">
                                @(client.IsOnline ? "在线" : "离线")
                            </span>
                        </td>
                        <td>@client.LastHeartbeat.ToString("MM-dd HH:mm:ss")</td>
                        <td>@client.OnlineCameras / @client.TotalCameras</td>
                        <td>
                            <a href="/Monitoring/ClientDetails/@client.ClientId">详情</a>
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>
```

### 第四阶段：测试和优化（1周）

#### 1. 单元测试

```csharp
// tests/MaterialClient.Urban.Tests/Services/DeviceStatusReporterTests.cs
public class DeviceStatusReporterTests
{
    [Test]
    public async Task ReportDeviceStatus_Should_Transmit_Successfully()
    {
        // Arrange
        var mockHttpMessageHandler = new MockHttpMessageHandler();
        var httpClient = new HttpClient(mockHttpMessageHandler);
        var reporter = new DeviceStatusReporter(httpClient, _configuration);
        
        var status = new DeviceStatusDto
        {
            ClientId = "test-client",
            MachineName = "TestMachine",
            IsOnline = true
        };
        
        // Act
        await reporter.ReportDeviceStatusAsync(status);
        
        // Assert
        mockHttpMessageHandler.VerifyRequest(HttpMethod.Post, "/api/device-status/report", Times.Once());
    }
}
```

#### 2. 性能测试和优化

```csharp
// tests/PerformanceTests/DeviceStatusPerformanceTests.cs
public class DeviceStatusPerformanceTests
{
    [Test]
    public async Task Concurrent_DeviceStatus_Updates_Should_Handle_Load()
    {
        // 并发测试
        var tasks = new List<Task>();
        for (int i = 0; i < 100; i++)
        {
            var status = new DeviceStatusDto { ClientId = $"client-{i}" };
            tasks.Add(_deviceStatusService.UpdateDeviceStatusAsync(status));
        }
        
        await Task.WhenAll(tasks);
        
        // 验证所有状态都成功更新
        var allClients = await _deviceStatusService.GetActiveClientsAsync();
        Assert.AreEqual(100, allClients.Count);
    }
}
```

## 配置和部署

### 客户端配置

```json
// src/MaterialClient.Urban/appsettings.json
{
  "UrbanManagement": {
    "ServerUrl": "http://localhost:5000",
    "ClientId": "urban-client-001",
    "ConnectionMode": "WebSocket",
    "HeartbeatInterval": "00:00:30",
    "StatusReportInterval": "00:01:00"
  },
  "ErrorLogging": {
    "EnableRemoteLogging": true,
    "LogLevel": "Information",
    "MaxBufferSize": 100,
    "UploadInterval": "00:00:30"
  },
  "DeviceMonitoring": {
    "EnableCameraMonitoring": true,
    "CameraCheckInterval": "00:00:30",
    "OfflineThreshold": "00:05:00"
  }
}
```

### 服务端配置

```json
// src/UrbanManagement.App/appsettings.json
{
  "ConnectionStrings": {
    "Default": "Data Source=UrbanManagement.db"
  },
  "WebSocket": {
    "Enabled": true,
    "Path": "/ws/devicestatus"
  },
  "DeviceMonitoring": {
    "OfflineThreshold": "00:15:00",
    "EnableAlerts": true
  },
  "ErrorLogging": {
    "RetentionDays": 90,
    "EnableCompression": true,
    "ArchiveInterval": "7.00:00:00"
  },
  "Alerts": {
    "Enabled": true,
    "EmailRecipients": ["admin@example.com"],
    "SmsRecipients": ["+1234567890"]
  }
}
```

## 监控和维护

### 健康检查端点

```csharp
// src/UrbanManagement.App/Controllers/HealthController.cs
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    [HttpGet]
    public IActionResult GetHealthStatus()
    {
        var health = new
        {
            Status = "Healthy",
            Timestamp = DateTime.UtcNow,
            Services = new
            {
                Database = CheckDatabaseHealth(),
                WebSocket = CheckWebSocketHealth(),
                ErrorLogs = CheckErrorLogServiceHealth()
            }
        };
        
        return Ok(health);
    }
    
    private object CheckDatabaseHealth()
    {
        try
        {
            _dbContext.Database.CanConnect();
            return new { Status = "Healthy", ResponseTime = "10ms" };
        }
        catch
        {
            return new { Status = "Unhealthy" };
        }
    }
}
```

### 日志轮转和清理

```csharp
// src/UrbanManagement.App/Services/LogMaintenanceService.cs
public class LogMaintenanceService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // 每天执行一次清理
                var now = DateTime.UtcNow;
                if (now.Hour == 2 && now.Minute < 5) // 凌晨2点执行
                {
                    await ArchiveOldLogsAsync();
                    await CleanupArchivedLogsAsync();
                }
                
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Log maintenance failed");
            }
        }
    }
    
    private async Task ArchiveOldLogsAsync()
    {
        var cutoffDate = DateTime.UtcNow.AddDays(-30);
        await _errorLogArchiveService.ArchiveLogsAsync(cutoffDate);
    }
}
```

## 风险缓解

### 网络故障处理

```csharp
// src/MaterialClient.Urban/Services/NetworkResilienceService.cs
public class NetworkResilienceService
{
    private readonly IDeviceStatusReporter _reporter;
    private readonly IDatabaseService _databaseService;
    
    public async Task ReportWithResilienceAsync(DeviceStatusDto status)
    {
        try
        {
            await _reporter.ReportDeviceStatusAsync(status);
        }
        catch (HttpRequestException ex)
        {
            // 网络故障，存储到本地数据库
            await _databaseService.StorePendingStatusAsync(status);
            _logger.LogWarning("Network unavailable, status stored locally");
        }
    }
    
    public async Task RetryPendingTransmissionsAsync()
    {
        var pendingStatuses = await _databaseService.GetPendingStatusesAsync();
        
        foreach (var status in pendingStatuses)
        {
            try
            {
                await _reporter.ReportDeviceStatusAsync(status);
                await _databaseService.MarkAsTransmittedAsync(status.Id);
            }
            catch
            {
                // 继续尝试下一个
                continue;
            }
        }
    }
}
```

### 数据一致性保证

```csharp
// src/UrbanManagement.Core/Services/DeviceStatusConsistencyService.cs
public class DeviceStatusConsistencyService
{
    public async Task EnsureConsistencyAsync()
    {
        // 修复长时间未更新的客户端状态
        var staleClients = await _deviceStatusRepository.GetListAsync(
            c => c.IsOnline && 
                 c.LastHeartbeat < DateTime.UtcNow.AddMinutes(-15));
        
        foreach (var client in staleClients)
        {
            client.IsOnline = false;
            client.LastOffline = DateTime.UtcNow;
            await _deviceStatusRepository.UpdateAsync(client);
            
            _logger.LogInformation("Marked client {ClientId} as offline due to stale heartbeat", 
                client.ClientId);
        }
    }
}
```

## 总结

### 实施时间表
- **第1-2周**: 数据模型设置和基础服务
- **第3-5周**: WebSocket 通信和错误日志
- **第6-7周**: 监控界面和分析功能
- **第8周**: 测试、优化和部署

### 关键成功因素
- ✅ **渐进式实施**: 分阶段部署，降低风险
- ✅ **向后兼容**: 保持现有系统正常运行
- ✅ **充分测试**: 确保新功能稳定性
- ✅ **监控完善**: 建立完善的监控和告警机制
- ✅ **文档齐全**: 维护清晰的文档和操作指南

### 预期收益
- 🚀 **实时监控**: 实现设备和状态的实时监控
- 📊 **数据驱动**: 基于数据分析进行决策
- 🛡️ **问题预防**: 提前发现和解决潜在问题
- 📈 **运维效率**: 减少运维工作量，提高系统可用性

---

**完成状态**: 调研完成，建议的解决方案已就绪，可以开始实施。