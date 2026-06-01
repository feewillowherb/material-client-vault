# MaterialClient.Urban 设备同步与状态监控解决方案调研

## 调研范围

本次调研针对 MaterialClient.Urban 与 UrbanManagement 之间的实时数据同步、设备状态监控和错误日志提交提供完整解决方案。

### 核心问题
1. **设备信息实时同步**：如何将 MaterialClient.Urban 中的设备信息实时同步到 UrbanManagement
2. **在线状态监控**：客户端和摄像头在线情况的实时监控方案
3. **错误日志提交**：Urban 需要把错误日志提交到服务器的方案

## 研究目标

- 分析现有架构和通信模式
- 设计实时数据同步方案
- 提供设备状态监控机制
- 设计错误日志收集和提交流程
- 提供具体实施建议和代码示例

## 文档结构

1. **调研总览** (本文档) - 调研背景、目标和现有架构分析
2. **设备实时同步方案** - MaterialClient.Urban 到 UrbanManagement 的数据同步
3. **状态监控方案** - 客户端和摄像头在线状态监控
4. **错误日志方案** - 错误日志收集和提交机制
5. **实施建议** - 具体实施步骤和注意事项

## 相关项目

- **MaterialClient.Urban**: 城管称重客户端，负责本地称重记录管理和数据上传
- **UrbanManagement**: ABP 框架的服务端应用，接收和处理城管称重数据
- **MaterialClient.Common**: 共享组件库，包含实体、服务和技术组件

## 现有架构分析

### MaterialClient.Urban 端

**核心组件**:
- `UrbanServerUploadService`: 负责将称重记录提交到 UrbanManagement 服务端
- `IUrbanManagementApi`: 基于 Refit 的 RESTful API 接口
- `UrbanWeighingExtensionService`: 管理称重记录的同步状态和扩展信息

**数据模型**:
- `UrbanWeighingExtension`: 存储同步状态、重试次数、异常标志等
- `WeighingRecord`: 本地称重记录主表
- `SyncStatus` 枚举: Pending (待上传)、Synced (已同步)、Failed (上传失败)

**通信机制**:
- 基于 Refit 的 HTTP 客户端
- POST `/api/urban/weighing-records` 提交称重记录
- JSON 格式数据传输
- 使用 `ClientRecordId` 进行幂等性控制

### UrbanManagement 端

**核心组件**:
- `UrbanWeighingRecordController`: 接收称重记录的 API 控制器
- `UrbanWeighingRecordAppService`: 处理称重记录的业务逻辑
- `FileService`: 处理文件上传和管理

**数据模型**:
- `UrbanWeighingRecord`: 服务端称重记录表
- `UrbanWeighingRecordAttachment`: 称重记录附件关联表
- `AttachmentFile`: 文件存储表

**数据库设计**:
- SQLite 存储
- `ClientRecordId` 唯一索引用于去重
- 支持附件关联存储

### 当前同步流程

1. **称重记录创建**: MaterialClient.Urban 创建称重记录时生成 `UrbanWeighingExtension`
2. **后台上传**: 后台服务查询待上传记录并调用 API 提交
3. **状态更新**: 根据上传结果更新 `SyncStatus`、`RetryCount`、`LastErrorTime`
4. **服务端处理**: UrbanManagement 接收数据，通过 `ClientRecordId` 去重，存储到数据库

## 关键发现

### 优势
- ✅ 已有基础的同步框架和 API 通信机制
- ✅ 支持同步状态跟踪和错误处理
- ✅ 服务端有完善的数据模型和去重机制
- ✅ 支持 ABP 框架，便于扩展新功能

### 限制和挑战
- ⚠️ 当前仅支持称重记录的批量同步，缺乏实时性
- ⚠️ 没有设备状态监控的专用机制
- ⚠️ 错误日志仅在本地存储，未集中到服务端
- ⚠️ 缺乏心跳和在线状态检测机制
- ⚠️ 服务端缺少设备管理界面

### 技术债务
- 需要建立实时通信机制（WebSocket 或长轮询）
- 需要设计设备状态数据模型
- 需要实现错误日志的远程传输和存储
- 需要考虑服务端的高可用性和负载均衡

## 下一步

本文档作为总览，具体的技术方案和实施细节请参考系列文档：

- [设备实时同步方案](./02-device-real-time-sync-solution.md)
- [状态监控方案](./03-status-monitoring-solution.md)  
- [错误日志方案](./04-error-logging-solution.md)
- [实施建议](./05-implementation-recommendations.md)

---

**调研时间**: 2026-06-01
**目标系统**: MaterialClient.Urban v1.0 + UrbanManagement v1.0
**技术栈**: .NET 8.0, ABP Framework, SQLite, Refit