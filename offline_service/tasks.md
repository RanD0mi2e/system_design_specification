# 任务列表：离线服务系统 (Offline Service)

## 任务 1: 扩展 IndexedDB Schema 与基础数据服务

- [ ] 1.1 在 `src/db/index.ts` 中新增 `IEamSyncTask` 和 `IEamNetworkSnapshot` 接口定义
- [ ] 1.2 升级 Dexie schema 版本（v4 → v5），添加 `eam_sync_tasks` 和 `eam_network_snapshots` 表及索引
- [ ] 1.3 创建 `src/db/sync.service.ts`，实现同步任务的 CRUD 操作（add、getByStatus、updateStatus、bulkGet、delete、clearCompleted）
- [ ] 1.4 创建 `src/db/network.service.ts`，实现网络快照的写入、查询和过期清理操作
- [ ] 1.5 验证数据库迁移：确保现有数据在 schema 升级后不丢失

## 任务 2: 实现 NetworkMonitor 网络监控器

- [ ] 2.1 创建 `src/services/NetworkMonitor.ts`，定义 `NetworkStatus`、`NetworkMonitorConfig` 类型
- [ ] 2.2 实现核心探测逻辑 `executeProbe()`：基于现有 `fetchWithTimeout` 发起探测请求，根据响应判断网络质量
- [ ] 2.3 实现自适应探测间隔：根据当前网络状态动态调整 `setTimeout` 间隔（good: 30s, weak: 10s, offline: 5s）
- [ ] 2.4 实现连续失败计数与离线判定：连续 3 次失败标记为 `offline`，成功时重置计数
- [ ] 2.5 实现浏览器 `online`/`offline` 事件监听：`online` 时立即触发探测，`offline` 时立即标记离线
- [ ] 2.6 实现事件发布/订阅机制：`onStatusChange` 注册回调，状态变化时通知所有监听者
- [ ] 2.7 实现 `start()` / `stop()` 生命周期管理：启动/停止探测循环和事件监听

## 任务 3: 实现 OfflineStrategyManager 离线策略管理器

- [ ] 3.1 创建 `src/services/OfflineStrategyManager.ts`，定义 `OfflineMode`、`OperationType`、`StrategyDecision` 类型
- [ ] 3.2 实现策略矩阵：为 online/cautious/offline 三种模式 × read/write/upload/download 四种操作定义策略决策
- [ ] 3.3 实现 `getStrategy(operation)` 方法：根据当前模式和操作类型返回对应策略
- [ ] 3.4 实现模式自动切换：监听 NetworkMonitor 状态变化，自动更新当前模式
- [ ] 3.5 实现 `onModeChange` 事件通知：模式变化时通知所有监听者

## 任务 4: 实现 SyncEngine 同步引擎

- [ ] 4.1 创建 `src/services/SyncEngine.ts`，定义 `SyncTask`、`SyncProgress`、`SyncEvent` 类型
- [ ] 4.2 实现 `enqueue()` 方法：生成任务 ID、设置初始状态、持久化到 IndexedDB、在线模式下自动触发同步
- [ ] 4.3 实现 `startSync()` 方法：从 IndexedDB 读取 pending 任务，按优先级和时间排序，逐个执行同步
- [ ] 4.4 实现单任务同步 `executeTaskSync()`：根据 businessType 和 operation 路由到对应 API，处理成功/失败/冲突
- [ ] 4.5 实现重试机制：失败任务增加 retryCount，未超限保持 pending，超限标记 failed
- [ ] 4.6 实现同步中断：监听网络状态，offline 时立即停止当前批次，保留未处理任务
- [ ] 4.7 实现网络恢复自动同步：监听网络状态从非 good 变为 good 时自动调用 startSync()
- [ ] 4.8 实现同步进度追踪：维护 `SyncProgress` 响应式状态，实时更新 total/completed/failed
- [ ] 4.9 实现事件发布：taskComplete、taskFailed、conflict、allComplete 事件

## 任务 5: 实现 ConflictResolver 冲突解决器

- [ ] 5.1 创建 `src/services/ConflictResolver.ts`，定义 `ConflictStrategy`、`ConflictInfo`、`ConflictResolution` 类型
- [ ] 5.2 实现 `resolve()` 方法：根据策略类型执行对应的冲突解决逻辑
- [ ] 5.3 实现 local-first 策略：直接返回本地数据
- [ ] 5.4 实现 remote-first 策略：返回远程数据并更新本地
- [ ] 5.5 实现 merge 策略：字段级合并（以最新修改时间为准）
- [ ] 5.6 实现 `registerMerger()` 方法：允许业务类型注册自定义合并逻辑

## 任务 6: 创建 Vue Composable 层

- [ ] 6.1 创建 `src/hooks/useNetworkMonitor.ts`：封装 NetworkMonitor 为响应式 composable，管理生命周期（onMounted start / onUnmounted stop）
- [ ] 6.2 创建 `src/hooks/useSyncEngine.ts`：封装 SyncEngine 为响应式 composable，暴露 enqueue、startSync、progress、pendingTasks
- [ ] 6.3 创建 `src/hooks/useOfflineService.ts`：组合 useNetworkMonitor + useSyncEngine，提供 `submitWithOfflineSupport()` 统一入口
- [ ] 6.4 在 `useOfflineService` 中实现 `submitWithOfflineSupport()`：在线时直接提交，失败或离线时降级到本地存储 + 同步队列
- [ ] 6.5 实现网络恢复自动 flush：watch 网络状态变化，恢复时自动触发同步

## 任务 7: 实现 UI 组件

- [ ] 7.1 创建 `src/components/OfflineIndicator.vue`：网络状态通知栏组件（弱网黄色、离线红色，使用 van-notice-bar）
- [ ] 7.2 创建 `src/components/SyncBadge.vue`：待同步任务数量角标组件（使用 van-badge）
- [ ] 7.3 在 Layout 或 App.vue 中集成 OfflineIndicator 和 SyncBadge 组件
- [ ] 7.4 实现同步完成 Toast 通知：同步完成时显示成功/失败数量

## 任务 8: 集成现有业务模块

- [ ] 8.1 重构 `src/db/patrol.service.ts`：将待上传队列迁移到统一的 SyncEngine（保持向后兼容）
- [ ] 8.2 重构 `src/db/work.service.ts`：将待上传队列迁移到统一的 SyncEngine（保持向后兼容）
- [ ] 8.3 重构 `src/hooks/useAttachmentOffline.ts`：使用 useOfflineService 替代直接调用 probeNetworkQuality
- [ ] 8.4 在巡视业务页面中接入 `useOfflineService`，替换原有的离线处理逻辑
- [ ] 8.5 在作业业务页面中接入 `useOfflineService`，替换原有的离线处理逻辑

## 任务 9: 存储空间管理与降级

- [ ] 9.1 实现存储空间监控：定期检查 IndexedDB 使用量（navigator.storage.estimate）
- [ ] 9.2 实现自动清理策略：清理超过 24 小时的网络快照、已完成超过 7 天的同步任务
- [ ] 9.3 实现 localStorage 降级：IndexedDB 不可用时降级到 localStorage 存储同步队列
- [ ] 9.4 实现存储空间不足告警：空间使用超过 80% 时通知用户

## 任务 10: 测试与验证

- [ ] 10.1 编写 NetworkMonitor 单元测试：状态转换、探测间隔、事件触发
- [ ] 10.2 编写 OfflineStrategyManager 单元测试：各模式下策略决策正确性
- [ ] 10.3 编写 SyncEngine 单元测试：任务入队、排序、状态流转、重试逻辑
- [ ] 10.4 编写 ConflictResolver 单元测试：各策略的冲突解决结果
- [ ] 10.5 编写集成测试：模拟网络状态切换的端到端离线→在线同步流程
- [ ] 10.6 在真实设备（企业微信）上进行离线场景手动测试
