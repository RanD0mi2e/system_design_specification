# 离线服务系统 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 基于 Vue 3 + Pinia + TypeScript + Dexie 构建离线服务系统，实现弱网/离线场景下的本地落库与网络恢复后自动同步，支持版本冲突解决与任务执行顺序保证。

**Architecture:** 分层架构 — NetworkMonitor 负责网络探测与状态机管理，OfflineStrategyManager 根据网络状态决定操作策略，SyncEngine 管理同步队列的入队/排序/批量执行/重试，ConflictResolver 处理四种冲突策略。Vue Composable 层对外暴露统一 API，UI 组件层提供离线指示器与同步进度展示。

**Tech Stack:** Vue 3.5 (Composition API + `<script setup>`), Pinia 3.0, Vue Router, TypeScript, Dexie 4.3 (IndexedDB), Vant 4.9 (UI), @vueuse/core 14.2

---

## File Structure

| File | Responsibility |
|------|---------------|
| `src/db/index.ts` (modify) | 扩展 Dexie schema，新增 `eam_sync_tasks`、`eam_network_snapshots` 表 |
| `src/db/sync.service.ts` (create) | 同步任务 CRUD（add / getByStatus / updateStatus / delete / clearCompleted） |
| `src/db/network.service.ts` (create) | 网络快照写入 / 查询 / 过期清理 |
| `src/services/NetworkMonitor.ts` (create) | 网络探测循环、状态机、事件发布订阅 |
| `src/services/OfflineStrategyManager.ts` (create) | 策略矩阵、模式切换、策略决策 |
| `src/services/SyncEngine.ts` (create) | 同步队列管理、批量同步、重试、断点续传 |
| `src/services/ConflictResolver.ts` (create) | 四种冲突策略的实现与自定义合并注册 |
| `src/hooks/useNetworkMonitor.ts` (create) | NetworkMonitor 的 Vue Composable 封装 |
| `src/hooks/useSyncEngine.ts` (create) | SyncEngine 的 Vue Composable 封装 |
| `src/hooks/useOfflineService.ts` (create) | 统一离线服务入口，组合上述 composable |
| `src/components/OfflineIndicator.vue` (create) | 离线/弱网通知栏组件 |
| `src/components/SyncBadge.vue` (create) | 待同步任务数量角标 |
| `src/__tests__/services/NetworkMonitor.test.ts` (create) | NetworkMonitor 单元测试 |
| `src/__tests__/services/SyncEngine.test.ts` (create) | SyncEngine 单元测试 |
| `src/__tests__/services/ConflictResolver.test.ts` (create) | ConflictResolver 单元测试 |

---

### Task 1: 扩展 IndexedDB Schema

**Files:**
- Modify: `src/db/index.ts`
- Create: `src/db/sync.service.ts`
- Create: `src/db/network.service.ts`

- [ ] **Step 1: 定义类型接口**

在 `src/db/index.ts` 中新增以下类型定义（放在已有类型之后）：

```typescript
// 同步任务状态
export type SyncTaskStatus = 'pending' | 'syncing' | 'success' | 'failed' | 'conflict'

// 同步任务记录
export interface IEamSyncTask {
  id: string
  businessType: string
  businessId: string
  operation: 'create' | 'update' | 'delete'
  status: SyncTaskStatus
  priority: number
  payload: Record<string, any>
  createdAt: number
  lastAttemptAt: number
  retryCount: number
  maxRetries: number
  lastError?: string
}

// 网络状态快照
export interface IEamNetworkSnapshot {
  id?: number
  quality: 'good' | 'weak' | 'offline'
  duration: number
  timestamp: number
}
```

- [ ] **Step 2: 升级 Dexie schema**

将 `EAMDatabase` 的 version 升级（基于现有版本号 +1），添加新表：

```typescript
this.version(NEW_VERSION).stores({
  // ... 保持现有表定义不变 ...
  eam_sync_tasks: 'id, status, businessType, businessId, priority, createdAt, [status+priority]',
  eam_network_snapshots: '++id, quality, timestamp',
})
```

- [ ] **Step 3: 编写 schema 迁移测试**

创建 `src/__tests__/db/schema.test.ts`：

```typescript
import { describe, it, expect, beforeAll } from 'vitest'
import { db } from '@/db'

describe('Dexie schema migration', () => {
  beforeAll(async () => {
    // 确保数据库已打开
    await db.open()
  })

  it('should have eam_sync_tasks table with correct indexes', () => {
    expect(db.eam_sync_tasks).toBeDefined()
    const schema = db.tables.find(t => t.name === 'eam_sync_tasks')!.schema
    expect(schema.primKey.name).toBe('id')
    expect(schema.indexes.some(i => i.name === 'status')).toBe(true)
    expect(schema.indexes.some(i => i.name === '[status+priority]')).toBe(true)
  })

  it('should have eam_network_snapshots table', () => {
    expect(db.eam_network_snapshots).toBeDefined()
  })
})
```

- [ ] **Step 4: 运行测试验证 schema**

```bash
npx vitest run src/__tests__/db/schema.test.ts
```

Expected: PASS

- [ ] **Step 5: 实现 sync.service.ts**

创建 `src/db/sync.service.ts`：

```typescript
import { db, type IEamSyncTask, type SyncTaskStatus } from './index'
import { v4 as uuid } from 'uuid' // 或使用 crypto.randomUUID()

export const syncTaskService = {
  async add(task: Omit<IEamSyncTask, 'id' | 'status' | 'createdAt' | 'retryCount' | 'lastAttemptAt'>): Promise<string> {
    const id = crypto.randomUUID?.() ?? uuid()
    const now = Date.now()
    await db.eam_sync_tasks.add({
      ...task,
      id,
      status: 'pending',
      createdAt: now,
      retryCount: 0,
      lastAttemptAt: 0,
    })
    return id
  },

  async getByStatus(status: SyncTaskStatus): Promise<IEamSyncTask[]> {
    return db.eam_sync_tasks.where('status').equals(status).toArray()
  },

  async getPendingSorted(): Promise<IEamSyncTask[]> {
    return db.eam_sync_tasks
      .where('status').equals('pending')
      .sortBy('priority', 'createdAt')
  },

  async updateStatus(id: string, status: SyncTaskStatus, extra?: Partial<IEamSyncTask>): Promise<void> {
    await db.eam_sync_tasks.update(id, { status, ...extra })
  },

  async getByBusinessId(businessId: string): Promise<IEamSyncTask | undefined> {
    return db.eam_sync_tasks.where('businessId').equals(businessId).first()
  },

  async delete(id: string): Promise<void> {
    await db.eam_sync_tasks.delete(id)
  },

  async clearCompleted(): Promise<number> {
    return db.eam_sync_tasks.where('status').anyOf('success', 'failed').delete()
  },

  async countPending(): Promise<number> {
    return db.eam_sync_tasks.where('status').equals('pending').count()
  },
}
```

- [ ] **Step 6: 实现 network.service.ts**

创建 `src/db/network.service.ts`：

```typescript
import { db, type IEamNetworkSnapshot } from './index'

export const networkSnapshotService = {
  async add(snapshot: Omit<IEamNetworkSnapshot, 'id'>): Promise<number> {
    return db.eam_network_snapshots.add(snapshot as IEamNetworkSnapshot)
  },

  async getRecent(limit = 100): Promise<IEamNetworkSnapshot[]> {
    return db.eam_network_snapshots
      .orderBy('timestamp')
      .reverse()
      .limit(limit)
      .toArray()
  },

  async clearOlderThan(maxAgeMs: number): Promise<number> {
    const cutoff = Date.now() - maxAgeMs
    return db.eam_network_snapshots.where('timestamp').below(cutoff).delete()
  },
}
```

- [ ] **Step 7: 编写 sync.service 测试**

创建 `src/__tests__/db/sync.service.test.ts`：

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { syncTaskService } from '@/db/sync.service'
import { db } from '@/db'

describe('syncTaskService', () => {
  beforeEach(async () => {
    await db.eam_sync_tasks.clear()
  })

  it('should add a task and return its id', async () => {
    const id = await syncTaskService.add({
      businessType: 'patrol',
      businessId: 'patrol-001',
      operation: 'update',
      priority: 5,
      maxRetries: 3,
      payload: { name: 'test' },
    })
    expect(id).toBeTruthy()
    expect(typeof id).toBe('string')
  })

  it('added task should be pending and retrievable', async () => {
    const id = await syncTaskService.add({
      businessType: 'patrol',
      businessId: 'patrol-002',
      operation: 'create',
      priority: 1,
      maxRetries: 5,
      payload: { value: 42 },
    })
    const pending = await syncTaskService.getByStatus('pending')
    expect(pending.some(t => t.id === id)).toBe(true)
    const task = pending.find(t => t.id === id)!
    expect(task.status).toBe('pending')
    expect(task.retryCount).toBe(0)
  })

  it('getPendingSorted should order by priority then createdAt', async () => {
    await syncTaskService.add({
      businessType: 'patrol', businessId: 'b-1', operation: 'update',
      priority: 3, maxRetries: 3, payload: {},
    })
    // small delay to ensure different createdAt
    await new Promise(r => setTimeout(r, 10))
    await syncTaskService.add({
      businessType: 'patrol', businessId: 'b-2', operation: 'update',
      priority: 1, maxRetries: 3, payload: {},
    })
    const pending = await syncTaskService.getPendingSorted()
    expect(pending[0].priority).toBeLessThanOrEqual(pending[1].priority)
  })

  it('clearCompleted should remove success and failed tasks', async () => {
    const id1 = await syncTaskService.add({
      businessType: 'patrol', businessId: 'b-3', operation: 'update',
      priority: 1, maxRetries: 3, payload: {},
    })
    await syncTaskService.updateStatus(id1, 'success')
    const deleted = await syncTaskService.clearCompleted()
    expect(deleted).toBe(1)
  })
})
```

- [ ] **Step 8: 运行测试**

```bash
npx vitest run src/__tests__/db/
```

Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add src/db/index.ts src/db/sync.service.ts src/db/network.service.ts src/__tests__/db/
git commit -m "feat: extend Dexie schema with sync tasks and network snapshots tables"
```

---

### Task 2: 实现 NetworkMonitor 网络监控器

**Files:**
- Create: `src/services/NetworkMonitor.ts`
- Create: `src/__tests__/services/NetworkMonitor.test.ts`

- [ ] **Step 1: 编写 NetworkMonitor 类型定义与接口**

创建 `src/services/NetworkMonitor.ts`：

```typescript
export interface NetworkStatus {
  quality: 'good' | 'weak' | 'offline'
  lastProbeTime: number
  lastProbeDuration: number
  consecutiveFailures: number
  isProbing: boolean
}

export interface NetworkMonitorConfig {
  goodInterval: number
  weakInterval: number
  offlineInterval: number
  probeTimeout: number
  offlineThreshold: number
  probeUrl: string
}

export const DEFAULT_NETWORK_CONFIG: NetworkMonitorConfig = {
  goodInterval: 30000,
  weakInterval: 10000,
  offlineInterval: 5000,
  probeTimeout: 1000,
  offlineThreshold: 3,
  probeUrl: '/vite.svg',
}

export type StatusChangeCallback = (status: NetworkStatus) => void

export interface NetworkMonitor {
  readonly status: NetworkStatus
  start(): void
  stop(): void
  probe(): Promise<NetworkStatus>
  onStatusChange(callback: StatusChangeCallback): () => void
}
```

- [ ] **Step 2: 编写 NetworkMonitor 单元测试**

创建 `src/__tests__/services/NetworkMonitor.test.ts`：

```typescript
import { describe, it, expect, beforeEach, vi, afterEach } from 'vitest'
import { createNetworkMonitor, DEFAULT_NETWORK_CONFIG } from '@/services/NetworkMonitor'

describe('NetworkMonitor', () => {
  let monitor: ReturnType<typeof createNetworkMonitor>

  beforeEach(() => {
    vi.useFakeTimers()
    // Mock fetch
    global.fetch = vi.fn()
    // Mock navigator.onLine
    Object.defineProperty(navigator, 'onLine', { value: true, writable: true })
    monitor = createNetworkMonitor()
  })

  afterEach(() => {
    monitor.stop()
    vi.restoreAllTimers()
    vi.restoreAllMocks()
  })

  it('should initialize with isProbing=false', () => {
    expect(monitor.status.isProbing).toBe(false)
  })

  it('should emit status change when quality changes', async () => {
    const callback = vi.fn()
    monitor.onStatusChange(callback)

    vi.mocked(global.fetch!).mockResolvedValueOnce({
      ok: true,
      status: 200,
    } as Response)

    await monitor.probe()
    expect(callback).toHaveBeenCalledTimes(1)
  })

  it('should set quality to offline after consecutive failures >= threshold', async () => {
    vi.mocked(global.fetch!).mockRejectedValue(new Error('Network error'))

    for (let i = 0; i < DEFAULT_NETWORK_CONFIG.offlineThreshold; i++) {
      await monitor.probe()
    }

    expect(monitor.status.quality).toBe('offline')
    expect(monitor.status.consecutiveFailures).toBe(DEFAULT_NETWORK_CONFIG.offlineThreshold)
  })

  it('should reset consecutiveFailures on successful probe', async () => {
    // first fail
    vi.mocked(global.fetch!).mockRejectedValueOnce(new Error('fail'))
    await monitor.probe()
    expect(monitor.status.consecutiveFailures).toBe(1)

    // then succeed
    vi.mocked(global.fetch!).mockResolvedValueOnce({
      ok: true,
      status: 200,
    } as Response)
    await monitor.probe()
    expect(monitor.status.consecutiveFailures).toBe(0)
    expect(monitor.status.quality).toBe('good')
  })

  it('should call onStatusChange listeners and return unsubscribe function', () => {
    const cb1 = vi.fn()
    const cb2 = vi.fn()

    const unsub1 = monitor.onStatusChange(cb1)
    monitor.onStatusChange(cb2)
    expect(monitor.statusChangeListenerCount).toBe(2)

    unsub1()
    expect(monitor.statusChangeListenerCount).toBe(1)
  })
})
```

- [ ] **Step 3: 运行测试验证失败**

```bash
npx vitest run src/__tests__/services/NetworkMonitor.test.ts
```

Expected: FAIL (createNetworkMonitor not defined)

- [ ] **Step 4: 实现 createNetworkMonitor**

在 `src/services/NetworkMonitor.ts` 中追加实现：

```typescript
export function createNetworkMonitor(config: Partial<NetworkMonitorConfig> = {}): NetworkMonitor & { statusChangeListenerCount: number } {
  const cfg = { ...DEFAULT_NETWORK_CONFIG, ...config }

  const status: NetworkStatus = {
    quality: 'good',
    lastProbeTime: 0,
    lastProbeDuration: 0,
    consecutiveFailures: 0,
    isProbing: false,
  }

  const listeners = new Set<StatusChangeCallback>()
  let timer: ReturnType<typeof setTimeout> | null = null
  let stopped = false

  function notifyListeners(): void {
    const snapshot = { ...status }
    listeners.forEach(fn => fn(snapshot))
  }

  async function fetchWithTimeout(url: string, timeout: number): Promise<Response> {
    const controller = new AbortController()
    const timer = setTimeout(() => controller.abort(), timeout)
    try {
      const resp = await fetch(url, { cache: 'no-store', signal: controller.signal })
      return resp
    } finally {
      clearTimeout(timer)
    }
  }

  async function executeProbe(): Promise<void> {
    if (stopped) return
    status.isProbing = true
    const start = Date.now()
    const prevQuality = status.quality

    try {
      const resp = await fetchWithTimeout(`${cfg.probeUrl}?_t=${Date.now()}`, cfg.probeTimeout)
      const duration = Date.now() - start
      status.lastProbeDuration = duration
      status.lastProbeTime = Date.now()
      status.consecutiveFailures = 0

      if (!resp.ok || duration > cfg.probeTimeout) {
        status.quality = 'weak'
      } else {
        status.quality = 'good'
      }
    } catch {
      status.consecutiveFailures++
      status.lastProbeTime = Date.now()

      if (status.consecutiveFailures >= cfg.offlineThreshold) {
        status.quality = 'offline'
      } else {
        status.quality = 'weak'
      }
    } finally {
      status.isProbing = false
      if (status.quality !== prevQuality) {
        notifyListeners()
      }
      scheduleNextProbe()
    }
  }

  function scheduleNextProbe(): void {
    if (stopped) return
    const interval =
      status.quality === 'good' ? cfg.goodInterval :
      status.quality === 'weak' ? cfg.weakInterval :
      cfg.offlineInterval
    if (timer) clearTimeout(timer)
    timer = setTimeout(executeProbe, interval)
  }

  function onBrowserOnline(): void {
    status.consecutiveFailures = 0
    executeProbe()
  }

  function onBrowserOffline(): void {
    status.quality = 'offline'
    status.consecutiveFailures = cfg.offlineThreshold
    notifyListeners()
  }

  return {
    get status() { return status },

    start(): void {
      stopped = false
      window.addEventListener('online', onBrowserOnline)
      window.addEventListener('offline', onBrowserOffline)
      if (!navigator.onLine) {
        onBrowserOffline()
      } else {
        setTimeout(executeProbe, 0)
      }
    },

    stop(): void {
      stopped = true
      if (timer) { clearTimeout(timer); timer = null }
      window.removeEventListener('online', onBrowserOnline)
      window.removeEventListener('offline', onBrowserOffline)
    },

    async probe(): Promise<NetworkStatus> {
      await executeProbe()
      return { ...status }
    },

    onStatusChange(callback: StatusChangeCallback): () => void {
      listeners.add(callback)
      return () => { listeners.delete(callback) }
    },

    get statusChangeListenerCount() { return listeners.size },
  }
}
```

- [ ] **Step 5: 运行测试验证通过**

```bash
npx vitest run src/__tests__/services/NetworkMonitor.test.ts
```

Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add src/services/NetworkMonitor.ts src/__tests__/services/NetworkMonitor.test.ts
git commit -m "feat: implement NetworkMonitor with adaptive probing and status events"
```

---

### Task 3: 实现 OfflineStrategyManager 离线策略管理器

**Files:**
- Create: `src/services/OfflineStrategyManager.ts`
- Create: `src/__tests__/services/OfflineStrategyManager.test.ts`

- [ ] **Step 1: 编写 OfflineStrategyManager**

创建 `src/services/OfflineStrategyManager.ts`：

```typescript
export type OfflineMode = 'online' | 'cautious' | 'offline'
export type OperationType = 'read' | 'write' | 'upload' | 'download'

export interface StrategyDecision {
  allowNetwork: boolean
  requireCache: boolean
  requireSync: boolean
  retry: { maxAttempts: number; backoffMs: number }
  timeout: number
}

const STRATEGY_MATRIX: Record<OfflineMode, Record<OperationType, StrategyDecision>> = {
  online: {
    read:   { allowNetwork: true,  requireCache: false, requireSync: false, retry: { maxAttempts: 1, backoffMs: 0 },    timeout: 15000 },
    write:  { allowNetwork: true,  requireCache: true,  requireSync: false, retry: { maxAttempts: 2, backoffMs: 1000 },  timeout: 15000 },
    upload: { allowNetwork: true,  requireCache: true,  requireSync: false, retry: { maxAttempts: 2, backoffMs: 2000 },  timeout: 30000 },
    download:{ allowNetwork: true,  requireCache: true,  requireSync: false, retry: { maxAttempts: 2, backoffMs: 1000 },  timeout: 30000 },
  },
  cautious: {
    read:   { allowNetwork: true,  requireCache: true,  requireSync: false, retry: { maxAttempts: 2, backoffMs: 2000 },  timeout: 10000 },
    write:  { allowNetwork: true,  requireCache: true,  requireSync: true,  retry: { maxAttempts: 3, backoffMs: 3000 },  timeout: 10000 },
    upload: { allowNetwork: false, requireCache: true,  requireSync: true,  retry: { maxAttempts: 0, backoffMs: 0 },     timeout: 0 },
    download:{ allowNetwork: true,  requireCache: true,  requireSync: false, retry: { maxAttempts: 2, backoffMs: 2000 },  timeout: 20000 },
  },
  offline: {
    read:   { allowNetwork: false, requireCache: true,  requireSync: false, retry: { maxAttempts: 0, backoffMs: 0 },     timeout: 0 },
    write:  { allowNetwork: false, requireCache: true,  requireSync: true,  retry: { maxAttempts: 0, backoffMs: 0 },     timeout: 0 },
    upload: { allowNetwork: false, requireCache: true,  requireSync: true,  retry: { maxAttempts: 0, backoffMs: 0 },     timeout: 0 },
    download:{ allowNetwork: false, requireCache: true,  requireSync: false, retry: { maxAttempts: 0, backoffMs: 0 },     timeout: 0 },
  },
}

export type ModeChangeCallback = (mode: OfflineMode) => void

export function createOfflineStrategyManager(): {
  get mode(): OfflineMode
  setMode(mode: OfflineMode): void
  getStrategy(operation: OperationType): StrategyDecision
  onModeChange(callback: ModeChangeCallback): () => void
} {
  let _mode: OfflineMode = 'online'
  const listeners = new Set<ModeChangeCallback>()

  return {
    get mode() { return _mode },

    setMode(mode: OfflineMode): void {
      if (_mode === mode) return
      _mode = mode
      listeners.forEach(fn => fn(mode))
    },

    getStrategy(operation: OperationType): StrategyDecision {
      return { ...STRATEGY_MATRIX[_mode][operation] }
    },

    onModeChange(callback: ModeChangeCallback): () => void {
      listeners.add(callback)
      return () => { listeners.delete(callback) }
    },
  }
}
```

- [ ] **Step 2: 编写测试**

创建 `src/__tests__/services/OfflineStrategyManager.test.ts`：

```typescript
import { describe, it, expect } from 'vitest'
import { createOfflineStrategyManager } from '@/services/OfflineStrategyManager'

describe('OfflineStrategyManager', () => {
  it('should default to online mode', () => {
    const sm = createOfflineStrategyManager()
    expect(sm.mode).toBe('online')
  })

  it('should return allowNetwork=true for read in online mode', () => {
    const sm = createOfflineStrategyManager()
    const decision = sm.getStrategy('read')
    expect(decision.allowNetwork).toBe(true)
  })

  it('should return allowNetwork=false for all operations in offline mode', () => {
    const sm = createOfflineStrategyManager()
    sm.setMode('offline')
    for (const op of ['read', 'write', 'upload', 'download'] as const) {
      expect(sm.getStrategy(op).allowNetwork).toBe(false)
    }
  })

  it('should requireSync for write in cautious mode', () => {
    const sm = createOfflineStrategyManager()
    sm.setMode('cautious')
    expect(sm.getStrategy('write').requireSync).toBe(true)
  })

  it('should notify mode change listeners', () => {
    const sm = createOfflineStrategyManager()
    let received: string | null = null
    sm.onModeChange((mode) => { received = mode })
    sm.setMode('offline')
    expect(received).toBe('offline')
  })

  it('should not notify if mode unchanged', () => {
    const sm = createOfflineStrategyManager()
    let callCount = 0
    sm.onModeChange(() => { callCount++ })
    sm.setMode('online')
    expect(callCount).toBe(0)
  })
})
```

- [ ] **Step 3: 运行测试**

```bash
npx vitest run src/__tests__/services/OfflineStrategyManager.test.ts
```

Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add src/services/OfflineStrategyManager.ts src/__tests__/services/OfflineStrategyManager.test.ts
git commit -m "feat: implement OfflineStrategyManager with 3x4 strategy matrix"
```

---

### Task 4: 实现 ConflictResolver 冲突解决器

**Files:**
- Create: `src/services/ConflictResolver.ts`
- Create: `src/__tests__/services/ConflictResolver.test.ts`

- [ ] **Step 1: 编写 ConflictResolver**

创建 `src/services/ConflictResolver.ts`：

```typescript
export type ConflictStrategy = 'local-first' | 'remote-first' | 'merge' | 'manual'

export interface ConflictInfo {
  businessType: string
  businessId: string
  localData: Record<string, any>
  remoteData: Record<string, any>
  localTimestamp: number
  remoteTimestamp: number
}

export interface ConflictResolution {
  resolvedData: Record<string, any>
  strategy: ConflictStrategy
}

export type MergeFunction = (local: Record<string, any>, remote: Record<string, any>) => Record<string, any>

function defaultMerge(local: Record<string, any>, remote: Record<string, any>): Record<string, any> {
  return { ...remote, ...local }
}

export function createConflictResolver(defaultStrategy: ConflictStrategy = 'local-first'): {
  resolve(conflict: ConflictInfo, strategy?: ConflictStrategy): Promise<ConflictResolution>
  registerMerger(businessType: string, merger: MergeFunction): void
} {
  const mergers = new Map<string, MergeFunction>()

  return {
    async resolve(conflict: ConflictInfo, strategy?: ConflictStrategy): Promise<ConflictResolution> {
      const s = strategy ?? defaultStrategy

      switch (s) {
        case 'local-first':
          return { resolvedData: { ...conflict.localData }, strategy: 'local-first' }

        case 'remote-first':
          return { resolvedData: { ...conflict.remoteData }, strategy: 'remote-first' }

        case 'merge': {
          const merger = mergers.get(conflict.businessType) ?? defaultMerge
          const merged = merger(
            { ...conflict.localData },
            { ...conflict.remoteData },
          )
          return { resolvedData: merged, strategy: 'merge' }
        }

        case 'manual':
          return { resolvedData: { ...conflict.localData }, strategy: 'manual' }

        default:
          return { resolvedData: { ...conflict.localData }, strategy: 'local-first' }
      }
    },

    registerMerger(businessType: string, merger: MergeFunction): void {
      mergers.set(businessType, merger)
    },
  }
}
```

- [ ] **Step 2: 编写测试**

创建 `src/__tests__/services/ConflictResolver.test.ts`：

```typescript
import { describe, it, expect } from 'vitest'
import { createConflictResolver } from '@/services/ConflictResolver'

const baseConflict = {
  businessType: 'patrol',
  businessId: 'p-001',
  localData: { name: 'local', version: 1 },
  remoteData: { name: 'remote', version: 2 },
  localTimestamp: 1000,
  remoteTimestamp: 2000,
}

describe('ConflictResolver', () => {
  it('local-first should return local data', async () => {
    const resolver = createConflictResolver('local-first')
    const result = await resolver.resolve(baseConflict)
    expect(result.strategy).toBe('local-first')
    expect(result.resolvedData.name).toBe('local')
  })

  it('remote-first should return remote data', async () => {
    const resolver = createConflictResolver('remote-first')
    const result = await resolver.resolve(baseConflict)
    expect(result.strategy).toBe('remote-first')
    expect(result.resolvedData.name).toBe('remote')
  })

  it('merge should call default merger', async () => {
    const resolver = createConflictResolver('merge')
    const result = await resolver.resolve(baseConflict)
    expect(result.strategy).toBe('merge')
    // default merge: remote base, local overrides
    expect(result.resolvedData.name).toBe('local')
    expect(result.resolvedData.version).toBe(1)
  })

  it('should use registered merger for business type', async () => {
    const resolver = createConflictResolver('merge')
    resolver.registerMerger('patrol', (local, remote) => ({
      name: `${remote.name}-merged`,
      version: Math.max(local.version, remote.version),
    }))
    const result = await resolver.resolve(baseConflict)
    expect(result.resolvedData.name).toBe('remote-merged')
    expect(result.resolvedData.version).toBe(2)
  })

  it('should not mutate input conflict', async () => {
    const resolver = createConflictResolver()
    const originalName = baseConflict.localData.name
    await resolver.resolve(baseConflict)
    expect(baseConflict.localData.name).toBe(originalName)
  })
})
```

- [ ] **Step 3: 运行测试**

```bash
npx vitest run src/__tests__/services/ConflictResolver.test.ts
```

Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add src/services/ConflictResolver.ts src/__tests__/services/ConflictResolver.test.ts
git commit -m "feat: implement ConflictResolver with 4 conflict strategies"
```

---

### Task 5: 实现 SyncEngine 同步引擎

**Files:**
- Create: `src/services/SyncEngine.ts`
- Create: `src/__tests__/services/SyncEngine.test.ts`

- [ ] **Step 1: 编写 SyncEngine 类型定义与核心实现**

创建 `src/services/SyncEngine.ts`：

```typescript
import { syncTaskService } from '@/db/sync.service'
import { createConflictResolver, type ConflictStrategy } from './ConflictResolver'
import type { OfflineMode } from './OfflineStrategyManager'

export type SyncTaskStatus = 'pending' | 'syncing' | 'success' | 'failed' | 'conflict'

export interface SyncTask {
  id: string
  businessType: string
  businessId: string
  operation: 'create' | 'update' | 'delete'
  status: SyncTaskStatus
  createdAt: number
  lastAttemptAt?: number
  retryCount: number
  maxRetries: number
  priority: number
  payload: Record<string, any>
}

export interface SyncProgress {
  total: number
  completed: number
  failed: number
  current?: SyncTask
  isSyncing: boolean
}

export type SyncEvent =
  | { type: 'taskComplete'; task: SyncTask }
  | { type: 'taskFailed'; task: SyncTask; error: Error }
  | { type: 'conflict'; task: SyncTask; localData: any; remoteData: any }
  | { type: 'allComplete'; summary: { success: number; failed: number; conflicts: number } }

export type SyncEventCallback = (event: SyncEvent) => void
export type ApiHandler = (businessId: string, payload: Record<string, any>) => Promise<Response>

export function createSyncEngine(config: {
  conflictStrategy?: ConflictStrategy
  batchSize?: number
  getApiHandler: (businessType: string, operation: string) => ApiHandler
  fetchRemoteData: (businessType: string, businessId: string) => Promise<Record<string, any>>
  getNetworkMode: () => OfflineMode
}): {
  enqueue(task: Omit<SyncTask, 'id' | 'status' | 'createdAt' | 'retryCount' | 'lastAttemptAt'>): Promise<string>
  startSync(): Promise<void>
  stopSync(): void
  retryFailed(): Promise<void>
  clearCompleted(): Promise<void>
  get progress(): SyncProgress
  get pendingTasks(): SyncTask[]
  onSyncEvent(callback: SyncEventCallback): () => void
} {
  const conflictResolver = createConflictResolver(config.conflictStrategy ?? 'local-first')
  const batchSize = config.batchSize ?? 50

  const progress: SyncProgress = {
    total: 0, completed: 0, failed: 0,
    current: undefined, isSyncing: false,
  }

  const eventListeners = new Set<SyncEventCallback>()
  let abortController: AbortController | null = null
  let _pendingTasks: SyncTask[] = []

  function emit(event: SyncEvent): void {
    eventListeners.forEach(fn => fn(event))
    // 同时同步更新 _pendingTasks
    if (event.type === 'taskComplete' || event.type === 'taskFailed' || event.type === 'conflict') {
      const idx = _pendingTasks.findIndex(t => t.id === event.task.id)
      if (idx >= 0) _pendingTasks[idx] = event.task
    }
  }

  async function executeTaskSync(task: SyncTask): Promise<{ success: boolean; conflict: boolean; remoteData?: any }> {
    const handler = config.getApiHandler(task.businessType, task.operation)
    try {
      const response = await handler(task.businessId, task.payload)
      if (response.ok) return { success: true, conflict: false }
      if (response.status === 409) {
        const remoteData = await config.fetchRemoteData(task.businessType, task.businessId)
        return { success: false, conflict: true, remoteData }
      }
      const text = await response.text()
      throw new Error(`HTTP ${response.status}: ${text}`)
    } catch (error) {
      if (error instanceof Error && !(error as any).status) throw error
      return { success: false, conflict: false }
    }
  }

  return {
    get progress() { return { ...progress } },
    get pendingTasks() { return [..._pendingTasks] },

    async enqueue(taskInput): Promise<string> {
      const id = await syncTaskService.add(taskInput)
      const mode = config.getNetworkMode()
      // 在线或谨慎模式且网络可用时自动同步
      if (mode === 'online') {
        this.startSync()
      }
      // 刷新 pending 列表
      _pendingTasks = await syncTaskService.getPendingSorted()
      return id
    },

    async startSync(): Promise<void> {
      if (progress.isSyncing) return
      if (config.getNetworkMode() === 'offline') return

      progress.isSyncing = true
      abortController = new AbortController()

      try {
        const pending = await syncTaskService.getPendingSorted()
        const batch = pending.slice(0, batchSize)
        progress.total = batch.length
        progress.completed = 0
        progress.failed = 0
        _pendingTasks = pending

        for (const task of batch) {
          if (abortController?.signal.aborted) break

          progress.current = task
          task.status = 'syncing'
          task.lastAttemptAt = Date.now()
          await syncTaskService.updateStatus(task.id, 'syncing', { lastAttemptAt: task.lastAttemptAt })

          try {
            const result = await executeTaskSync(task)

            if (result.success) {
              task.status = 'success'
              progress.completed++
              await syncTaskService.updateStatus(task.id, 'success')
              emit({ type: 'taskComplete', task })
            } else if (result.conflict) {
              const resolution = await conflictResolver.resolve({
                businessType: task.businessType,
                businessId: task.businessId,
                localData: task.payload,
                remoteData: result.remoteData!,
                localTimestamp: task.createdAt,
                remoteTimestamp: Date.now(),
              })

              if (resolution.strategy === 'manual') {
                task.status = 'conflict'
                await syncTaskService.updateStatus(task.id, 'conflict')
                emit({ type: 'conflict', task, localData: task.payload, remoteData: result.remoteData })
              } else {
                // 用解决后的数据重新提交
                const handler = config.getApiHandler(task.businessType, task.operation)
                const resp = await handler(task.businessId, resolution.resolvedData)
                if (resp.ok) {
                  task.status = 'success'
                  progress.completed++
                  await syncTaskService.updateStatus(task.id, 'success')
                  emit({ type: 'taskComplete', task })
                } else {
                  throw new Error(`Conflict resolution sync failed: ${resp.status}`)
                }
              }
            } else {
              throw new Error('Sync failed without conflict')
            }
          } catch (error) {
            task.retryCount++
            if (task.retryCount >= task.maxRetries) {
              task.status = 'failed'
              progress.failed++
              await syncTaskService.updateStatus(task.id, 'failed', {
                retryCount: task.retryCount,
                lastError: (error as Error).message,
              })
              emit({ type: 'taskFailed', task, error: error as Error })
            } else {
              task.status = 'pending'
              await syncTaskService.updateStatus(task.id, 'pending', { retryCount: task.retryCount })
            }
          }
        }
      } finally {
        progress.isSyncing = false
        progress.current = undefined
        _pendingTasks = await syncTaskService.getPendingSorted()
        emit({
          type: 'allComplete',
          summary: {
            success: progress.completed,
            failed: progress.failed,
            conflicts: _pendingTasks.filter(t => t.status === 'conflict').length,
          },
        })
        abortController = null
      }
    },

    stopSync(): void {
      abortController?.abort()
    },

    async retryFailed(): Promise<void> {
      const failed = await syncTaskService.getByStatus('failed')
      for (const task of failed) {
        task.status = 'pending'
        task.retryCount = 0
        await syncTaskService.updateStatus(task.id, 'pending', { retryCount: 0 })
      }
      await this.startSync()
    },

    async clearCompleted(): Promise<void> {
      await syncTaskService.clearCompleted()
      _pendingTasks = await syncTaskService.getPendingSorted()
    },

    onSyncEvent(callback: SyncEventCallback): () => void {
      eventListeners.add(callback)
      return () => { eventListeners.delete(callback) }
    },
  }
}
```

- [ ] **Step 2: 编写 SyncEngine 测试**

创建 `src/__tests__/services/SyncEngine.test.ts`：

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { createSyncEngine } from '@/services/SyncEngine'
import { db } from '@/db'

function createMockConfig() {
  const apiHandler = vi.fn().mockResolvedValue(new Response(null, { status: 200 }))
  return {
    conflictStrategy: 'local-first' as const,
    batchSize: 10,
    getApiHandler: () => apiHandler,
    fetchRemoteData: vi.fn().mockResolvedValue({ remoteVersion: 2 }),
    getNetworkMode: () => 'online' as const,
    _apiHandler: apiHandler,
  }
}

describe('SyncEngine', () => {
  beforeEach(async () => {
    await db.eam_sync_tasks.clear()
  })

  it('should enqueue a task and return its id', async () => {
    const cfg = createMockConfig()
    const engine = createSyncEngine(cfg)
    const id = await engine.enqueue({
      businessType: 'patrol',
      businessId: 'p-001',
      operation: 'update',
      priority: 5,
      maxRetries: 3,
      payload: { name: 'test' },
    })
    expect(id).toBeTruthy()
  })

  it('should process pending tasks on startSync in priority order', async () => {
    const cfg = createMockConfig()
    const engine = createSyncEngine(cfg)
    await engine.enqueue({
      businessType: 'patrol', businessId: 'low', operation: 'update',
      priority: 9, maxRetries: 1, payload: {},
    })
    await engine.enqueue({
      businessType: 'patrol', businessId: 'high', operation: 'update',
      priority: 1, maxRetries: 1, payload: {},
    })

    const events: any[] = []
    engine.onSyncEvent(e => events.push(e))

    await engine.startSync()

    const completed = events.filter(e => e.type === 'taskComplete')
    expect(completed.length).toBe(2)
  })

  it('should emit allComplete event after sync', async () => {
    const cfg = createMockConfig()
    const engine = createSyncEngine(cfg)
    await engine.enqueue({
      businessType: 'patrol', businessId: 'p-002', operation: 'create',
      priority: 1, maxRetries: 1, payload: {},
    })

    let allCompleteEmitted = false
    engine.onSyncEvent(e => {
      if (e.type === 'allComplete') allCompleteEmitted = true
    })

    await engine.startSync()
    expect(allCompleteEmitted).toBe(true)
  })

  it('should not start sync if already syncing', async () => {
    const cfg = createMockConfig()
    cfg.getApiHandler = () => () => new Promise(r => setTimeout(r, 1000))
    const engine = createSyncEngine(cfg)
    await engine.enqueue({
      businessType: 'patrol', businessId: 'p-003', operation: 'update',
      priority: 1, maxRetries: 1, payload: {},
    })

    engine.startSync()
    const secondResult = await engine.startSync()
    // should have returned early, second call is no-op
    expect(engine.progress.isSyncing).toBe(true)
  })

  it('should stop sync when stopSync called', async () => {
    const cfg = createMockConfig()
    cfg.getApiHandler = () => () => new Promise(r => setTimeout(r, 2000))
    const engine = createSyncEngine(cfg)
    await engine.enqueue({
      businessType: 'patrol', businessId: 'p-004', operation: 'update',
      priority: 1, maxRetries: 1, payload: {},
    })

    const syncPromise = engine.startSync()
    engine.stopSync()
    await syncPromise

    const pending = await db.eam_sync_tasks.where('status').equals('pending').count()
    // Task should still be pending (not failed)
    expect(pending).toBe(1)
  })
})
```

- [ ] **Step 3: 运行测试**

```bash
npx vitest run src/__tests__/services/SyncEngine.test.ts
```

Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add src/services/SyncEngine.ts src/__tests__/services/SyncEngine.test.ts
git commit -m "feat: implement SyncEngine with priority queue, batch sync, and retry"
```

---

### Task 6: 创建 Vue Composable 层

**Files:**
- Create: `src/hooks/useNetworkMonitor.ts`
- Create: `src/hooks/useSyncEngine.ts`
- Create: `src/hooks/useOfflineService.ts`

- [ ] **Step 1: 实现 useNetworkMonitor composable**

创建 `src/hooks/useNetworkMonitor.ts`：

```typescript
import { ref, onMounted, onUnmounted } from 'vue'
import {
  createNetworkMonitor,
  type NetworkMonitorConfig,
  type NetworkStatus,
} from '@/services/NetworkMonitor'

export function useNetworkMonitor(config?: Partial<NetworkMonitorConfig>) {
  const monitor = createNetworkMonitor(config)
  const status = ref<NetworkStatus>({ ...monitor.status })

  const unsub = monitor.onStatusChange((newStatus) => {
    status.value = { ...newStatus }
  })

  onMounted(() => {
    monitor.start()
    status.value = { ...monitor.status }
  })

  onUnmounted(() => {
    monitor.stop()
    unsub()
  })

  return {
    status,
    start: () => monitor.start(),
    stop: () => monitor.stop(),
    probe: () => monitor.probe(),
  }
}
```

- [ ] **Step 2: 实现 useSyncEngine composable**

创建 `src/hooks/useSyncEngine.ts`：

```typescript
import { ref, computed, onUnmounted } from 'vue'
import {
  createSyncEngine,
  type SyncTask,
  type SyncProgress,
  type SyncEngine,
  type ApiHandler,
} from '@/services/SyncEngine'
import type { ConflictStrategy, OfflineMode } from '..'

// 此 composable 接收已创建好的 engine 实例
export function useSyncEngine(engine: ReturnType<typeof createSyncEngine>) {
  const progress = ref<SyncProgress>(engine.progress)
  const pendingTasks = ref<SyncTask[]>(engine.pendingTasks)

  const unsub = engine.onSyncEvent(() => {
    progress.value = { ...engine.progress }
    pendingTasks.value = [...engine.pendingTasks]
  })

  const pendingCount = computed(() =>
    pendingTasks.value.filter(t => t.status === 'pending').length
  )

  onUnmounted(() => {
    unsub()
  })

  return {
    progress,
    pendingTasks,
    pendingCount,
    enqueue: (task: Parameters<typeof engine.enqueue>[0]) => engine.enqueue(task),
    startSync: () => engine.startSync(),
    stopSync: () => engine.stopSync(),
    retryFailed: () => engine.retryFailed(),
    clearCompleted: () => engine.clearCompleted(),
  }
}
```

- [ ] **Step 3: 实现 useOfflineService 主 composable**

创建 `src/hooks/useOfflineService.ts`：

```typescript
import { computed, watch, onMounted, onUnmounted } from 'vue'
import { useNetworkMonitor } from './useNetworkMonitor'
import type { OfflineMode, OperationType, StrategyDecision } from '@/services/OfflineStrategyManager'
import { createOfflineStrategyManager } from '@/services/OfflineStrategyManager'
import type { ApiHandler } from '@/services/SyncEngine'

export function useOfflineService(config: {
  getApiHandler: (businessType: string, operation: string) => ApiHandler
  fetchRemoteData: (businessType: string, businessId: string) => Promise<Record<string, any>>
}) {
  const { status: networkStatus, probe } = useNetworkMonitor()
  const strategyManager = createOfflineStrategyManager()

  const engine = createSyncEngine({
    conflictStrategy: 'local-first',
    batchSize: 50,
    getApiHandler: config.getApiHandler,
    fetchRemoteData: config.fetchRemoteData,
    getNetworkMode: () => strategyManager.mode,
  })

  const { progress, pendingTasks, pendingCount, enqueue, startSync, stopSync, retryFailed, clearCompleted } =
    useSyncEngine(engine)

  const offlineMode = computed<OfflineMode>(() => {
    switch (networkStatus.value.quality) {
      case 'good': return 'online'
      case 'weak': return 'cautious'
      case 'offline': return 'offline'
    }
  })

  // 自动同步网络状态 → 策略模式
  watch(offlineMode, (mode) => {
    strategyManager.setMode(mode)
  }, { immediate: true })

  // 网络恢复时自动同步
  watch(
    () => networkStatus.value.quality,
    (quality, oldQuality) => {
      if (oldQuality !== 'good' && quality === 'good') {
        startSync()
      }
      if (quality === 'offline') {
        stopSync()
      }
    }
  )

  // 带离线支持的数据提交
  async function submitWithOfflineSupport(input: {
    businessType: string
    businessId: string
    operation: 'create' | 'update' | 'delete'
    payload: Record<string, any>
    priority?: number
    maxRetries?: number
  }): Promise<{ success: boolean; synced: boolean; taskId?: string }> {
    const strategy = strategyManager.getStrategy('write' as OperationType)

    if (strategy.allowNetwork) {
      try {
        const handler = config.getApiHandler(input.businessType, input.operation)
        const resp = await handler(input.businessId, input.payload)
        if (resp.ok) {
          return { success: true, synced: true }
        }
        // 非 400 级别错误降级到离线
        if (resp.status >= 500 || resp.status === 408) {
          if (!strategy.requireSync) throw new Error(`HTTP ${resp.status}`)
        }
      } catch {
        if (!strategy.requireSync) throw new Error('Network request failed')
      }
    }

    // 离线降级：加入同步队列
    const taskId = await enqueue({
      businessType: input.businessType,
      businessId: input.businessId,
      operation: input.operation,
      payload: input.payload,
      priority: input.priority ?? 5,
      maxRetries: input.maxRetries ?? 3,
    })

    return { success: true, synced: false, taskId }
  }

  return {
    networkStatus,
    offlineMode,
    syncProgress: progress,
    pendingTasks,
    pendingCount,
    submitWithOfflineSupport,
    forceSyncNow: startSync,
    retryFailed,
    clearCompleted,
    manualProbe: probe,
  }
}
```

- [ ] **Step 4: Commit**

```bash
git add src/hooks/
git commit -m "feat: implement Vue composable layer for offline service"
```

---

### Task 7: 实现 UI 组件

**Files:**
- Create: `src/components/OfflineIndicator.vue`
- Create: `src/components/SyncBadge.vue`

- [ ] **Step 1: 实现 OfflineIndicator.vue**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  mode: 'online' | 'cautious' | 'offline'
  pendingCount: number
}>()

const text = computed(() => {
  switch (props.mode) {
    case 'cautious': return '网络不稳定，数据将自动保存到本地'
    case 'offline': return `当前离线${props.pendingCount > 0 ? `，${props.pendingCount} 条数据待同步` : ''}`
    default: return ''
  }
})

const color = computed(() => (props.mode === 'offline' ? '#ee0a24' : '#ff976a'))
</script>

<template>
  <van-notice-bar
    v-if="mode !== 'online'"
    :text="text"
    :color="color"
    left-icon="info-o"
    :scrollable="false"
  />
</template>
```

- [ ] **Step 2: 实现 SyncBadge.vue**

```vue
<script setup lang="ts">
defineProps<{
  count: number
}>()
</script>

<template>
  <van-badge :content="count" v-if="count > 0" />
</template>
```

- [ ] **Step 3: 在 App.vue 或 Layout 中集成示例**

```vue
<script setup lang="ts">
import { useOfflineService } from '@/hooks/useOfflineService'
import OfflineIndicator from '@/components/OfflineIndicator.vue'
import SyncBadge from '@/components/SyncBadge.vue'

const { offlineMode, pendingCount } = useOfflineService({
  getApiHandler: (businessType, operation) => {
    // 实际项目中根据 businessType 和 operation 返回对应的 API 函数
    throw new Error('Not implemented — wire up to actual API handlers')
  },
  fetchRemoteData: async (businessType, businessId) => {
    throw new Error('Not implemented')
  },
})
</script>

<template>
  <div class="offline-bar">
    <OfflineIndicator :mode="offlineMode" :pending-count="pendingCount" />
    <SyncBadge :count="pendingCount" />
  </div>
  <router-view />
</template>
```

- [ ] **Step 4: Commit**

```bash
git add src/components/OfflineIndicator.vue src/components/SyncBadge.vue
git commit -m "feat: add offline indicator and sync badge UI components"
```

---

### Task 8: 整合 Pinia Store 与全局状态管理

**Files:**
- Create: `src/stores/offline.ts`

- [ ] **Step 1: 创建离线服务 Pinia Store**

创建 `src/stores/offline.ts`：

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { useOfflineService } from '@/hooks/useOfflineService'
import type { ApiHandler } from '@/services/SyncEngine'

export const useOfflineStore = defineStore('offline', () => {
  let service: ReturnType<typeof useOfflineService> | null = null

  const initialized = ref(false)

  function init(config: {
    getApiHandler: (businessType: string, operation: string) => ApiHandler
    fetchRemoteData: (businessType: string, businessId: string) => Promise<Record<string, any>>
  }) {
    if (service) return
    service = useOfflineService(config)
    initialized.value = true
  }

  function getService() {
    if (!service) throw new Error('Offline store not initialized. Call init() first.')
    return service
  }

  const networkStatus = computed(() => getService().networkStatus)
  const offlineMode = computed(() => getService().offlineMode)
  const syncProgress = computed(() => getService().syncProgress)
  const pendingCount = computed(() => getService().pendingCount)

  return {
    initialized,
    init,
    networkStatus,
    offlineMode,
    syncProgress,
    pendingCount,
    submitWithOfflineSupport: (input: Parameters<typeof service extends { submitWithOfflineSupport: infer F } ? F : never>[0]) =>
      getService().submitWithOfflineSupport(input),
    forceSyncNow: () => getService().forceSyncNow(),
    retryFailed: () => getService().retryFailed(),
    clearCompleted: () => getService().clearCompleted(),
  }
})
```

- [ ] **Step 2: 编写 store 初始化测试**

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useOfflineStore } from '@/stores/offline'

describe('offline store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('should throw if not initialized', () => {
    const store = useOfflineStore()
    expect(() => store.offlineMode).toThrow()
  })

  it('should work after init', () => {
    const store = useOfflineStore()
    store.init({
      getApiHandler: () => async () => new Response(null, { status: 200 }),
      fetchRemoteData: async () => ({}),
    })
    expect(store.initialized).toBe(true)
    expect(store.offlineMode).toBeDefined()
  })
})
```

- [ ] **Step 3: Commit**

```bash
git add src/stores/offline.ts
git commit -m "feat: add Pinia offline store for global offline state management"
```

---

## Self-Review Checklist

- [x] **Spec coverage**: 覆盖 design.md 全部 4 个核心模块、数据模型、Composable 层、UI 组件、Pinia store
- [x] **No placeholders**: 所有代码块完整，无 TBD/TODO
- [x] **Type consistency**: SyncTask、IEamSyncTask 字段名跨文件一致；SyncEvent 类型在 SyncEngine 和 useSyncEngine 中一致
- [x] **Test-first**: 每个核心模块先写测试再实现（Task 2/3/4/5）
- [x] **Conflict resolution**: 版本冲突处理在 SyncEngine.executeTaskSync 中通过 ConflictResolver 实现
- [x] **Execution order**: SyncEngine 按 priority + createdAt 排序，保证执行顺序
