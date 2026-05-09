# 需求文档：Three.js 模型爆炸图组件 (threejs-model-explode)

## 需求概述

面向「模型厂商批量提供 glTF/bin 装配体模型 + Web 端进行结构拆解展示」的业务场景，基于 Three.js 封装一套边界清晰、可组合的渲染与爆炸图组件。业务方只需传入模型资源地址与挂载容器，即可完成模型加载、渲染、轨道操作，并通过按钮触发「单次 / 循环」两种爆炸图动画的播放控制。

整体能力拆分为两个高内聚、低耦合的模块：`core` 渲染核心模块和 `explode` 爆炸图模块；两者之间仅通过最小接口 `ViewerHandle` 交互，便于独立测试、替换与复用。

---

## 功能需求

### FR-1: 模型加载与渲染（core 模块）

#### FR-1.1: glTF/bin 模型加载
- 系统应支持加载厂商提供的 `.gltf` + `.bin` 资源以及 `.glb` 单文件
- 模型加载过程中应提供进度回调（0..1）
- 加载完成后返回模型根节点、原始 glTF 对象、包围盒三项信息

#### FR-1.2: 场景与渲染器初始化
- 系统应根据业务方传入的 DOM 容器自动初始化 Scene、PerspectiveCamera、WebGLRenderer、默认灯光
- 渲染器应采用基于 `requestAnimationFrame` 的时间驱动循环（非帧驱动），保证不同刷新率设备上表现一致
- 渲染尺寸应跟随容器尺寸变化（通过 ResizeObserver）自动适配

#### FR-1.3: 相机自适应
- 模型加载完成后应根据模型包围盒自动调整相机位置与距离
- 业务方可通过 `cameraDistanceScale` 配置相机距离倍数（默认 2.5）
- 业务方可手动调用 `fitCameraToModel()` 重新对焦

#### FR-1.4: 轨道交互控制
- 默认启用 `OrbitControls`，支持旋转、平移、缩放
- 业务方可通过 `enableControls: false` 关闭交互

#### FR-1.5: 压缩扩展支持
- 系统应支持 DRACO 压缩的 glTF（`KHR_draco_mesh_compression`）
- 当业务方提供 `dracoDecoderPath` 时才启用 DRACO 解码器，避免无谓增加首屏体积

#### FR-1.6: 模型切换与资源释放
- 业务方可多次调用 `load(url)` 切换模型；旧模型应自动从场景移除并释放 GPU 资源
- 业务方可调用 `unload()` 卸载当前模型但保留 viewer
- 业务方可调用 `dispose()` 销毁 viewer，释放全部 WebGL 资源（geometry / material / texture / renderer）

#### FR-1.7: 渲染循环钩子
- 系统应对外暴露 `onBeforeRender(fn)` 钩子，返回取消订阅函数
- 钩子回调参数为 `(deltaMs, elapsedMs)`
- 钩子用于 `explode` 模块等外部模块挂接每帧更新逻辑，避免独立启动 RAF

---

### FR-2: 爆炸图播放控制（explode 模块）

#### FR-2.1: 零件分析
- 系统应遍历已加载的模型，识别出参与爆炸的「零件单元」
- 每个零件单元应记录：目标 Object3D、原始本地位置、世界中心、爆炸位移向量
- 分析结果缓存在 `ExplodeController` 内，不在每帧重复遍历场景树

#### FR-2.2: 分组策略
- 支持 `groupByParent: true`（默认）：按父 Group 作为爆炸单元，同一父节点下的 Mesh 一同移动
- 支持 `groupByParent: false`：每个 Mesh 独立作为爆炸单元

#### FR-2.3: 零件过滤
- 业务方可通过 `partFilter(mesh)` 回调过滤参与爆炸的 Mesh
- 被过滤掉的零件不出现在爆炸单元数组中，也不会在动画中移动

#### FR-2.4: 爆炸方向策略
- 支持 `radial`（默认）：从模型包围盒中心向外辐射
- 支持 `axis-x` / `axis-y` / `axis-z`：沿指定轴展开
- 支持自定义方向函数 `(partCenter, modelCenter) => Vector3`
- 当零件中心与模型中心重合时（退化情况），radial 模式固定使用向上方向避免 NaN

#### FR-2.5: 单次爆炸动画（once 模式）
- 调用 `play({ mode: 'once' })` 时从 0 推进到 1，到达终点后触发 `complete` 事件并进入 `idle` 状态
- 动画时长由 `durationMs` 配置（默认 1000ms）
- 支持自定义缓动函数（默认 easeInOutCubic），缓动函数必须满足 `easing(0) === 0 && easing(1) === 1`

#### FR-2.6: 循环爆炸动画（loop 模式）
- 调用 `play({ mode: 'loop' })` 时按 `exploding → paused-at-exploded → collapsing → paused-at-collapsed` 循环永不自行终止
- 两端停留时间由 `loopPauseMs` 配置（默认 600ms）

#### FR-2.7: 播放控制
- 业务方可调用 `pause()` 暂停当前动画，保留进度
- 业务方可调用 `resume()` 从暂停处继续
- 业务方可调用 `reset()` 停止动画并复原所有零件到原始位置

#### FR-2.8: 事件订阅
- 业务方可通过 `on(fn)` 订阅事件，返回取消订阅函数
- 事件类型包括：`start`、`progress`（含 progress、phase）、`complete`、`pause`、`resume`、`reset`

---

### FR-3: 模块边界与协作

#### FR-3.1: 最小接口约定
- `explode` 模块只能通过 `ViewerHandle` 接口访问 `core`
- `ViewerHandle` 仅暴露：`modelRoot`、`onBeforeRender(fn)`、`onModelLoaded(fn)`
- `explode` 模块不得直接引用 `WebGLRenderer`、`OrbitControls`、`Scene`、`Camera` 等 core 内部实现

#### FR-3.2: 渲染循环共享
- `explode` 模块不得启动独立的 `requestAnimationFrame` 循环
- 爆炸动画的每帧推进必须挂接在 `core` 的 `onBeforeRender` 钩子上

#### FR-3.3: 独立可测性
- `explode` 模块应支持注入 Mock `ViewerHandle` 进行单元测试
- `core` 模块中的 `SceneManager`、`RendererManager`、`ModelLoader` 等应可分别被替换或 Mock

---

### FR-4: 错误处理与降级

#### FR-4.1: 模型加载失败
- 加载 404 / 网络错误 / 格式错误时应抛出 `ModelLoadError(url, cause)`
- 加载失败不应破坏当前已渲染的模型（如有）
- 业务方可捕获异常后重试 `load()`

#### FR-4.2: DRACO 未配置但模型需要
- 若模型使用 draco 压缩但未配置 `dracoDecoderPath`，抛出明确错误信息 `DRACO decoder not configured`

#### FR-4.3: 容器尺寸为 0
- 当容器 `clientWidth === 0` 或 `clientHeight === 0` 时（如 `display: none`），渲染器应跳过渲染而不报错
- 容器尺寸恢复后通过 ResizeObserver 自动恢复渲染

#### FR-4.4: 模型未加载时创建 ExplodeController
- `modelRoot === null` 时创建 `ExplodeController` 不应抛异常
- 内部应订阅 `onModelLoaded`，加载完成后再执行零件分析
- 在加载前调用 `play()` 应进入「等待状态」，加载完成后自动开始（仅首次有效）

#### FR-4.5: 零件分析结果为空
- 当 `partFilter` 过滤掉所有零件或模型不含 Mesh 时，`analyze()` 返回空数组
- `play()` 应立即触发 `complete` 事件而非抛异常

#### FR-4.6: WebGL 上下文丢失
- 监听 `webglcontextlost` 事件并停止渲染循环
- 对外抛出事件供业务方感知；监听 `webglcontextrestored` 后可重新 `load()` 恢复

---

## 非功能需求

### NFR-1: 性能
- 零件分析仅在模型加载后执行一次，结果缓存
- 每帧位置更新为 O(N)，N 通常 < 200，单帧耗时应 < 1ms
- 爆炸动画仅修改 `target.position`，不主动触发 `updateMatrixWorld`（依赖 Three.js 自动传播）

### NFR-2: 资源安全
- `dispose()` 必须遍历 scene 调用 `geometry.dispose()` / `material.dispose()` / `texture.dispose()`
- 多次 `load()` → `dispose()` 循环不应产生内存泄漏（可通过 `renderer.info.memory` 验证）
- `dispose()` 后不应再有 RAF 循环在运行；注册的 `onBeforeRender` 回调不再被调用

### NFR-3: 可移植性
- 公开 API 提供 TypeScript 类型定义（`ViewerOptions`、`ExplodeOptions`、`ViewerHandle` 等）
- Three.js 版本兼容 `^0.160.0` 或以上
- 浏览器环境：现代浏览器（支持 WebGL2、ResizeObserver、IndexedDB 非必需）

### NFR-4: 可测性
- 关键函数（`PartAnalyzer.analyze`、`ExplodeAnimator.tick`、`ExplodeController.reset`）具有明确的前置条件、后置条件、循环不变量，便于形式化验证
- 提供可用于 fast-check 的正确性属性（PBT）

### NFR-5: 可观测性
- `ExplodeController` 对外发布 `start/progress/complete/pause/resume/reset` 事件
- `ModelViewer.load()` 支持 `onProgress(pct)` 进度回调
- `options.debug: true` 时可显示坐标轴和包围盒辅助调试

---

## 验收标准

### AC-1: 模型加载与渲染验收
- [ ] AC-1.1: 传入厂商提供的 `.gltf` + `.bin` 资源可成功加载并渲染
- [ ] AC-1.2: 传入 `.glb` 单文件可成功加载并渲染
- [ ] AC-1.3: 首次加载后相机自动对焦到模型包围盒，模型完整显示在可视区域内
- [ ] AC-1.4: 容器尺寸变化时 viewport 自动适配，无画面拉伸
- [ ] AC-1.5: 默认 OrbitControls 启用，可旋转、缩放、平移模型
- [ ] AC-1.6: 连续两次 `load()` 不同模型，旧模型被正确释放，新模型正常渲染
- [ ] AC-1.7: DRACO 压缩模型在配置 `dracoDecoderPath` 后可正常加载

### AC-2: 爆炸图功能验收
- [ ] AC-2.1: 点击「单次爆炸」按钮，零件按配置方向展开，动画结束后停留在爆炸态
- [ ] AC-2.2: 点击「循环爆炸」按钮，零件反复经历展开 → 停留 → 收回 → 停留，永不自行停止
- [ ] AC-2.3: 点击「复位」按钮，所有零件返回原始位置（误差 < 1e-6）
- [ ] AC-2.4: 动画播放过程中点击「暂停」，零件停留在当前位置；点击「恢复」从暂停处继续
- [ ] AC-2.5: once 模式下 `complete` 事件在进度到达 1 时触发，且仅触发一次
- [ ] AC-2.6: loop 模式下 `phase` 永远不会变成 `idle`（除非调用 reset/dispose）

### AC-3: 爆炸方向与过滤验收
- [ ] AC-3.1: `direction: 'radial'` 下零件沿中心向外辐射
- [ ] AC-3.2: `direction: 'axis-y'` 下零件沿 Y 轴正负方向分开
- [ ] AC-3.3: 自定义方向函数返回的向量能正确影响零件运动方向
- [ ] AC-3.4: `partFilter` 返回 false 的 Mesh 在动画中保持不动
- [ ] AC-3.5: `groupByParent: true` 下同一父 Group 下的 Mesh 作为整体移动

### AC-4: 模块边界验收
- [ ] AC-4.1: 将 `explode` 模块接入自定义 `ViewerHandle` Mock，不依赖 `ModelViewer` 也能运行单元测试
- [ ] AC-4.2: 全文搜索 `explode/` 目录，不出现 `WebGLRenderer` / `OrbitControls` 的直接 import
- [ ] AC-4.3: `explode` 模块不调用 `requestAnimationFrame`，仅使用 `onBeforeRender` 回调

### AC-5: 错误处理验收
- [ ] AC-5.1: 加载不存在的 URL 抛出 `ModelLoadError`，当前模型保持不变
- [ ] AC-5.2: 加载 draco 压缩模型但未配置解码器路径，抛出明确错误
- [ ] AC-5.3: 容器 `display: none` 时不报错且不耗 CPU；恢复显示后自动重绘
- [ ] AC-5.4: 模型未加载前创建 `ExplodeController` 并调用 `play()`，加载完成后自动开始爆炸
- [ ] AC-5.5: 全部 Mesh 被 `partFilter` 过滤掉时，`play()` 立即触发 `complete`，无异常

### AC-6: 资源释放验收
- [ ] AC-6.1: 调用 `viewer.dispose()` 后，`renderer.info.memory.geometries` 和 `textures` 归零（在该 viewer 范围内）
- [ ] AC-6.2: 调用 `viewer.dispose()` 后，RAF 循环停止（通过断点或 PerformanceObserver 验证）
- [ ] AC-6.3: `explode.dispose()` 后，之前注册到 `onBeforeRender` 的回调不再被调用
- [ ] AC-6.4: 连续 10 次 `load → dispose` 循环不出现内存持续增长

### AC-7: 正确性属性验收（PBT）
- [ ] AC-7.1: once 模式下任意 tick 序列中，progress 单调不降
- [ ] AC-7.2: 任意 tick 序列后 progress ∈ [0, 1]
- [ ] AC-7.3: 任意时刻 reset 后，所有零件位置与 originalPosition 距离 < 1e-6
- [ ] AC-7.4: position 增量始终与 displacement 共线（叉积接近 0）
- [ ] AC-7.5: loop 模式下任意 tick 序列后 phase !== 'idle'
- [ ] AC-7.6: pause 后任意 tick，progress 与 position 保持不变
- [ ] AC-7.7: 整个播放生命周期中 `originalPosition` 保持快照不变
