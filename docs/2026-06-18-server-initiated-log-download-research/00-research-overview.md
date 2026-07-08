# Server-Initiated Log Download to Online Clients — Research

> **Date**: 2026-06-18
> **Scope**: How to design a mechanism where the UrbanManagement server can proactively instruct online MaterialClient instances to package and upload their log files, with attention to performance and log file classification.
> **Vault**: material-client-vault · 项目源码在 [MaterialMonospec](../../../MaterialMonospec/)

---

## 1. Current State Summary

### 1.1 Existing Communication Infrastructure

| Component | Location | Protocol | Direction |
|-----------|----------|----------|-----------|
| `DeviceStatusHub` | `UrbanManagement.Core/Hubs/` | SignalR (WebSocket) | Client → Server (status upload), Server → Browser (status broadcast) |
| `DeviceStatusSignalRClient` | `MaterialClient.Common/Services/` | SignalR Client | Client → Server only (UploadStatus, VerifyJwtAsync, GetClientProjectLicenseInfo) |
| `IUrbanManagementApi` | `MaterialClient.Urban/Api/` | Refit HTTP | Client → Server (weighing records, attachments) |
| `PollingBackgroundService` | `MaterialClient.Backgrounds/` | HTTP | Client → Server (periodic sync) |

**Key finding**: The existing `DeviceStatusHub` + `DeviceStatusSignalRClient` establishes a persistent bidirectional WebSocket connection but is currently only used for client→server communication. The client already listens for `DeviceStatusUpdate` and `HelloResponse` from the server (see `DeviceStatusSignalRClient.cs` lines 131-136), proving the bidirectional pipe exists but server→client command dispatch is not yet implemented.

### 1.2 Existing Connection Lifecycle Tracking

The server maintains a real-time view of online clients through:

- **`ClientConnectionCacheItem`** — Distributed cache (`ClientConnection:{proId}`) with `IsConnected`, `ConnectedAt`, `DisconnectedAt`
- **`__connection_registry__`** — Cache key holding all known ProIds
- **`_connectionProIdMap`** — In-memory `Dictionary<ConnectionId, ConnectionProIdMapping>` in `DeviceStatusHub`
- **`client_connection` SignalR group** — Broadcasts connect/disconnect events to browser subscribers

This means the server already knows exactly which clients are online and can target commands to specific `ConnectionId`s or ProIds.

### 1.3 Existing Log Infrastructure

| Aspect | Configuration |
|--------|---------------|
| Framework | Serilog with `Serilog.Sinks.File` |
| Log directory | `{AppContext.BaseDirectory}/Logs/` |
| File pattern | `MaterialClient-.log` (daily rolling) / `MaterialClient.Urban-.log` |
| Rolling interval | Daily (`RollingInterval.Day`) |
| Retention | 30 files (`retainedFileCountLimit: 30`) |
| Output template | `{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}` |
| Encoding | UTF-8 |
| Classification | Currently **single sink** — all log levels go to one file per app |

**Key finding**: There is no log file classification today. All logs (application, device, communication, error) go into a single rolling file. Classification must be designed from scratch.

---

## 2. Design Proposal: Server-Initiated Log Download

### 2.1 Architecture Overview

```
UrbanManagement Server                          MaterialClient.Urban
┌───────────────────────┐                     ┌──────────────────────────┐
│ Admin/Debug UI         │                     │ LogCollectionService     │
│  ├─ Select client(s)   │                     │  ├─ Enumerate log files  │
│  ├─ Choose categories  │                     │  ├─ Classify by category │
│  └─ Trigger command    │                     │  ├─ Compress (ZIP)       │
│        │               │                     │  └─ Upload via HTTP      │
│        ▼               │                     │         │                │
│ DeviceStatusHub        │──SignalR WS ───────►│ LogCommandHandler       │
│  └─ SendCommand()     │   "CollectLogs"     │  └─ OnCollectLogs()      │
│                        │                     │         │                │
│ Log Collection API     │◄── HTTP POST ──────│ UploadLogPackage()      │
│  └─ Receive ZIP        │                     │                          │
└───────────────────────┘                     └──────────────────────────┘
```

**Design principle**: Use SignalR for command dispatch (server→client), HTTP for file upload (client→server). This separates the thin control plane (SignalR) from the heavy data plane (HTTP), avoiding WebSocket message size limits and keeping the persistent connection responsive.

### 2.2 Protocol Design

#### Step 1: Server sends command via SignalR

```csharp
// Server → Client: Log collection command
public record CollectLogsCommand
{
    public string CommandId { get; init; } = "";        // Unique ID for tracking
    public string RequestedCategories { get; init; } = ""; // Comma-separated or "*"
    public DateTime? Since { get; init; }                 // Optional: only logs after this time
    public DateTime? Until { get; init; }                 // Optional: only logs before this time
    public int? MaxFileSizeMb { get; init; }              // Optional: size cap for response
    public int TimeoutSeconds { get; init; } = 120;       // Client-side timeout
}
```

#### Step 2: Client acknowledges and begins processing

```csharp
// Client → Server: Acknowledgment
public record CollectLogsAck
{
    public string CommandId { get; init; } = "";
    public CollectLogsStatus Status { get; init; }        // Accepted / Rejected / Busy
    public string? RejectReason { get; init; }
}

public enum CollectLogsStatus
{
    Accepted,
    Rejected,       // e.g. another collection in progress
    Busy            // client busy, try later
}
```

#### Step 3: Client uploads log package via HTTP

```csharp
// Client → Server: HTTP POST multipart/form-data
// POST /api/log-collection/upload
// Content-Type: multipart/form-data
// Fields:
//   - commandId: string
//   - proId: string
//   - categories: string (comma-separated)
//   - file: IFormFile (ZIP archive)
```

#### Step 4: Server confirms receipt

```csharp
// Server → Client (via SignalR): Upload confirmation
// SignalR event "LogCollectionResult"
public record LogCollectionResult
{
    public string CommandId { get; init; } = "";
    public bool Success { get; init; }
    public string? ErrorMessage { get; init; }
    public long FileSizeBytes { get; init; }
    public int FileCount { get; init; }
}
```

### 2.3 Log File Classification Design

Currently, all logs go to a single file. Classification can be achieved through **multiple Serilog sinks** without changing logging call sites:

```csharp
// MaterialClientUrbanModule.cs — Enhanced Serilog configuration
private void ConfigureSerilog(IServiceCollection services, IConfiguration configuration)
{
    var logsDirectory = Path.Combine(AppContext.BaseDirectory, "Logs");
    Directory.CreateDirectory(logsDirectory);

    var loggerConfig = new LoggerConfiguration()
        .Enrich.FromLogContext()
        .MinimumLevel.Is(LogEventLevel.Information)
        .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
        .MinimumLevel.Override("Volo.Abp", LogEventLevel.Warning)
        .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning);

    // Category-based log sinks
    loggerConfig
        // Main application log (everything)
        .WriteTo.File(
            Path.Combine(logsDirectory, "app-.log"),
            rollingInterval: RollingInterval.Day,
            retainedFileCountLimit: 30,
            outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {SourceContext} {Message:lj}{NewLine}{Exception}")

        // Device hardware log (Scale, Camera, LPR, Sound, Printer)
        .WriteTo.Logger(lc => lc
            .Filter.ByIncludingOnly(MatchesSourceContext("MaterialClient.*.Device*")
                .Or(MatchesSourceContext("MaterialClient.*.Lpr*"))
                .Or(MatchesSourceContext("MaterialClient.*.Camera*"))
                .Or(MatchesSourceContext("MaterialClient.*.Scale*"))
                .Or(MatchesSourceContext("MaterialClient.*.Sound*")))
            .WriteTo.File(
                Path.Combine(logsDirectory, "device-.log"),
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 14,
                outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"))

        // Communication/network log (SignalR, HTTP, upload)
        .WriteTo.Logger(lc => lc
            .Filter.ByIncludingOnly(MatchesSourceContext("MaterialClient.*SignalR*")
                .Or(MatchesSourceContext("MaterialClient.*Upload*"))
                .Or(MatchesSourceContext("MaterialClient.*Polling*")))
            .WriteTo.File(
                Path.Combine(logsDirectory, "comm-.log"),
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 14))

        // Error-only log (all Error/Critical from any source)
        .WriteTo.Logger(lc => lc
            .Filter.ByIncludingOnly(e => e.Level >= LogEventLevel.Error)
            .WriteTo.File(
                Path.Combine(logsDirectory, "error-.log"),
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 60)); // Keep errors longer

    Log.Logger = loggerConfig.CreateLogger();
    services.AddLogging(logging =>
    {
        logging.ClearProviders();
        logging.AddSerilog(Log.Logger);
    });
}
```

#### Recommended Log Categories

| Category | File Pattern | Retention | Contents |
|----------|-------------|-----------|----------|
| **Application** (`app`) | `app-.log` | 30 days | All log events (full copy) |
| **Device** (`device`) | `device-.log` | 14 days | Hardware device interactions (Scale, Camera, LPR, Sound, Printer) |
| **Communication** (`comm`) | `comm-.log` | 14 days | SignalR, HTTP upload, polling events |
| **Error** (`error`) | `error-.log` | 60 days | Error/Critical level from any source |

When the server requests specific categories, the client only packages the matching files.

---

## 3. Performance Considerations

### 3.1 Command Dispatch (SignalR)

| Concern | Mitigation |
|---------|------------|
| **WebSocket blocking** | Command dispatch is a single small JSON message (~200 bytes). No impact on existing `UploadStatus` traffic. |
| **Rate limiting** | Extend existing `IsRateLimited()` in `DeviceStatusHub` with a separate counter for admin commands (e.g., max 1 command per 10 seconds per client). |
| **Concurrency** | Client tracks one active collection at a time via `SemaphoreSlim(1,1)`. Reject concurrent commands with `CollectLogsStatus.Busy`. |
| **Timeout** | Server tracks command state in distributed cache with TTL. If no upload received within `TimeoutSeconds`, mark command as expired. |

### 3.2 Log Packaging (Client-Side)

| Concern | Mitigation |
|---------|------------|
| **Large log files** | Daily rolling + 30-day retention = max ~30 files per category. Typical size: 5-50MB total. ZIP compression reduces by ~90%. |
| **Disk I/O during packaging** | Use `FileShare.Read` to avoid locking active log files. Compress in a temp directory, delete after upload. |
| **Memory usage** | Stream-based ZIP creation (e.g., `System.IO.Compression.ZipArchive` with `CreateEntryFromFile`). Never load entire ZIP into memory. |
| **CPU pressure** | ZIP compression runs at Normal priority with `CancellationToken`. If client is actively weighing, defer compression to idle time. |

### 3.3 File Upload (HTTP)

| Concern | Mitigation |
|---------|------------|
| **Upload size** | Typical compressed package: 1-10MB. Use `MaxFileSizeMb` parameter to cap at server's comfort level (default 50MB). |
| **Upload timeout** | Use `HttpClient.Timeout` matching `TimeoutSeconds` from the command. |
| **Retry** | Single retry on transient HTTP failure (5xx, network error). No infinite retries. |
| **Channel separation** | Upload uses dedicated `HttpClient` instance, independent from the Refit `IUrbanManagementApi` used for weighing records. |
| **Progress reporting** | Report upload progress via SignalR event (`LogCollectionProgress`) for admin visibility. |

### 3.4 Server-Side Storage

| Concern | Mitigation |
|---------|------------|
| **Disk space** | Store uploaded ZIPs under `{StorageRoot}/log-collections/{proId}/{commandId}.zip`. Auto-cleanup after 7 days via background worker. |
| **Concurrent uploads** | Multiple clients can upload simultaneously — no locking needed since each file has a unique `commandId`. |
| **Database tracking** | Optional `LogCollectionRecord` entity for audit trail (commandId, proId, timestamp, fileCount, fileSize, status). |

---

## 4. Implementation Design

### 4.1 Server-Side Changes (UrbanManagement)

#### 4.1.1 New Hub Methods in `DeviceStatusHub`

Add to existing `DeviceStatusHub.cs`:

```csharp
/// <summary>
///     Sends a log collection command to a specific online client.
///     Called by the admin UI or API.
/// </summary>
public async Task<CollectLogsCommandResult> CollectLogsFromClient(string proId, CollectLogsCommand command)
{
    var connectionId = GetConnectionIdByProId(proId);
    if (connectionId == null)
    {
        return CollectLogsCommandResult.ClientOffline(proId);
    }

    command.CommandId = Guid.NewGuid().ToString("N")[..8];
    await CacheCommandStateAsync(command); // Track in distributed cache

    await Clients.Client(connectionId).SendAsync("CollectLogs", command);
    return CollectLogsCommandResult.Accepted(command.CommandId, proId);
}
```

#### 4.1.2 New HTTP Endpoint for ZIP Upload

```csharp
// src/UrbanManagement.Core/Services/LogCollectionService.cs
public interface ILogCollectionService : ITransientDependency
{
    Task<LogCollectionReceipt> ReceiveLogPackageAsync(
        string commandId, string proId, string categories, Stream fileStream, string fileName);
    Task CleanupExpiredCollectionsAsync();
}
```

```csharp
// Controller for log upload
[ApiController]
[Route("api/log-collection")]
public class LogCollectionController : AbpController
{
    [HttpPost("upload")]
    [RequestSizeLimit(50 * 1024 * 1024)] // 50MB
    public async Task<IActionResult> Upload(
        [FromForm] string commandId,
        [FromForm] string proId,
        [FromForm] string? categories,
        IFormFile file)
    {
        // Validate commandId exists in cache and is not expired
        // Save file to storage
        // Update command state
        // Notify via SignalR to admin subscribers
    }
}
```

### 4.2 Client-Side Changes (MaterialClient.Urban / MaterialClient.Common)

#### 4.2.1 SignalR Command Handler Registration

In `DeviceStatusSignalRClient.StartAsync()`, register a new handler:

```csharp
// Subscribe to server-initiated log collection command
_connection.On<CollectLogsCommand>("CollectLogs", async command =>
{
    _logger.LogInformation(
        "DeviceStatusSignalRClient: Received CollectLogs command. CommandId={CommandId}, Categories={Categories}",
        command.CommandId, command.RequestedCategories);

    await _localEventBus.PublishAsync(new CollectLogsCommandReceivedEto(command));
});
```

#### 4.2.2 Log Collection Service

```csharp
// src/MaterialClient.Common/Services/LogCollectionService.cs
public interface ILogCollectionService : ISingletonDependency
{
    /// <summary>
    ///     Enumerates and packages log files matching the requested categories.
    ///     Returns a stream containing a ZIP archive.
    /// </summary>
    Task<Stream> PackageLogsAsync(CollectLogsCommand command, CancellationToken ct);

    /// <summary>
    ///     Uploads the log package to the server via HTTP.
    /// </summary>
    Task<LogUploadResult> UploadPackageAsync(string commandId, Stream packageStream, CancellationToken ct);

    /// <summary>
    ///     Gets the list of available log categories and file counts.
    /// </summary>
    Task<LogInventory> GetLogInventoryAsync();
}

[AutoConstructor]
public partial class LogCollectionService : ILogCollectionService, IAsyncDisposable
{
    private readonly ILogger<LogCollectionService> _logger;
    private readonly ILicenseService _licenseService;
    private readonly HttpClient _uploadHttpClient;
    private readonly SemaphoreSlim _collectionLock = new(1, 1);
    private readonly IConfiguration _configuration;

    // Known log category -> file prefix mapping
    private static readonly Dictionary<string, string[]> CategoryFilePatterns = new()
    {
        ["app"]    = ["app-"],
        ["device"] = ["device-"],
        ["comm"]   = ["comm-"],
        ["error"]  = ["error-"],
    };

    public async Task<Stream> PackageLogsAsync(CollectLogsCommand command, CancellationToken ct)
    {
        var logsDirectory = Path.Combine(AppContext.BaseDirectory, "Logs");
        var tempZipPath = Path.Combine(Path.GetTempPath(), $"log-collection-{command.CommandId}.zip");

        try
        {
            using var zipStream = new FileStream(tempZipPath, FileMode.Create);
            using var archive = new ZipArchive(zipStream, ZipArchiveMode.Create);

            var filesToInclude = EnumerateMatchingLogFiles(
                logsDirectory, command.RequestedCategories, command.Since, command.Until);

            foreach (var file in filesToInclude)
            {
                var entryName = Path.GetRelativePath(AppContext.BaseDirectory, file);
                archive.CreateEntryFromFile(file, entryName, CompressionLevel.Optimal);
            }

            zipStream.Position = 0;

            // Return a copy stream (zipStream will be disposed, we need the file)
            var resultStream = new FileStream(tempZipPath, FileMode.Open, FileAccess.Read);
            return resultStream;
        }
        catch
        {
            File.Delete(tempZipPath);
            throw;
        }
    }
}
```

#### 4.2.3 Event-Driven Handler (ABP LocalEventBus)

```csharp
// src/MaterialClient.Common/Events/CollectLogsCommandReceivedEto.cs
public class CollectLogsCommandReceivedEto(string Command)
{
    public CollectLogsCommand Command { get; init; } = Command;
}

// src/MaterialClient.Urban/Handlers/CollectLogsCommandHandler.cs
[AutoConstructor]
public class CollectLogsCommandHandler : ILocalEventHandler<CollectLogsCommandReceivedEto>,
    ITransientDependency
{
    private readonly ILogCollectionService _logCollectionService;
    private readonly IDeviceStatusSignalRClient _signalRClient;
    private readonly ILogger<CollectLogsCommandHandler> _logger;

    public async Task HandleEventAsync(CollectLogsCommandReceivedEto eventData)
    {
        var command = eventData.Command;

        // 1. Send ACK via SignalR (InvokeAsync to get server confirmation)
        try
        {
            await _signalRClient.SendLogCollectionAckAsync(new CollectLogsAck
            {
                CommandId = command.CommandId,
                Status = CollectLogsStatus.Accepted
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send CollectLogs ACK for CommandId={CommandId}", command.CommandId);
            return;
        }

        // 2. Package logs
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(command.TimeoutSeconds));
        Stream? packageStream = null;
        try
        {
            packageStream = await _logCollectionService.PackageLogsAsync(command, cts.Token);
        }
        catch (OperationCanceledException)
        {
            _logger.LogWarning("Log packaging timed out for CommandId={CommandId}", command.CommandId);
            return;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to package logs for CommandId={CommandId}", command.CommandId);
            return;
        }

        // 3. Upload via HTTP
        await using (packageStream)
        {
            var result = await _logCollectionService.UploadPackageAsync(command.CommandId, packageStream, cts.Token);
            if (!result.Success)
            {
                _logger.LogError("Failed to upload log package for CommandId={CommandId}: {Error}",
                    command.CommandId, result.ErrorMessage);
            }
        }
    }
}
```

---

## 5. Sequence Diagram

```
Admin          UrbanManagement                  MaterialClient.Urban
  │                    │                              │
  │  POST /api/admin/collect-logs                    │
  │  { proId, categories: "error,device" }          │
  │──────────────────►│                              │
  │                    │  Check client online         │
  │                    │  (ClientConnectionCache)      │
  │                    │                              │
  │                    │  SignalR: "CollectLogs"       │
  │                    │─────────────────────────────►│
  │                    │                              │  ACK: Accepted
  │                    │◄─────────────────────────────│
  │  { commandId }    │                              │
  │◄──────────────────│                              │
  │                    │                              │  Package logs (ZIP)
  │                    │                              │  [async, non-blocking]
  │                    │                              │
  │                    │  POST /api/log-collection/upload
  │                    │◄─────────────────────────────│
  │                    │                              │  multipart ZIP file
  │                    │─────────────────────────────►│
  │                    │  Save ZIP to storage          │
  │                    │  SignalR: "LogCollectionResult"
  │                    │─────────────────────────────►│
  │                    │                              │
  │  (optional) Poll for result                     │
  │──────────────────►│                              │
  │  { status, fileSize, fileCount }                │
  │◄──────────────────│                              │
```

---

## 6. Evidence Trace

### Files Inspected

| File | Relevance |
|------|-----------|
| `MaterialClient.Common/Services/DeviceStatusSignalRClient.cs` | Existing SignalR client with bidirectional capability, auto-reconnect, message queue |
| `MaterialClient.Common/Configuration/SignalRClientOptions.cs` | Connection configuration (URL, reconnect delays, queue size) |
| `MaterialClient.Common/Models/DeviceStatusMessage.cs` | Existing SignalR message model pattern |
| `MaterialClient/MaterialClientModule.cs` (lines 106-146) | Serilog configuration — single sink, daily rolling, 30-day retention |
| `UrbanManagement.Core/Hubs/DeviceStatusHub.cs` | Hub with `UploadStatus`, connection lifecycle, rate limiting, ProId tracking |
| `UrbanManagement.Core/Services/DeviceStatusService.cs` | Connection caching, client registry, device offline marking |
| `UrbanManagement.Core/Services/FileService.cs` | File storage patterns (Base64 images, disk storage, compression) |
| `UrbanManagement.Core/Models/ClientConnectionCacheItem.cs` | Online client tracking data model |
| `UrbanManagement.Core/Models/ClientConnectionDto.cs` | Client connection status DTO |
| `MaterialClient.Common/Events/` | Event-driven patterns (Eto naming, MessageBus, LocalEventBus) |
| `docs/2026-06-01-device-sync-status-logging-solutions/03-错误日志方案.md` | Prior error log design — SignalR push for real-time error events (not file download) |

### Key Architecture Decisions

1. **SignalR for command, HTTP for data**: SignalR WebSocket frame size is typically limited (32KB default in some transports). Log ZIP packages are 1-50MB. Using HTTP multipart upload avoids frame limits and keeps the persistent connection responsive.

2. **Event-driven handler pattern**: Following the project's established ABP `ILocalEventHandler<T>` + `[AutoConstructor]` pattern ensures clean separation from the SignalR client and consistent DI registration.

3. **Category-based multi-sink Serilog**: Enables targeted log collection without modifying any existing `ILogger` call sites. The admin can request only "error" logs (typically tiny) or "all" logs for full diagnostics.

4. **Fault isolation**: Following the project's Monitor isolation principle (AGENTS.md), the log collection service must never block UI or weighing. All operations run on background threads with cancellation support.

---

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Active log file locking** | ZIP creation may fail if Serilog holds exclusive lock | Use `FileShare.ReadWrite` or copy files before compression. Serilog's File sink uses `FileShare.Read` by default. |
| **Large log accumulation** | 30 days of app logs at verbose level could exceed 500MB | Retention policies per category; compress before upload; `MaxFileSizeMb` server-side cap. |
| **Client offline when command sent** | Command lost | Server checks `ClientConnectionCacheItem.IsConnected` before dispatch. If offline, store command in pending queue, deliver on reconnect. |
| **Simultaneous commands** | Resource contention | Single `SemaphoreSlim(1,1)` per client. Reject with `Busy` status. |
| **Upload failure** | Partial state | Track command lifecycle in distributed cache with TTL. Client retries once. Admin can re-trigger. |
| **Network instability** | Timeout during large upload | Configurable timeout. Client uses `HttpClient.Timeout`. Server uses `[RequestSizeLimit]`. |
| **Security** | Unauthorized log collection | Require `[Authorize]` on both the admin command endpoint and the upload endpoint. Validate `commandId` matches a server-issued command. |

---

## 8. Recommended Follow-Up Actions

1. **Implement Serilog multi-sink classification** — Add category-based log files to `MaterialClientUrbanModule.cs` (and optionally `MaterialClientModule.cs`). This is a prerequisite for selective log collection.

2. **Extend `DeviceStatusHub` with `CollectLogs` method** — Add the server→client command dispatch. This leverages the existing bidirectional connection.

3. **Implement `ILogCollectionService` in `MaterialClient.Common`** — Shared implementation usable by both MaterialClient and MaterialClient.Urban.

4. **Add HTTP upload endpoint in UrbanManagement** — `POST /api/log-collection/upload` with `[RequestSizeLimit]` and command validation.

5. **Add admin UI panel** — "Log Collection" tab showing online clients, category selection, and download history.

6. **Add background cleanup worker** — Auto-delete uploaded ZIP files older than 7 days to prevent disk bloat.

7. **Consider future enhancements**:
   - Log streaming (real-time tail) via SignalR for live debugging
   - Differential collection (only new logs since last collection)
   - Automatic collection trigger on error threshold exceeded
   - Batch collection (send command to all online clients at once)
