# 任务列表：Three.js 模型爆炸图组件 (threejs-model-explode)

## 任务 1: 项目脚手架与依赖

- [ ] 1.1 在 `threejs_explode/` 下初始化 TypeScript 项目（`package.json`、`tsconfig.json`），配置 ES Module 输出
- [ ] 1.2 安装 `three`、`@types/three` 依赖（版本 `^0.160.0` 或以上）
- [ ] 1.3 安装开发依赖：`vitest`、`fast-check`、`@vitest/coverage-v8`
- [ ] 1.4 创建目录骨架：`core/`、`explode/`、聚合入口 `index.ts`
- [ ] 1.5 配置 Vitest + JSDOM 环境，提供最小 WebGL Mock 供单测使用

## 任务 2: core 模块 - 类型定义与公共约定

- [ ] 2.1 创建 `core/types.ts`，定义 `ViewerOptions`、`ViewerHandle`、`RenderTickFn`、`ModelLoadResult`
- [ ] 2.2 定义 `ModelLoadError` 错误类，包含 `url` 与 `cause` 字段
- [ ] 2.3 为所有公开类型添加 JSDoc，标注默认值、取值范围、单位
- [ ] 2.4 在 `core/index.ts` 中只导出对外公开类型与 `ModelViewer` 类

## 任务 3: core 模块 - SceneManager

- [ ] 3.1 创建 `core/SceneManager.ts`，在构造函数中初始化 `THREE.Scene`、`PerspectiveCamera`、默认环境光
- [ ] 3.2 实现 `fitCameraToObject(object, distanceScale)`：基于包围盒自动设置相机位置
- [ ] 3.3 实现 `updateAspect(width, height)`：更新相机 aspect 并 `updateProjectionMatrix()`
- [ ] 3.4 实现 `dispose()`：清空 scene 子节点，释放灯光资源
- [ ] 3.5 编写单测：给定不同尺寸的 BoundingBox，验证相机距离计算正确

## 任务 4: core 模块 - RendererManager

- [ ] 4.1 创建 `core/RendererManager.ts`，封装 `WebGLRenderer`（antialias、pixelRatio 自适应）
- [ ] 4.2 实现时间驱动的 RAF 渲染循环 `start(scene, camera)` / `stop()`，每帧计算 `deltaMs`、`elapsedMs`
- [ ] 4.3 实现 `onBeforeRender(fn)` 订阅机制，返回取消订阅函数
- [ ] 4.4 实现 `setSize(width, height)` 与 `ResizeObserver` 联动；容器尺寸为 0 时跳过渲染
- [ ] 4.5 实现 `dispose()`：停止 RAF、清空订阅者、`renderer.dispose()`、释放 canvas context
- [ ] 4.6 监听 `webglcontextlost` / `webglcontextrestored` 事件并对外暴露
- [ ] 4.7 编写单测（使用 Mock WebGLRenderer）：验证订阅取消、tick 调用顺序、dispose 清理

## 任务 5: core 模块 - ControlsManager

- [ ] 5.1 创建 `core/ControlsManager.ts`，封装 `OrbitControls`
- [ ] 5.2 `enableControls: false` 时不创建实例，对外返回 `null`
- [ ] 5.3 在 RAF 循环中调用 `controls.update()`
- [ ] 5.4 实现 `dispose()`：`controls.dispose()` 并解绑事件

## 任务 6: core 模块 - ModelLoader

- [ ] 6.1 创建 `core/ModelLoader.ts`，封装 `GLTFLoader`
- [ ] 6.2 实现按需加载 `DRACOLoader`：仅在 `dracoDecoderPath` 提供时初始化
- [ ] 6.3 实现 `load(url, onProgress)`，返回 `ModelLoadResult { root, raw, boundingBox }`
- [ ] 6.4 加载失败时包装为 `ModelLoadError` 抛出
- [ ] 6.5 实现 `dispose()`：释放 DRACOLoader 解码器
- [ ] 6.6 编写单测：Mock `GLTFLoader`，验证成功、失败、进度回调路径

## 任务 7: core 模块 - ModelViewer 门面

- [ ] 7.1 创建 `core/ModelViewer.ts`，组合 SceneManager、RendererManager、ControlsManager、ModelLoader
- [ ] 7.2 在构造函数中挂载 canvas 到传入的容器 DOM，启动渲染循环
- [ ] 7.3 实现 `load(url)`：加载 → 卸载旧模型 → 挂载新模型 → 自适应相机 → 触发 `onModelLoaded`
- [ ] 7.4 实现 `unload()`：从 scene 移除 modelRoot，释放 geometry/material/texture，置 `modelRoot = null`
- [ ] 7.5 实现 `fitCameraToModel()`：对当前 modelRoot 重新调用 `SceneManager.fitCameraToObject`
- [ ] 7.6 实现 `onBeforeRender(fn)` 与 `onModelLoaded(fn)`，满足 `ViewerHandle` 接口
- [ ] 7.7 实现 `dispose()`：按顺序销毁 controls、模型资源、renderer、scene，移除 canvas
- [ ] 7.8 编写单测：load → unload → load 循环，验证旧模型释放与新模型挂载正确

## 任务 8: explode 模块 - 类型定义

- [ ] 8.1 创建 `explode/types.ts`，定义 `PlaybackMode`、`ExplodeDirection`、`ExplodeOptions`、`PlayOptions`
- [ ] 8.2 定义 `ExplodePart`、`PlaybackState`、`ExplodeEvent`、`EasingFn` 类型
- [ ] 8.3 在 `explode/index.ts` 中导出对外公开类型与 `ExplodeController` 类

## 任务 9: explode 模块 - 缓动函数

- [ ] 9.1 创建 `explode/easings.ts`，提供 `linear`、`easeInOutCubic`、`easeOutQuad` 等常用缓动
- [ ] 9.2 每个缓动函数满足 `f(0) === 0 && f(1) === 1`
- [ ] 9.3 编写单测：验证端点值与单调性

## 任务 10: explode 模块 - PartAnalyzer

- [ ] 10.1 创建 `explode/PartAnalyzer.ts`，实现 `analyze(modelRoot, options)` 方法
- [ ] 10.2 实现 `groupByParent: true` 策略：收集「直接包含 Mesh 的父 Group 节点」作为爆炸单元
- [ ] 10.3 实现 `groupByParent: false` 策略：收集所有 Mesh 作为独立爆炸单元
- [ ] 10.4 实现 `partFilter` 过滤（仅作用于 Mesh 层级，group 模式下检查 Group 是否包含任一保留 Mesh）
- [ ] 10.5 实现 `computeDirection`：radial / axis-x/y/z / 自定义函数，处理零件与中心重合的退化情况
- [ ] 10.6 计算每个零件的 `worldCenter`、`displacement`、`originalPosition.clone()`
- [ ] 10.7 保证返回数组顺序稳定（深度优先遍历）
- [ ] 10.8 后置断言：所有 `p.originalPosition.equals(p.target.position)`
- [ ] 10.9 编写单测：构造已知几何布局的 Mock Object3D 树，验证零件数量、displacement 方向、过滤效果

## 任务 11: explode 模块 - ExplodeAnimator

- [ ] 11.1 创建 `explode/ExplodeAnimator.ts`，内部维护 `PlaybackState`
- [ ] 11.2 实现 `start(parts, mode, options)`：保存引用、重置状态为 `exploding`
- [ ] 11.3 实现 `tick(deltaMs)` 状态机：处理 `exploding` / `paused-at-exploded` / `collapsing` / `paused-at-collapsed` / `paused` / `idle` 转移
- [ ] 11.4 实现 `applyPositions(parts, t)`：对每个 part 计算 `originalPosition + t × displacement`
- [ ] 11.5 once 模式：进度到达 1 时进入 `idle` 并返回 false
- [ ] 11.6 loop 模式：四阶段循环，永不自行进入 `idle`
- [ ] 11.7 实现 `pause()` / `resume()`：保存/恢复 savedState，冻结时间推进
- [ ] 11.8 实现 `stop()`：进入 idle 并复原所有 part 位置到 originalPosition
- [ ] 11.9 编写单测覆盖每个状态转移分支
- [ ] 11.10* 编写 PBT（fast-check）：进度单调性、有界性、loop 非终止性、position 线性关系

## 任务 12: explode 模块 - ExplodeController 门面

- [ ] 12.1 创建 `explode/ExplodeController.ts`，构造函数接收 `ViewerHandle` 与 `ExplodeOptions`
- [ ] 12.2 合并默认 options；校验 `strength > 0`、`durationMs > 0`、`loopPauseMs >= 0`、`easing` 端点
- [ ] 12.3 启动时若 `modelRoot` 已就绪立即调用 `PartAnalyzer.analyze`；否则订阅 `onModelLoaded` 延迟分析
- [ ] 12.4 订阅 `viewer.onBeforeRender`，每帧调用 `animator.tick(deltaMs)` 并发布 `progress` 事件
- [ ] 12.5 实现 `play(options)`：若 parts 为空立即触发 `complete`；否则调用 `animator.start`，发布 `start`
- [ ] 12.6 实现 `pause` / `resume` / `reset`，并发布对应事件
- [ ] 12.7 实现 `on(fn)` 事件订阅，返回取消订阅函数
- [ ] 12.8 实现「加载前 play()」等待机制：首次标记 pendingPlay，onModelLoaded 后自动开始（仅首次有效）
- [ ] 12.9 实现 `dispose()`：取消所有订阅、清空 parts、释放引用
- [ ] 12.10 编写单测（用 Mock `ViewerHandle`）：验证事件顺序、加载前/后播放、dispose 后无回调

## 任务 13: 模块边界校验

- [ ] 13.1 在 `explode/` 目录下全文搜索确认不存在 `WebGLRenderer`、`OrbitControls`、`GLTFLoader` 的 import
- [ ] 13.2 在 `explode/` 目录下全文搜索确认不存在 `requestAnimationFrame`
- [ ] 13.3 提供 `tests/mock-viewer-handle.ts`，演示 `explode` 模块如何在无 `core` 环境下被单测驱动

## 任务 14: 聚合入口与使用示例

- [ ] 14.1 在 `threejs_explode/index.ts` 中聚合导出 `ModelViewer`、`ExplodeController` 及关键类型
- [ ] 14.2 创建 `examples/basic.html` + `examples/basic.ts`：最小接入示例（加载模型 + 按钮触发单次/循环/复位）
- [ ] 14.3 创建 `examples/events.ts`：事件订阅示例（进度条、按钮状态联动）
- [ ] 14.4 创建 `examples/custom-direction.ts`：自定义方向 + partFilter 示例
- [ ] 14.5 在 `README.md` 中补充快速开始、API 概览、目录结构说明

## 任务 15: 测试策略与覆盖

- [ ] 15.1 汇总单元测试覆盖率目标：核心路径 > 90%
- [ ] 15.2 编写端到端集成测试：使用真实 glTF fixture（`__fixtures__/models/`）完整走通 load → play → pause → reset → dispose
- [ ] 15.3 资源泄漏测试：连续 10 次 `load → dispose`，断言 `renderer.info.memory` 不持续增长
- [ ] 15.4* 集成 fast-check，跑通设计文档中列出的所有 10 条正确性属性
- [ ] 15.5 在 CI 中配置 Vitest 运行与覆盖率报告

## 任务 16: 错误处理与边界场景

- [ ] 16.1 验证 FR-4.1：加载错误 URL 抛出 `ModelLoadError` 且当前模型不变
- [ ] 16.2 验证 FR-4.2：draco 模型未配置解码器时抛明确错误
- [ ] 16.3 验证 FR-4.3：容器 `display: none` 时跳过渲染；恢复后自动重绘
- [ ] 16.4 验证 FR-4.4：加载前创建 controller 并 play，加载后自动开始
- [ ] 16.5 验证 FR-4.5：所有零件被过滤时 play 立即触发 complete
- [ ] 16.6 验证 FR-4.6：WebGL 上下文丢失事件被正确捕获并对外抛出

> 标记 `*` 的子任务为可选增强项（如 PBT 覆盖），可在 MVP 完成后迭代补齐。
