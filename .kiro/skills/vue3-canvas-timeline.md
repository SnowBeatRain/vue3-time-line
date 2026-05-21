# Skill: Vue3 Canvas 时间轴组件开发

## 概述
本 Skill 用于指导 AI 开发基于 Vue 3 + Canvas 2D 的视频监控回放时间轴组件。涵盖核心数学模型、绘制系统、交互系统、多窗口架构和打包配置。

---

## 技术栈约束

- 框架：Vue 3 Composition API (`<script setup>`)
- 渲染：Canvas 2D Context API（不使用 DOM 绘制刻度）
- 时间库：dayjs（轻量，2KB）
- 样式：SCSS scoped + vite-plugin-style-inject（CSS 注入 JS）
- 构建：Vite lib 模式，输出 ESM + UMD + d.ts
- 外部化：vue 作为 peerDependency 不打包

---

## 核心数学模型

### 时间-像素映射（所有计算的基础）

```
totalMS = ZOOM[currentZoomIndex] * 3600000
PX_PER_MS = canvasWidth / totalMS
currentTime = startTimestamp + totalMS / 2

时间戳→像素: x = (timestamp - startTimestamp) * PX_PER_MS
像素→时间戳: timestamp = startTimestamp + x / PX_PER_MS
```

### 刻度对齐算法

```javascript
const msPerGrid = ZOOM_HOUR_GRID[zoomIndex] * ONE_HOUR_STAMP
const msOffset = msPerGrid - (startTimestamp % msPerGrid)
const pxOffset = (msOffset / msPerGrid) * pxPerGrid
// 第i格: x = pxOffset + i * pxPerGrid
// 第i格时间: time = startTimestamp + msOffset + i * msPerGrid
```

### 缩放锚点
缩放时保持中心时间不变：`startTimestamp = currentTime - newTotalMS / 2`

### 拖拽计算
```javascript
newStartTimestamp = mousedownCacheStartTimestamp - Math.round(diffX / PX_PER_MS)
```
向右拖→时间回退，向左拖→时间前进。

---

## ZOOM 分辨率系统

```typescript
const ONE_HOUR_STAMP = 3600000
const ZOOM = [0.5, 1, 2, 6, 12, 24, 72, 360, 720, 8760, 87600]
// 对应：半小时、1h、2h、6h、12h、1天、3天、15天、30天、1年、10年

const ZOOM_HOUR_GRID = [1/60, 1/60, 2/60, 1/6, 0.25, 0.5, 1, 4, 4, 720, 7200]
// 每格代表的小时数

// 格数 = ZOOM[i] / ZOOM_HOUR_GRID[i]
```

每个分辨率有对应的"时间文字显示规则"函数，决定哪些刻度显示文字。

---

## 绘制系统规则

### 绘制顺序（层级）
1. drawTimeSegments() — 最底层
2. addGraduations() — 刻度在时间段上方
3. drawMiddleLine() — 中心线在最上层

### 全量重绘策略
任何状态变化 → `clearRect(0, 0, w, h)` → 重新执行 `draw()`

### 时间段绘制要点
- 坐标四舍五入防间隙：`x = Math.round(x); w = Math.round(w)`
- 最小宽度1px：`w = Math.max(1, w)`
- 左边界裁剪：x < 0 时设 x=0，w 用 endTime-startTimestamp 计算
- 只有 beginTime 无 endTime 时绘制 1px 宽线段

### 时间段点击检测
使用 `ctx.beginPath()` + `ctx.rect()` + `ctx.isPointInPath(x, y)` 实现 hitTest。

---

## 交互系统规则

### 点击/拖拽区分
mousedown 到 mouseup 移动距离 ≤ maxClickDistance(默认3px) → 点击，否则 → 拖拽

### 事件监听策略
- 容器 div：mousedown, mousemove, mouseout, mouseleave, touchstart, touchmove
- Canvas：mousewheel（stop.prevent）
- 全局 window：mouseup/touchend（确保拖出仍能松开）, resize

### 移动端适配
`isMobile=true` 时切换到 touch 事件 + MOBILE_ZOOM_HOUR_GRID（格子更稀疏）

---

## 组件架构

### Props 设计原则
- 时间参数同时接受 Number（时间戳）和 String（日期字符串）
- 函数类型 Props 用于自定义（customShowTime, formatTime, hoverTimeFormat）
- 布尔开关控制功能（enableZoom, enableDrag, showCenterLine, showHoverTime）
- 对象类型 Props 用于样式配置（centerLineStyle, lineHeightRatio）

### Events 设计原则
- timeChange：高频（每帧），用于实时同步
- dragTimeChange：低频（拖拽结束），用于请求数据
- click_timeSegments vs click_timeline：互斥，优先检测时间段

### Expose 设计原则
- setTime：外部控制时间（拖动中忽略）
- setZoom：以当前时间为锚点切换分辨率
- watchTime：观察者模式，追踪时间点像素位置
- reRender：完整重置（nextTick内执行）

---

## 多窗口架构

### 父子组件关系
- 父组件（TimeLine.vue）：管理主Canvas + 时间状态 + 事件
- 子组件（WindowListItem.vue）：独立Canvas，共享 startTimestamp/totalMS/width
- 同步方式：父组件 draw() 时调用 `WindowListItemRef[i].draw()`

### 显示条件
`windowList.length > 1` 时显示多窗口区域，Canvas 高度变为 baseTimeLineHeight(50px)

---

## Vite 打包配置模板

```javascript
// lib 模式关键配置
build: {
  outDir: 'lib',
  lib: { entry: './src/components/index.ts', name: 'xxx', fileName: 'xxx' },
  rollupOptions: {
    external: ['vue'],
    output: { exports: 'named', globals: { vue: 'Vue' } }
  }
}
plugins: [vue(), dts({ rollupTypes: true }), VitePluginStyleInject()]
```

---

## 常见陷阱

| 陷阱 | 解决 |
|------|------|
| Canvas 模糊 | 必须设置 canvas.width/height DOM 属性（非CSS） |
| 时间段间隙 | roundWidthTimeSegments=true |
| mouseup 丢失 | 监听 window.mouseup |
| 拖动中 setTime 冲突 | setTime 开头检查 mousedown 状态 |
| extendZOOM 全局污染 | 直接 push 到模块级数组，多实例会累积 |
| hover 时 timeChange 频率过高 | 业务层 throttle |

---

## AI 开发提示词核心要素

开发类似组件时，提示词中必须包含：
1. 核心公式：PX_PER_MS = width / totalMS
2. 状态核心：startTimestamp 是唯一的位移状态
3. 刻度对齐：msOffset = msPerGrid - (startTimestamp % msPerGrid)
4. 绘制顺序：时间段 → 刻度 → 中心线
5. 交互流程：修改 startTimestamp → clearRect → draw()
6. 缩放锚点：startTimestamp = currentTime - newTotalMS / 2
