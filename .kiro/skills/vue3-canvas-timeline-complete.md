# Skill: Vue3 Canvas 视频回放时间轴组件 — 完整开发规格书

> 本文档是一份 100% 自包含的开发规格书。任何 AI 或开发者仅凭此文档即可从零完整实现一个功能完备的视频监控回放时间轴组件，无需参考任何其他文档或源码。

---

## 第一部分：技术栈与项目结构

### 1.1 技术栈（强制要求）

| 层面 | 技术 | 版本要求 | 用途 |
|------|------|---------|------|
| 框架 | Vue 3 | ≥3.3 | Composition API + `<script setup>` |
| 渲染 | Canvas 2D API | 浏览器原生 | 绘制刻度/时间段/指示线 |
| 时间处理 | dayjs | ≥1.11 | 格式化/比较/解析 |
| 样式 | SCSS (scoped) | — | 组件内样式 |
| 构建 | Vite | ≥4.0 | lib模式打包 |
| 类型 | TypeScript | ≥5.0 | 类型声明生成 |
| 打包插件 | vite-plugin-dts | — | 生成 .d.ts |
| 样式注入 | vite-plugin-style-inject | — | CSS注入JS（无需单独引入CSS） |

### 1.2 项目目录结构

```
project-root/
├── src/
│   └── components/TimeLine/
│       ├── TimeLine.vue            # 主组件（~600行）
│       ├── WindowListItem.vue      # 多窗口子组件（~170行）
│       ├── constant.ts             # 所有常量定义
│       └── index.ts                # Vue 插件入口
├── lib/                            # 构建输出
│   ├── vue3-time-line.mjs          # ESM格式
│   ├── vue3-time-line.umd.js       # UMD格式
│   └── vue3-time-line.d.ts         # 类型声明
├── package.json
├── vite.config.js
├── tsconfig.json
└── tsconfig.app.json
```

### 1.3 Vue 插件入口（index.ts 完整代码）

```typescript
import Vue3TimeLineCom from './TimeLine.vue'

interface IOptions {
  comName?: string
}

interface ITimeLine {
  install(app: any, options?: IOptions): void
}

const Vue3TimeLine: ITimeLine = {
  install(app, options) {
    const comName = options?.comName ? options?.comName : 'TimeLine'
    app.component(comName, Vue3TimeLineCom)
  }
}
export default Vue3TimeLine
```

使用方式：
```typescript
import TimeLine from '@scope/vue3-time-line'
const app = createApp(App)
app.use(TimeLine, { comName: 'TimeLine' })
app.mount('#app')
```

---

## 第二部分：常量系统（constant.ts 完整代码）

```typescript
// ============================================================
// constant.ts — 时间轴所有常量定义
// ============================================================

// 一小时的毫秒数
export const ONE_HOUR_STAMP = 60 * 60 * 1000  // 3,600,000

// 时间分辨率数组：整个时间轴表示的时间范围（单位：小时）
// 索引: 0=半小时, 1=1小时, 2=2小时, 3=6小时, 4=12小时,
//       5=1天(默认), 6=3天, 7=15天, 8=30天, 9=365天(年月模式), 10=3650天(年模式)
export const ZOOM = [0.5, 1, 2, 6, 12, 24, 72, 360, 720, 8760, 87600]

// PC端：每个分辨率对应的每格小时数（即最小刻度格代表多少小时）
// 格数 = ZOOM[i] / ZOOM_HOUR_GRID[i]
export const ZOOM_HOUR_GRID = [
  1 / 60,   // 0: 1分钟/格, 共30格
  1 / 60,   // 1: 1分钟/格, 共60格
  2 / 60,   // 2: 2分钟/格, 共60格
  1 / 6,    // 3: 10分钟/格, 共36格
  0.25,     // 4: 15分钟/格, 共48格
  0.5,      // 5: 30分钟/格, 共48格
  1,        // 6: 1小时/格, 共72格
  4,        // 7: 4小时/格, 共90格
  4,        // 8: 4小时/格, 共180格
  720,      // 9: 30天/格, 共~12格
  7200      // 10: 300天/格, 共~12格
]

// 移动端：每格代表更多小时（格子更稀疏，适应小屏幕）
export const MOBILE_ZOOM_HOUR_GRID = [
  1 / 20,   // 0: 3分钟/格, 共10格
  1 / 30,   // 1: 2分钟/格, 共30格
  1 / 20,   // 2: 3分钟/格, 共40格
  1 / 3,    // 3: 20分钟/格, 共18格
  0.5,      // 4: 30分钟/格, 共24格
  2,        // 5: 2小时/格, 共12格
  4,        // 6: 4小时/格, 共18格
  4,        // 7: 4小时/格, 共90格
  4,        // 8: 4小时/格, 共180格
  720,      // 9: 30天/格
  7200      // 10: 300天/格
]

// PC端：每个分辨率对应的"时间文字是否显示"判断函数
// 参数 date 是当前刻度对应的 Date 对象
// 返回 true 则在该刻度下方显示时间文字
export const ZOOM_DATE_SHOW_RULE = [
  () => true,                                                    // 0: 全部显示
  (date: Date) => date.getMinutes() % 5 === 0,                 // 1: 每5分钟
  (date: Date) => date.getMinutes() % 10 === 0,                // 2: 每10分钟
  (date: Date) => date.getMinutes() === 0 || date.getMinutes() === 30,  // 3: 整点和半点
  (date: Date) => date.getMinutes() === 0,                     // 4: 整点
  (date: Date) => date.getHours() % 2 === 0 && date.getMinutes() === 0,  // 5: 偶数小时
  (date: Date) => date.getHours() % 3 === 0 && date.getMinutes() === 0,  // 6: 每3小时
  (date: Date) => date.getHours() % 12 === 0 && date.getMinutes() === 0, // 7: 每12小时
  () => false,   // 8: 全部不显示（只有0点显示日期）
  () => true,    // 9: 全部显示（年月模式）
  () => true     // 10: 全部显示（年模式）
]

// 移动端规则（更稀疏）
export const MOBILE_ZOOM_DATE_SHOW_RULE = [
  () => true,
  (date: Date) => date.getMinutes() % 5 === 0,
  (date: Date) => date.getMinutes() % 10 === 0,
  (date: Date) => date.getMinutes() === 0 || date.getMinutes() === 30,
  (date: Date) => date.getHours() % 2 === 0 && date.getMinutes() === 0,  // 差异：PC是整点
  (date: Date) => date.getHours() % 4 === 0 && date.getMinutes() === 0,  // 差异：PC是偶数小时
  (date: Date) => date.getHours() % 3 === 0 && date.getMinutes() === 0,
  (date: Date) => date.getHours() % 12 === 0 && date.getMinutes() === 0,
  () => false,
  () => true,
  () => true
]
```

### 2.1 ZOOM 对照表（供快速参考）

| 索引 | ZOOM(h) | 含义 | PC格数 | PC每格 | 移动端格数 | 移动端每格 |
|------|---------|------|--------|--------|-----------|-----------|
| 0 | 0.5 | 半小时 | 30 | 1min | 10 | 3min |
| 1 | 1 | 1小时 | 60 | 1min | 30 | 2min |
| 2 | 2 | 2小时 | 60 | 2min | 40 | 3min |
| 3 | 6 | 6小时 | 36 | 10min | 18 | 20min |
| 4 | 12 | 12小时 | 48 | 15min | 24 | 30min |
| 5 | 24 | 1天(**默认**) | 48 | 30min | 12 | 2h |
| 6 | 72 | 3天 | 72 | 1h | 18 | 4h |
| 7 | 360 | 15天 | 90 | 4h | 90 | 4h |
| 8 | 720 | 30天 | 180 | 4h | 180 | 4h |
| 9 | 8760 | 365天 | ~12 | 30天 | ~12 | 30天 |
| 10 | 87600 | 10年 | ~12 | 300天 | ~12 | 300天 |

---

## 第三部分：核心数学模型

### 3.1 时间-像素双向映射

这是整个组件所有计算的基础。时间轴是一个从 `startTimestamp` 到 `startTimestamp + totalMS` 的线性映射。

```
┌──────────────── Canvas宽度 (width px) ─────────────────┐
│                                                         │
│ startTimestamp        currentTime          endTimestamp  │
│ ◄─────────────── totalMS (毫秒) ──────────────────────► │
│                      ↑ width/2                          │
│                   中心线                                 │
└─────────────────────────────────────────────────────────┘
```

**核心公式**：
```javascript
// 总毫秒数
const totalMS = ZOOM[currentZoomIndex] * ONE_HOUR_STAMP

// 像素/毫秒 转换率
const PX_PER_MS = canvasWidth / totalMS

// 当前时间（中心点）
const currentTime = startTimestamp + totalMS / 2

// 时间戳 → Canvas X坐标
function timeToX(timestamp) {
  return (timestamp - startTimestamp) * PX_PER_MS
}

// Canvas X坐标 → 时间戳
function xToTime(x) {
  return startTimestamp + x / PX_PER_MS
}
```

### 3.2 刻度对齐算法

确保每条刻度线都对齐到"整数时间点"（如整分钟、整小时），而非随机位置。

```javascript
// 每格代表的毫秒数
const msPerGrid = ZOOM_HOUR_GRID[currentZoomIndex] * ONE_HOUR_STAMP

// 每格的像素宽度
const pxPerGrid = canvasWidth / (ZOOM[currentZoomIndex] / ZOOM_HOUR_GRID[currentZoomIndex])

// 关键偏移量：从startTimestamp到下一个"整数时间点"的毫秒距离
const msOffset = msPerGrid - (startTimestamp % msPerGrid)

// 对应的像素偏移
const pxOffset = (msOffset / msPerGrid) * pxPerGrid

// 遍历所有格子
for (let i = 0; i < gridNum; i++) {
  const x = pxOffset + i * pxPerGrid           // 第i格的X坐标
  const time = startTimestamp + msOffset + i * msPerGrid  // 第i格对应的时间
}
```

**原理说明**：
- `startTimestamp` 通常不在整分钟上（如 10:00:37）
- `startTimestamp % msPerGrid` = 距离上一个对齐点的毫秒数
- `msPerGrid - (startTimestamp % msPerGrid)` = 距离下一个对齐点的毫秒数
- 这样第一条刻度线就精确对齐到了"整分钟"

### 3.3 年/月模式特殊修正

年(index=10)和月(index=9)模式下，由于月份天数不固定（28/30/31天），不能简单用等间距，需要额外修正：

```javascript
let adjustMsOffset = 0

if (yearMode) {  // currentZoomIndex === 10
  // 将时间对齐到该年的1月1日 00:00:00
  const yearStr = dayjs(currentGridTime).format('YYYY')
  adjustMsOffset = currentGridTime - new Date(`${yearStr}-01-01 00:00:00`).getTime()
} else if (yearMonthMode) {  // currentZoomIndex === 9
  // 将时间对齐到该月的1日 00:00:00
  const yearStr = dayjs(currentGridTime).format('YYYY')
  const monthStr = dayjs(currentGridTime).format('MM')
  adjustMsOffset = currentGridTime - new Date(`${yearStr}-${monthStr}-01 00:00:00`).getTime()
}

// 修正后的X坐标和时间
const correctedX = x - (adjustMsOffset / msPerGrid) * pxPerGrid
const correctedTime = currentGridTime - adjustMsOffset
```

### 3.4 拖拽位移公式

```javascript
// mousedown时缓存: mousedownX, mousedownCacheStartTimestamp
// mousemove时计算:
const diffX = currentMouseX - mousedownX  // 鼠标移动的像素差
const diffMS = Math.round(diffX / PX_PER_MS)  // 对应的毫秒差

// 关键：向右拖(diffX>0)→看更早的时间→startTimestamp减小
let newStartTimestamp = mousedownCacheStartTimestamp - diffMS

// 边界钳位
const centerTime = newStartTimestamp + totalMS / 2
if (timeRange.start && centerTime < timeRange.start) {
  newStartTimestamp = timeRange.start - totalMS / 2
}
if (timeRange.end && centerTime > timeRange.end) {
  newStartTimestamp = timeRange.end - totalMS / 2
}

startTimestamp = newStartTimestamp
```

### 3.5 缩放锚点公式

缩放时保持中心时间不变（用户看的位置不跳）：

```javascript
// 缩放前记录当前中心时间
const centerTime = startTimestamp + totalMS / 2  // 即 currentTime

// 改变分辨率
currentZoomIndex += delta  // +1缩小，-1放大

// 缩放后重新计算startTimestamp，使centerTime仍在中心
startTimestamp = centerTime - newTotalMS / 2
// 其中 newTotalMS = ZOOM[newZoomIndex] * ONE_HOUR_STAMP
```

---

## 第四部分：内部状态系统

### 4.1 reactive 状态对象（完整字段）

```javascript
const defaultData = reactive({
  // === Canvas 尺寸 ===
  width: 0,                         // Canvas 宽度（像素），init()时从容器获取
  height: 0,                        // Canvas 高度（像素）

  // === 渲染上下文 ===
  ctx: null,                        // CanvasRenderingContext2D 实例

  // === 时间状态（核心！）===
  currentZoomIndex: 0,              // 当前分辨率索引（0~10+扩展）
  currentTime: 0,                   // 当前中心点时间戳（每次draw()后更新）
  startTimestamp: 0,                // 时间轴左端时间戳（所有绘制基于此值！）

  // === 鼠标/触摸交互状态 ===
  mousedown: false,                 // 当前是否按下
  mousedownX: 0,                    // 按下时的X坐标（相对容器左侧）
  mousedownY: 0,                    // 按下时的Y坐标（相对容器顶部）
  mousedownCacheStartTimestamp: 0,  // 按下时缓存的startTimestamp（拖拽基准）
  mousemoveX: -1,                   // 当前鼠标X位置，-1表示不在容器内

  // === 多窗口 ===
  showWindowList: false,            // 是否显示多窗口列表
  windowListInner: [],              // 内部窗口列表（带active字段）

  // === 观察者 ===
  watchTimeList: []                 // watchTime注册的观察列表
})
```

### 4.2 computed 属性

```javascript
// 整个时间轴的总毫秒数
const totalMS = computed(() => ZOOM[defaultData.currentZoomIndex] * ONE_HOUR_STAMP)

// 时间范围转时间戳
const timeRangeTimestamp = computed(() => {
  const t = {}
  if (props.timeRange.start) {
    t.start = typeof props.timeRange.start === 'number'
      ? props.timeRange.start
      : new Date(props.timeRange.start).getTime()
  }
  if (props.timeRange.end) {
    t.end = typeof props.timeRange.end === 'number'
      ? props.timeRange.end
      : new Date(props.timeRange.end).getTime()
  }
  return t
})

// 根据isMobile切换配置
const ACT_ZOOM_HOUR_GRID = computed(() =>
  props.isMobile ? MOBILE_ZOOM_HOUR_GRID : ZOOM_HOUR_GRID
)
const ACT_ZOOM_DATE_SHOW_RULE = computed(() =>
  props.isMobile ? MOBILE_ZOOM_DATE_SHOW_RULE : ZOOM_DATE_SHOW_RULE
)

// 特殊模式
const yearMonthMode = computed(() => defaultData.currentZoomIndex === 9)
const yearMode = computed(() => defaultData.currentZoomIndex === 10)
```

### 4.3 模板 ref

```javascript
const timeLineContainer = ref(null)  // 外层容器div
const canvas = ref(null)              // Canvas元素
const WindowListItemRef = ref([])     // 子窗口组件ref数组（v-for的ref）
```



---

## 第五部分：Props 完整 API（26个属性）

### 5.1 Props 速查表

| # | 属性名 | 类型 | 默认值 | 说明 |
|---|--------|------|--------|------|
| 1 | initTime | Number\|String | '' | 初始中心时间 |
| 2 | timeRange | Object | {} | 时间限制范围 {start,end} |
| 3 | initZoomIndex | Number | 5 | 初始分辨率索引 |
| 4 | showCenterLine | Boolean | true | 是否显示中心竖线 |
| 5 | centerLineStyle | Object | {width:2,color:'#fff'} | 中心线样式 |
| 6 | textColor | String | 'rgba(151,158,167,1)' | 刻度文字颜色 |
| 7 | hoverTextColor | String | 'rgb(194,202,215)' | hover文字颜色 |
| 8 | lineColor | String | 'rgba(151,158,167,1)' | 刻度线颜色 |
| 9 | lineHeightRatio | Object | {date:0.3,time:0.2,none:0.1,hover:0.3} | 各类刻度线高度比 |
| 10 | showHoverTime | Boolean | true | 是否显示hover时间 |
| 11 | hoverTimeFormat | Function | undefined | 自定义hover时间格式 |
| 12 | timeSegments | Array | [] | 时间段数据 |
| 13 | backgroundColor | String | '#262626' | 背景色 |
| 14 | multiSegmentActiveColor | String | undefined | 多窗口选中背景色 |
| 15 | enableZoom | Boolean | true | 是否允许滚轮缩放 |
| 16 | enableDrag | Boolean | true | 是否允许拖拽 |
| 17 | windowList | Array | [] | 多窗口列表 |
| 18 | baseTimeLineHeight | Number | 50 | 多窗口时主轴高度 |
| 19 | initSelectWindowTimeLineIndex | Number | -1 | 初始选中窗口索引 |
| 20 | isMobile | Boolean | false | 是否移动端模式 |
| 21 | maxClickDistance | Number | 3 | 点击/拖拽判定阈值(px) |
| 22 | roundWidthTimeSegments | Boolean | true | 时间段坐标四舍五入 |
| 23 | customShowTime | Function | undefined | 自定义时间显示规则 |
| 24 | showDateAtZero | Boolean | true | 0点是否显示日期 |
| 25 | extendZOOM | Array | [] | 扩展分辨率列表 |
| 26 | formatTime | Function | undefined | 自定义刻度文字格式 |

### 5.2 Props 逐项完整定义

#### Prop 1: initTime
```javascript
initTime: { type: [Number, String], default: '' }
```
**作用**：设置时间轴初始中心点时间。
**接受格式**：
- 数字类型时间戳：`1610640000000`
- 字符串类型：`'2021-01-15 00:00:00'`
- 空值：默认当天 00:00:00

**内部处理逻辑**：
```javascript
const initTimestamp = props.initTime
  ? (typeof props.initTime === 'number'
    ? props.initTime
    : new Date(props.initTime).getTime())
  : new Date(dayjs().format('YYYY-MM-DD 00:00:00')).getTime()

// startTimestamp = 中心时间 - 半个时间范围 = 左端点
defaultData.startTimestamp = initTimestamp - totalMS.value / 2
```

#### Prop 2: timeRange
```javascript
timeRange: { type: Object, default() { return {} } }
```
**格式**：
```javascript
{
  start: '2020-12-19 18:30:00',  // 或时间戳数字
  end: '2021-01-20 10:00:00'     // 或时间戳数字
}
```
**作用**：限制时间轴中心点（currentTime）的可移动范围。
**检查逻辑**（在拖拽和setTime时调用）：
```javascript
const fixStartTimestamp = () => {
  const hfms = totalMS.value / 2
  const ct = defaultData.startTimestamp + hfms  // 中心点
  if (timeRangeTimestamp.value.start && ct < timeRangeTimestamp.value.start) {
    defaultData.startTimestamp = timeRangeTimestamp.value.start - hfms
  }
  if (timeRangeTimestamp.value.end && ct > timeRangeTimestamp.value.end) {
    defaultData.startTimestamp = timeRangeTimestamp.value.end - hfms
  }
}
```

#### Prop 3: initZoomIndex
```javascript
initZoomIndex: { type: Number, default: 5 }
```
**有效范围**：`0` ~ `ZOOM.length - 1`（默认0~10），超出回退到5。
**对应关系**：见第二部分ZOOM对照表。

#### Prop 4: showCenterLine
```javascript
showCenterLine: { type: Boolean, default: true }
```
**false时**：不绘制 Canvas 正中间的竖线。

#### Prop 5: centerLineStyle
```javascript
centerLineStyle: {
  type: Object,
  default() { return { width: 2, color: '#fff' } }
}
```
- `width`：Canvas lineWidth（像素）
- `color`：CSS颜色字符串

#### Prop 6: textColor
```javascript
textColor: { type: String, default: 'rgba(151,158,167,1)' }
```
**影响**：所有刻度文字（包括0点日期和普通时间）的 `ctx.fillStyle`。

#### Prop 7: hoverTextColor
```javascript
hoverTextColor: { type: String, default: 'rgb(194, 202, 215)' }
```
**影响**：鼠标悬停时显示的时间文字颜色。

#### Prop 8: lineColor
```javascript
lineColor: { type: String, default: 'rgba(151,158,167,1)' }
```
**影响**：所有刻度竖线 + hover指示竖线的 `ctx.strokeStyle`。

#### Prop 9: lineHeightRatio
```javascript
lineHeightRatio: {
  type: Object,
  default() {
    return {
      date: 0.3,   // 0点日期刻度线高度 = canvasHeight * 0.3
      time: 0.2,   // 有文字的普通时间刻度线高度 = canvasHeight * 0.2
      none: 0.1,   // 无文字的小刻度线高度 = canvasHeight * 0.1
      hover: 0.3   // hover指示线高度 = canvasHeight * 0.3
    }
  }
}
```
**文字定位**：`ctx.fillText(text, x - 13, lineHeight + 15)`

#### Prop 10: showHoverTime
```javascript
showHoverTime: { type: Boolean, default: true }
```
**false时**：鼠标在时间轴上移动不显示hover效果，只有拖拽功能。

#### Prop 11: hoverTimeFormat
```javascript
hoverTimeFormat: { type: Function }
// 签名: (time: number) => string
```
**参数**：鼠标所在位置对应的时间戳（数字）。
**默认行为**：`dayjs(time).format('YYYY-MM-DD HH:mm:ss')`
**返回空字符串**：不显示文字，但仍显示指示竖线。
**示例**：
```javascript
hoverTimeFormat(time) {
  if (dayjs(time).isBefore(dayjs().startOf('day'))) return ''
  return dayjs(time).format('HH:mm:ss')
}
```

#### Prop 12: timeSegments
```javascript
timeSegments: { type: Array, default: () => [] }
```
**每项数据结构**：
```typescript
interface TimeSegment {
  name?: string        // 自定义名称，点击时返回
  beginTime: number    // 起始时间戳（必填）
  endTime?: number     // 结束时间戳（不填则绘制1px线段）
  color: string        // 填充颜色（必填），如 '#FA3239'
  startRatio?: number  // 纵向起始比例（默认0.6），top = height * startRatio
  endRatio?: number    // 纵向结束比例（默认0.9），bottom = height * endRatio
  [key: string]: any   // 可附加任意自定义字段
}
```
**响应式**：`watch(() => props.timeSegments, reRender, { deep: true })`
**示例**：
```javascript
timeSegments: [
  {
    name: '录像片段1',
    beginTime: new Date('2021-01-13 10:00:00').getTime(),
    endTime: new Date('2021-01-14 23:00:00').getTime(),
    color: '#1a94bc',
    startRatio: 0.65,
    endRatio: 0.9
  }
]
```

#### Prop 13: backgroundColor
```javascript
backgroundColor: { type: String, default: '#262626' }
```
**实现**：通过 `:style="{ backgroundColor }"` 设置到外层div。Canvas本身透明。

#### Prop 14: multiSegmentActiveColor
```javascript
multiSegmentActiveColor: { type: String }
```
**作用**：多窗口时间轴中选中窗口的背景高亮色。传递给 WindowListItem 子组件。

#### Prop 15: enableZoom
```javascript
enableZoom: { type: Boolean, default: true }
```
**false时**：`onMouseweel` 函数第一行 return，滚轮无效。

#### Prop 16: enableDrag
```javascript
enableDrag: { type: Boolean, default: true }
```
**false时**：mousemove 中不执行 `drag()`，mouseup 不 emit `dragTimeChange`。

#### Prop 17: windowList
```javascript
windowList: { type: Array, default: () => [] }
```
**每项结构**：
```typescript
interface WindowItem {
  name: string                    // 窗口名称
  timeSegments?: TimeSegment[]    // 该窗口的时间段
  [key: string]: any              // 任意自定义字段
}
```
**显示条件**：`windowList.length > 1` 时在 Canvas 下方显示多窗口列表。
**副作用**：Canvas 高度变为 `baseTimeLineHeight`，剩余空间给窗口列表。

#### Prop 18: baseTimeLineHeight
```javascript
baseTimeLineHeight: { type: Number, default: 50 }
```
**仅在** `windowList.length > 1` 时生效。

#### Prop 19: initSelectWindowTimeLineIndex
```javascript
initSelectWindowTimeLineIndex: { type: Number, default: -1 }
```
**-1**：不选中任何窗口。**0~N**：初始选中第N个。

#### Prop 20: isMobile
```javascript
isMobile: { type: Boolean, default: false }
```
**true时变化**：
1. 忽略mouse事件，只处理touch事件
2. 使用 MOBILE_ZOOM_HOUR_GRID
3. 使用 MOBILE_ZOOM_DATE_SHOW_RULE
4. 全局监听 touchend 而非 mouseup

#### Prop 21: maxClickDistance
```javascript
maxClickDistance: { type: Number, default: 3 }
```
**判定**：mousedown到mouseup的X、Y移动距离均≤此值 → 点击事件；否则 → 拖拽事件。

#### Prop 22: roundWidthTimeSegments
```javascript
roundWidthTimeSegments: { type: Boolean, default: true }
```
**true时**：`x = Math.round(x); w = Math.round(w)` 防止相邻时间段出现1px间隙。

#### Prop 23: customShowTime
```javascript
customShowTime: { type: Function }
// 签名: (date: Date, currentZoomIndex: number) => boolean | undefined
```
**三值逻辑**：
- 返回 `true`：强制显示该时间文字
- 返回 `false`：强制隐藏
- 返回 `undefined`/其他值：走内置ZOOM_DATE_SHOW_RULE规则

**示例**：
```javascript
customShowTime(date, zoomIndex) {
  if (zoomIndex === 6) {  // 3天模式
    return date.getHours() % 12 === 0 && date.getMinutes() === 0
  }
  // 不返回 → 走内置规则
}
```

#### Prop 24: showDateAtZero
```javascript
showDateAtZero: { type: Boolean, default: true }
```
**true**：0点刻度绘制最长线段 + 显示 `MM-DD` 格式。
**false**：0点和其他时间一样判断是否显示。

#### Prop 25: extendZOOM
```javascript
extendZOOM: { type: Array, default() { return [] } }
```
**每项格式**：
```javascript
{ zoom: 25, zoomHourGrid: 0.5, mobileZoomHourGrid: 2 }
```
**处理时机**：组件创建时立即push到全局ZOOM数组（注意：不可逆！）
**对应索引**：从11开始（0~10是内置）
**⚠️ 必须同时传 customShowTime**：否则内置SHOW_RULE数组越界报错

#### Prop 26: formatTime
```javascript
formatTime: { type: Function }
// 签名: (time: dayjs.Dayjs) => string | '' | undefined
// 注意参数是 dayjs 对象！不是时间戳
```
**返回值逻辑**：
- 返回非空字符串 → 使用该字符串作为刻度文字
- 返回空值/falsy → 走内置规则

**内置规则**：
```javascript
if (yearMode) return time.format('YYYY')
if (yearMonthMode) return time.format('YYYY-MM')
if (hour===0 && minute===0) return time.format('MM-DD')
else return time.format('HH:mm')
```

**示例**（24点显示为24:00）：
```javascript
formatTime(time) {
  if (time.isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) return '24:00'
  if (time.hour() === 0 && time.minute() === 0) return time.format('HH:mm')
}
```

---

## 第六部分：Events 事件系统（8个事件）

### 6.1 事件速查表

| # | 事件名 | 触发时机 | 参数 |
|---|--------|---------|------|
| 1 | timeChange | 每次draw()后 | `(currentTime: number)` |
| 2 | mousedown | 按下时 | `(event: Event)` |
| 3 | mouseup | 松开时 | `(event: Event)` |
| 4 | dragTimeChange | 拖拽结束 | `(currentTime: number)` |
| 5 | click_timeSegments | 点击到时间段 | `(segments: TimeSegment[], time: number, date: string, x: number)` |
| 6 | click_timeline | 点击空白 | `(time: number, date: string, x: number)` |
| 7 | change_window_time_line | 切换窗口 | `(index: number, item: WindowItem)` |
| 8 | click_window_timeSegments | 点击窗口时间段 | `(segments: TimeSegment[], index: number, item: WindowItem)` |

### 6.2 事件详解

#### Event 1: timeChange
```javascript
emits('timeChange', defaultData.currentTime)
```
- **currentTime** = `startTimestamp + totalMS / 2`
- **触发频率**：非常高！每次draw()都触发（拖拽每帧、hover每帧、缩放、setTime）
- **⚠️ 业务层建议throttle**

#### Event 2: mousedown
```javascript
emits('mousedown', e)
```
在 `onPointerdown` 中触发，参数为原始事件对象。

#### Event 3: mouseup
```javascript
emits('mouseup', e)
```
在 `onPointerup` 中触发（无论是点击还是拖拽结束都会触发）。

#### Event 4: dragTimeChange
```javascript
emits('dragTimeChange', defaultData.currentTime)
```
**触发条件**：mousedown=true && enableDrag=true && 移动距离 > maxClickDistance
**典型用途**：拖拽结束后请求对应时间点的视频数据。

#### Event 5: click_timeSegments
```javascript
emits('click_timeSegments', timeSegments, time, date, x)
```
- `timeSegments`：命中的时间段数组（可能多个重叠）
- `time`：点击位置时间戳
- `date`：`dayjs(time).format('YYYY-MM-DD HH:mm:ss')`
- `x`：点击位置相对时间轴左侧的像素
- **触发条件**：点击坐标通过 `ctx.isPointInPath` 命中了某个时间段

#### Event 6: click_timeline
```javascript
emits('click_timeline', time, date, x)
```
**触发条件**：点击位置不在任何时间段上（与click_timeSegments互斥）

#### Event 7: change_window_time_line
```javascript
emits('change_window_time_line', index, windowListInner[index])
```
**触发时机**：点击某个窗口时间轴切换选中状态时。

#### Event 8: click_window_timeSegments
```javascript
emits('click_window_timeSegments', data, index, item)
```
**触发时机**：点击窗口时间轴中的时间段时（由子组件冒泡）。

### 6.3 事件触发流程

```
用户操作 → mousedown → 记录位置 → emit('mousedown')
       ↓
  mousemove → 拖动（drag）/ 显示hover
       ↓
  mouseup → 计算移动距离
       ├── ≤ maxClickDistance → 判定为点击
       │    ├── isPointInPath命中 → emit('click_timeSegments')
       │    └── 未命中 → emit('click_timeline')
       └── > maxClickDistance → 判定为拖拽结束
            └── emit('dragTimeChange')
  最终 → emit('mouseup')
```

---

## 第七部分：Expose 暴露方法（4个）

### 7.1 setTime(t)

```javascript
const setTime = (t) => {
  // 正在拖动时忽略外部setTime（防冲突）
  if (defaultData.mousedown) return

  // 支持时间戳和字符串
  const ts = typeof t === 'number' ? t : new Date(t).getTime()

  // 计算新的startTimestamp使ts成为中心
  defaultData.startTimestamp = ts - totalMS.value / 2

  // 边界修正
  fixStartTimestamp()

  // 清除并重绘
  clearCanvas(defaultData.width, defaultData.height)
  draw()

  // 如果鼠标仍在Canvas上，补绘hover
  if (defaultData.mousemoveX !== -1 && !props.isMobile) {
    hoverShow(defaultData.mousemoveX, true)
  }
}
```

**典型用法**：
```javascript
// 定时器每秒推进
setInterval(() => {
  currentTime += 1000
  timelineRef.value.setTime(currentTime)
}, 1000)

// 跳转到指定时间
timelineRef.value.setTime('2021-01-01 00:00:00')
timelineRef.value.setTime(1609459200000)
```

### 7.2 setZoom(index)

```javascript
const setZoom = (index) => {
  // 有效性校验
  defaultData.currentZoomIndex = (index >= 0 && index < ZOOM.length) ? index : 5

  clearCanvas(defaultData.width, defaultData.height)

  // 以当前中心时间为锚点重新计算左端点
  defaultData.startTimestamp = defaultData.currentTime - totalMS.value / 2

  draw()
}
```

**典型用法**：
```javascript
// select下拉框切换
timelineRef.value.setZoom(6)  // 3天视图
timelineRef.value.setZoom(0)  // 半小时视图（最精确）
```

### 7.3 watchTime(time, callback, windowTimeLineIndex?)

```javascript
const watchTime = (time, callback, windowTimeLineIndex) => {
  if (!time || !callback) return

  defaultData.watchTimeList.push({
    time: typeof time === 'number' ? time : new Date(time).getTime(),
    callback,
    // windowTimeLineIndex从1开始（外部API），内部存储从0开始
    windowTimeLineIndex: typeof windowTimeLineIndex === 'number' ? windowTimeLineIndex - 1 : -1
  })
}
```

**回调签名**：`(x: number, y: number) => void`
- 时间点在可视范围：返回相对浏览器视口的绝对坐标 `(x + canvasLeft, canvasTop)`
- 时间点不在可视范围：返回 `(-1, -1)`
- 如果指定了 windowTimeLineIndex，y 坐标为对应窗口时间轴的top

**更新时机**：每次 `draw()` 后自动调用 `updateWatchTime()`

**典型用法**：
```javascript
// 在时间轴上显示一个浮动图标
timelineRef.value.watchTime('2021-01-01 23:30:00', (x, y) => {
  if (x === -1 || y === -1) {
    icon.style.display = 'none'
  } else {
    icon.style.display = 'block'
    icon.style.left = x + 'px'
    icon.style.top = y + 'px'
  }
})

// 指定显示在第2个窗口时间轴上
timelineRef.value.watchTime('2021-01-02 02:30:00', callback, 2)
```

### 7.4 reRender()

```javascript
const reRender = () => {
  nextTick(() => {
    clearCanvas(defaultData.width, defaultData.height)
    reset()          // 归零所有内部状态
    setInitData()    // 根据当前props重新计算
    init()           // 重新获取容器尺寸
    draw()           // 重新绘制
  })
}
```

**用途**：完全重置组件状态并重新渲染。
**使用场景**：容器尺寸变化、props大幅变更、强制刷新。



---

## 第八部分：绘制系统完整实现

### 8.1 draw() 主绘制函数

这是组件的核心调度函数，所有视觉更新都通过此函数执行。

```javascript
const draw = () => {
  // ★ 绘制顺序决定层级（先画的在底层）
  drawTimeSegments()    // 第1层：时间段色块（底层）
  addGraduations()      // 第2层：刻度线和文字（中层）
  drawMiddleLine()      // 第3层：中心指示线（顶层）

  // 更新当前时间状态
  defaultData.currentTime = defaultData.startTimestamp + totalMS.value / 2
  emits('timeChange', defaultData.currentTime)

  // 通知所有子窗口组件重绘
  try {
    WindowListItemRef.value.forEach((item) => {
      item.draw()
    })
  } catch (error) {
    console.error(error)
  }

  // 更新watchTime观察者位置
  updateWatchTime()
}
```

### 8.2 drawTimeSegments() 时间段绘制（完整实现）

此函数有两种模式：绘制模式（正常渲染）和路径模式（用于hitTest点击检测）。

```javascript
const drawTimeSegments = (callback, path) => {
  const PX_PER_MS = defaultData.width / totalMS.value

  props.timeSegments.forEach((item) => {
    // === 可见性判断 ===
    // 时间段起点在视口右侧之外 → 跳过
    if (item.beginTime > defaultData.startTimestamp + totalMS.value) return

    // 时间段是否有结束时间在视口范围内
    let hasEndTime = item.endTime >= defaultData.startTimestamp

    // === 开始绘制路径 ===
    defaultData.ctx.beginPath()

    // === 计算X坐标和宽度 ===
    let x = (item.beginTime - defaultData.startTimestamp) * PX_PER_MS
    let w

    if (x < 0) {
      // 时间段起点在Canvas左侧之外 → 从0开始绘制
      x = 0
      w = hasEndTime ? (item.endTime - defaultData.startTimestamp) * PX_PER_MS : 1
    } else {
      w = hasEndTime ? (item.endTime - item.beginTime) * PX_PER_MS : 1
    }

    // === 计算Y坐标（纵向位置）===
    let heightStartRatio = item.startRatio === undefined ? 0.6 : item.startRatio
    let heightEndRatio = item.endRatio === undefined ? 0.9 : item.endRatio

    // === 四舍五入防间隙 ===
    if (props.roundWidthTimeSegments) {
      x = Math.round(x)
      w = Math.round(w)
    }

    // === 最小宽度保证 ===
    w = Math.max(1, w)

    // === 绘制或记录路径 ===
    if (path) {
      // 路径模式：仅创建rect路径用于hitTest，不实际填充
      defaultData.ctx.rect(
        x,
        defaultData.height * heightStartRatio,
        w,
        defaultData.height * (heightEndRatio - heightStartRatio)
      )
    } else {
      // 绘制模式：实际填充颜色
      defaultData.ctx.fillStyle = item.color
      defaultData.ctx.fillRect(
        x,
        defaultData.height * heightStartRatio,
        w,
        defaultData.height * (heightEndRatio - heightStartRatio)
      )
    }

    // hitTest时的回调：检测该路径是否被点击
    callback && callback(item)
  })
}
```

### 8.3 addGraduations() 刻度绘制（完整实现）

```javascript
const addGraduations = () => {
  defaultData.ctx.beginPath()

  // === 计算格子参数 ===
  // 总格数
  const gridNum = ZOOM[defaultData.currentZoomIndex] /
    ACT_ZOOM_HOUR_GRID.value[defaultData.currentZoomIndex]
  // 每格毫秒数
  const msPerGrid = ACT_ZOOM_HOUR_GRID.value[defaultData.currentZoomIndex] * ONE_HOUR_STAMP
  // 每格像素宽
  const pxPerGrid = defaultData.width / gridNum
  // 起始偏移（对齐到整数时间点）
  const msOffset = msPerGrid - (defaultData.startTimestamp % msPerGrid)
  const pxOffset = (msOffset / msPerGrid) * pxPerGrid

  // === 遍历所有刻度格 ===
  for (let i = 0; i < gridNum; i++) {
    let currentStartTimestamp = defaultData.startTimestamp + msOffset + i * msPerGrid

    // === 年/月模式特殊修正 ===
    let adjustMsOffset = 0
    if (yearMode.value) {
      adjustMsOffset = currentStartTimestamp -
        new Date(`${dayjs(currentStartTimestamp).format('YYYY')}-01-01 00:00:00`).getTime()
    } else if (yearMonthMode.value) {
      adjustMsOffset = currentStartTimestamp -
        new Date(`${dayjs(currentStartTimestamp).format('YYYY')}-${dayjs(currentStartTimestamp).format('MM')}-01 00:00:00`).getTime()
    }

    // 修正后的X坐标和时间
    let x = pxOffset + i * pxPerGrid - (adjustMsOffset / msPerGrid) * pxPerGrid
    let graduationTime = currentStartTimestamp - adjustMsOffset

    // === 判断刻度类型并绘制 ===
    let h = 0
    let date = new Date(graduationTime)

    if (props.showDateAtZero && date.getHours() === 0 && date.getMinutes() === 0) {
      // ★ 0点：最长线段 + 日期文字
      h = defaultData.height * (props.lineHeightRatio.date === undefined ? 0.3 : props.lineHeightRatio.date)
      defaultData.ctx.fillStyle = props.textColor
      defaultData.ctx.fillText(graduationTitle(graduationTime), x - 13, h + 15)
    } else if (checkShowTime(date)) {
      // ★ 显示时间的刻度：中等线段 + 时间文字
      h = defaultData.height * (props.lineHeightRatio.time === undefined ? 0.2 : props.lineHeightRatio.time)
      defaultData.ctx.fillStyle = props.textColor
      defaultData.ctx.fillText(graduationTitle(graduationTime), x - 13, h + 15)
    } else {
      // ★ 不显示时间的刻度：最短线段
      h = defaultData.height * (props.lineHeightRatio.none === undefined ? 0.1 : props.lineHeightRatio.none)
    }

    // 绘制刻度竖线
    drawLine(x, 0, x, h, 1, props.lineColor)
  }
}
```

### 8.4 drawMiddleLine() 中心线绘制

```javascript
const drawMiddleLine = () => {
  if (!props.showCenterLine) return
  defaultData.ctx.beginPath()
  let { width, color } = props.centerLineStyle
  let x = defaultData.width / 2
  drawLine(x, 0, x, defaultData.height, width, color)
}
```

### 8.5 drawLine() 基础线段工具

```javascript
const drawLine = (x1, y1, x2, y2, lineWidth = 1, color = '#fff') => {
  defaultData.ctx.beginPath()
  defaultData.ctx.strokeStyle = color
  defaultData.ctx.lineWidth = lineWidth
  defaultData.ctx.moveTo(x1, y1)
  defaultData.ctx.lineTo(x2, y2)
  defaultData.ctx.stroke()
}
```

### 8.6 clearCanvas() 清除画布

```javascript
const clearCanvas = (w, h) => {
  defaultData.ctx.clearRect(0, 0, w, h)
}
```

### 8.7 graduationTitle() 刻度文字格式化

```javascript
const graduationTitle = (datetime) => {
  let time = dayjs(datetime)

  // 优先使用用户自定义格式化
  if (props.formatTime) {
    let res = props.formatTime(time)
    if (res) return res  // 非空则使用
  }

  // 内置规则
  if (yearMode.value) {
    return time.format('YYYY')          // "2021"
  } else if (yearMonthMode.value) {
    return time.format('YYYY-MM')       // "2021-01"
  } else if (time.hour() === 0 && time.minute() === 0 && time.millisecond() === 0) {
    return time.format('MM-DD')         // "01-15"
  } else {
    return time.format('HH:mm')         // "14:30"
  }
}
```

### 8.8 checkShowTime() 时间显示判断

```javascript
const checkShowTime = (date) => {
  // 先检查用户自定义规则
  if (props.customShowTime) {
    let res = props.customShowTime(date, defaultData.currentZoomIndex)
    if (res === true) return true    // 强制显示
    if (res === false) return false  // 强制隐藏
    // 其他值（undefined等）→ 继续走内置规则
  }
  // 内置规则
  return ACT_ZOOM_DATE_SHOW_RULE.value[defaultData.currentZoomIndex](date)
}
```

### 8.9 hoverShow() 鼠标悬停时间显示

```javascript
const hoverShow = (x, noDraw) => {
  const PX_PER_MS = defaultData.width / totalMS.value
  let time = defaultData.startTimestamp + x / PX_PER_MS

  // 如果不是补绘模式，先清除重绘底图
  if (!noDraw) {
    clearCanvas(defaultData.width, defaultData.height)
    draw()
  }

  // 绘制hover指示竖线
  let h = defaultData.height *
    (props.lineHeightRatio.hover === undefined ? 0.3 : props.lineHeightRatio.hover)
  drawLine(x, 0, x, h, 1, props.lineColor)

  // 绘制hover时间文字
  defaultData.ctx.fillStyle = props.hoverTextColor
  let t = props.hoverTimeFormat
    ? props.hoverTimeFormat(time)
    : dayjs(time).format('YYYY-MM-DD HH:mm:ss')
  let w = defaultData.ctx.measureText(t).width
  defaultData.ctx.fillText(t, x - w / 2, h + 20)  // 文字居中于指示线下方
}
```

### 8.10 updateWatchTime() 观察者位置更新

```javascript
const updateWatchTime = () => {
  defaultData.watchTimeList.forEach((item) => {
    // 判断时间点是否在可视范围内
    if (item.time < defaultData.startTimestamp ||
        item.time > defaultData.startTimestamp + totalMS.value) {
      // 不在范围内 → 返回(-1,-1)
      item.callback(-1, -1)
    } else {
      // 在范围内 → 计算像素位置
      let x = (item.time - defaultData.startTimestamp) * (defaultData.width / totalMS.value)
      let y = 0
      let { left, top } = canvas.value.getBoundingClientRect()

      // 如果指定了窗口时间轴索引
      if (item.windowTimeLineIndex !== -1 &&
          props.windowList.length > 1 &&
          item.windowTimeLineIndex >= 0 &&
          item.windowTimeLineIndex < props.windowList.length) {
        let rect = WindowListItemRef.value[item.windowTimeLineIndex].getRect()
        y = rect ? rect.top : top
      } else {
        y = top
      }

      // 返回相对视口的绝对坐标
      item.callback(x + left, y)
    }
  })
}
```

---

## 第九部分：交互系统完整实现

### 9.1 事件绑定策略总览

```
┌─ 容器div ─────────────────────────────┐
│  @touchstart="onTouchstart"            │  ← 触摸开始
│  @touchmove="onTouchmove"              │  ← 触摸移动
│  @mousedown="onMousedown"              │  ← 鼠标按下
│  @mouseout="onMouseout"                │  ← 鼠标移出（清除hover）
│  @mousemove="onMousemove"              │  ← 鼠标移动
│  @mouseleave="onMouseleave"            │  ← 鼠标离开
│                                        │
│  ┌─ Canvas ────────────────────────┐   │
│  │  @mousewheel.stop.prevent       │   │  ← 滚轮缩放
│  └─────────────────────────────────┘   │
└────────────────────────────────────────┘

全局 window 监听：
  - mouseup / touchend  ← 确保拖出Canvas仍能松开
  - resize              ← 容器尺寸变化
```

### 9.2 坐标转换工具

```javascript
const getClientOffset = (e) => {
  if (!timeLineContainer.value || !e) return [0, 0]
  let { left, top } = timeLineContainer.value.getBoundingClientRect()
  return [e.clientX - left, e.clientY - top]
}
```

### 9.3 PC端鼠标事件完整实现

#### mousedown → onMousedown → onPointerdown
```javascript
const onMousedown = (e) => {
  if (props.isMobile) return  // 移动端忽略鼠标
  onPointerdown(e)
}

const onPointerdown = (e) => {
  let pos = getClientOffset(e)
  e.target.style.cursor = 'grabbing'
  defaultData.mousedownX = pos[0]
  defaultData.mousedownY = pos[1]
  defaultData.mousedown = true
  defaultData.mousedownCacheStartTimestamp = defaultData.startTimestamp
  emits('mousedown', e)
}
```

#### mousemove → onMousemove → onPointermove
```javascript
const onMousemove = (e) => {
  if (props.isMobile) return
  onPointermove(e)
}

const onPointermove = (e) => {
  let x = getClientOffset(e)[0]
  defaultData.mousemoveX = x

  if (defaultData.mousedown && props.enableDrag) {
    drag(x)       // 按下状态 → 拖拽
  } else if (props.showHoverTime) {
    hoverShow(x)  // 非按下状态 → hover显示
  }
}
```

#### mouseup（window全局）→ onMouseup → onPointerup
```javascript
let onMouseup = (e) => {
  if (props.isMobile) return
  onPointerup(e)
}

const onPointerup = (e) => {
  e.target.style.cursor = 'pointer'
  let pos = getClientOffset(e)

  const reset = () => {
    defaultData.mousedown = false
    defaultData.mousedownX = 0
    defaultData.mousedownY = 0
    defaultData.mousedownCacheStartTimestamp = 0
  }

  // 判定点击还是拖拽
  if (Math.abs(pos[0] - defaultData.mousedownX) <= props.maxClickDistance &&
      Math.abs(pos[1] - defaultData.mousedownY) <= props.maxClickDistance) {
    reset()
    onClick(...pos)  // → 点击事件
    return
  }

  if (defaultData.mousedown && props.enableDrag) {
    reset()
    emits('dragTimeChange', defaultData.currentTime)  // → 拖拽结束
  } else {
    reset()
  }
  emits('mouseup', e)
}
```

#### mouseout → onMouseout（清除hover效果）
```javascript
const onMouseout = () => {
  clearCanvas(defaultData.width, defaultData.height)
  draw()
}
```

#### mouseleave → onMouseleave
```javascript
const onMouseleave = () => {
  defaultData.mousemoveX = -1  // 标记鼠标离开
}
```

### 9.4 拖拽实现 drag()

```javascript
const drag = (x) => {
  if (!props.enableDrag) return

  const PX_PER_MS = defaultData.width / totalMS.value
  let diffX = x - defaultData.mousedownX

  // 计算新的startTimestamp
  let hfms = totalMS.value / 2
  let _newStartTimestamp = defaultData.mousedownCacheStartTimestamp - Math.round(diffX / PX_PER_MS)

  // 边界钳位
  let ct = _newStartTimestamp + hfms
  if (timeRangeTimestamp.value.start && ct < timeRangeTimestamp.value.start) {
    _newStartTimestamp = timeRangeTimestamp.value.start - hfms
  }
  if (timeRangeTimestamp.value.end && ct > timeRangeTimestamp.value.end) {
    _newStartTimestamp = timeRangeTimestamp.value.end - hfms
  }

  defaultData.startTimestamp = _newStartTimestamp
  clearCanvas(defaultData.width, defaultData.height)
  draw()
}
```

### 9.5 滚轮缩放 onMouseweel()

```javascript
const onMouseweel = (event) => {
  if (!props.enableZoom) return

  let e = window.event || event
  let delta = Math.max(-1, Math.min(1, e.wheelDelta || -e.detail))

  if (delta < 0) {
    // 向下滚 → 缩小（增大时间范围）
    if (defaultData.currentZoomIndex + 1 >= ZOOM.length - 1) {
      defaultData.currentZoomIndex = ZOOM.length - 1
    } else {
      defaultData.currentZoomIndex++
    }
  } else if (delta > 0) {
    // 向上滚 → 放大（缩小时间范围）
    if (defaultData.currentZoomIndex - 1 <= 0) {
      defaultData.currentZoomIndex = 0
    } else {
      defaultData.currentZoomIndex--
    }
  }

  clearCanvas(defaultData.width, defaultData.height)
  // ★ 以当前中心时间为锚点
  defaultData.startTimestamp = defaultData.currentTime - totalMS.value / 2
  draw()
}
```

### 9.6 点击事件处理 onClick()

```javascript
const onClick = (x, y) => {
  const PX_PER_MS = defaultData.width / totalMS.value
  let time = defaultData.startTimestamp + x / PX_PER_MS
  let date = dayjs(time).format('YYYY-MM-DD HH:mm:ss')

  // 检测是否命中时间段
  let timeSegments = getClickTimeSegments(x, y)
  if (timeSegments && timeSegments.length > 0) {
    emits('click_timeSegments', timeSegments, time, date, x)
  } else {
    emits('click_timeline', time, date, x)
  }
}
```

### 9.7 hitTest 点击检测 getClickTimeSegments()

```javascript
const getClickTimeSegments = (x, y) => {
  let inItems = []

  // 以路径模式重新"绘制"所有时间段，逐个检测
  drawTimeSegments((item) => {
    if (defaultData.ctx.isPointInPath(x, y)) {
      inItems.push(item)
    }
  }, true)  // path=true → ctx.rect而非ctx.fillRect

  return inItems
}
```

**原理说明**：
1. 调用 `drawTimeSegments(callback, path=true)`
2. 内部对每个时间段调用 `ctx.beginPath()` + `ctx.rect()`（不填充）
3. 紧接着调用 `callback(item)`，在callback中执行 `ctx.isPointInPath(x, y)`
4. 因为每个时间段都有独立的 `beginPath()`，所以 isPointInPath 只检测当前路径
5. 命中则 push 到结果数组

### 9.8 移动端触摸事件

```javascript
// touchstart
const onTouchstart = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]    // 取第一个触摸点
  onPointerdown(e)    // 复用通用逻辑
}

// touchmove
const onTouchmove = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]
  onPointermove(e)
}

// touchend（全局window监听）
let onTouchend = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]    // 注意：touchend时touches可能为空
  onPointerup(e)
}
```

### 9.9 窗口切换 toggleActive()

```javascript
const toggleActive = (index) => {
  // 先全部取消选中
  defaultData.windowListInner.forEach((item) => {
    item.active = false
  })
  // 设置当前选中
  defaultData.windowListInner[index].active = true
  emits('change_window_time_line', index, defaultData.windowListInner[index])
}
```



---

## 第十部分：多窗口子组件 WindowListItem 完整实现

### 10.1 组件职责

每个 WindowListItem 代表一个"播放窗口"的独立时间轴，拥有自己的 Canvas，绘制该窗口的时间段。所有窗口共享父组件的 `startTimestamp` 和 `totalMS`，保证时间同步。

### 10.2 WindowListItem.vue 完整代码

```vue
<template>
  <div class="windowListItem" :class="{ active: active }" ref="windowListItem" @click="onClick">
    <span class="order">{{ index + 1 }}</span>
    <canvas class="windowListItemCanvas" ref="canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, nextTick } from 'vue'
defineOptions({ name: 'WindowListItem' })

const props = defineProps({
  index: { type: Number },
  data: { type: Object, default() { return {} } },
  totalMS: { type: Number },
  startTimestamp: { type: Number },
  width: { type: Number },
  active: { type: Boolean, default: false },
  multiSegmentActiveColor: { type: String, default: '#333' }
})

const emits = defineEmits(['click', 'click_window_timeSegments'])
const windowListItem = ref(null)
const canvas = ref(null)
const defaultData = reactive({
  height: 0,
  ctx: null
})

// === 初始化 ===
const init = () => {
  let { height } = windowListItem.value.getBoundingClientRect()
  defaultData.height = height - 1  // -1 给 border-bottom 留空间
  canvas.value.width = props.width
  canvas.value.height = defaultData.height
  defaultData.ctx = canvas.value.getContext('2d')
}

// === 绘制时间段 ===
const drawTimeSegments = (callback, path) => {
  if (!props.data.timeSegments || props.data.timeSegments.length <= 0) return

  const PX_PER_MS = props.width / props.totalMS

  props.data.timeSegments.forEach((item) => {
    if (item.beginTime <= props.startTimestamp + props.totalMS &&
        item.endTime >= props.startTimestamp) {
      defaultData.ctx.beginPath()
      let x = (item.beginTime - props.startTimestamp) * PX_PER_MS
      let w
      if (x < 0) {
        x = 0
        w = (item.endTime - props.startTimestamp) * PX_PER_MS
      } else {
        w = (item.endTime - item.beginTime) * PX_PER_MS
      }

      let heightStartRatio = item.startRatio === undefined ? 0.6 : item.startRatio
      let heightEndRatio = item.endRatio === undefined ? 0.9 : item.endRatio

      if (path) {
        defaultData.ctx.rect(x, defaultData.height * heightStartRatio, w,
          defaultData.height * (heightEndRatio - heightStartRatio))
      } else {
        defaultData.ctx.fillStyle = item.color
        defaultData.ctx.fillRect(x, defaultData.height * heightStartRatio, w,
          defaultData.height * (heightEndRatio - heightStartRatio))
      }
      callback && callback(item)
    }
  })
}

// === 清除画布 ===
const clearCanvas = () => {
  defaultData.ctx.clearRect(0, 0, props.width, defaultData.height)
}

// === 绘制（由父组件调用） ===
const draw = () => {
  nextTick(() => {
    clearCanvas()
    drawTimeSegments()
  })
}

// === 点击事件 ===
const onClick = (e) => {
  emits('click', e)
  let { left, top } = windowListItem.value.getBoundingClientRect()
  let x = e.clientX - left
  let y = e.clientY - top
  let timeSegments = getClickTimeSegments(x, y)
  if (timeSegments.length > 0) {
    emits('click_window_timeSegments', timeSegments, props.index, props.data)
  }
}

// === hitTest ===
const getClickTimeSegments = (x, y) => {
  if (!props.data.timeSegments || props.data.timeSegments.length <= 0) return []
  let inItems = []
  drawTimeSegments((item) => {
    if (defaultData.ctx.isPointInPath(x, y)) {
      inItems.push(item)
    }
  }, true)
  return inItems
}

// === 获取位置信息（供watchTime使用） ===
const getRect = () => {
  return windowListItem.value ? windowListItem.value.getBoundingClientRect() : null
}

onMounted(() => {
  init()
  drawTimeSegments()
})

defineExpose({ draw, init, getRect })
</script>

<style lang="scss" scoped>
.windowListItem {
  width: 100%;
  height: 30px;
  position: relative;
  border-bottom: 1px solid rgba(153, 153, 153, 1);
  user-select: none;

  &.active {
    background-color: v-bind(multiSegmentActiveColor);
  }

  .order {
    position: absolute;
    width: 30px;
    height: 30px;
    display: flex;
    justify-content: center;
    align-items: center;
    color: #fff;
    border-right: 1px solid rgba(153, 153, 153, 1);
  }

  .windowListItemCanvas {
    width: 100%;
    height: 100%;
  }
}
</style>
```

### 10.3 父子组件数据流

```
TimeLine.vue（父）
  │
  ├─ Props 下发给每个 WindowListItem：
  │   • totalMS        ← 共享时间范围
  │   • startTimestamp ← 共享起始时间（保证同步）
  │   • width          ← 共享Canvas宽度
  │   • data           ← 该窗口的数据（含 timeSegments）
  │   • index          ← 窗口索引
  │   • active         ← 是否选中
  │   • multiSegmentActiveColor ← 选中背景色
  │
  ├─ Ref 调用：
  │   • WindowListItemRef[i].draw()   ← 每次父draw()后调用
  │   • WindowListItemRef[i].init()   ← resize时调用
  │   • WindowListItemRef[i].getRect() ← watchTime定位时调用
  │
  └─ Events 冒泡：
      • @click → toggleActive(index) → 切换选中
      • @click_window_timeSegments → emit('click_window_timeSegments')
```

---

## 第十一部分：生命周期完整实现

### 11.1 onMounted（组件初始化）

```javascript
onMounted(() => {
  setInitData()   // 1. 计算初始时间、分辨率、窗口列表
  init()          // 2. 获取容器尺寸、创建Canvas context
  draw()          // 3. 首次绘制

  // 4. 绑定全局事件
  onMouseup = onMouseup.bind(this)
  onResize = onResize.bind(this)
  onTouchend = onTouchend.bind(this)

  if (props.isMobile) {
    window.addEventListener('touchend', onTouchend)
  } else {
    window.addEventListener('mouseup', onMouseup)
  }
  window.addEventListener('resize', onResize)
})
```

### 11.2 onBeforeUnmount（清理）

```javascript
onBeforeUnmount(() => {
  if (props.isMobile) {
    window.removeEventListener('touchend', onTouchend)
  } else {
    window.removeEventListener('mouseup', onMouseup)
  }
  window.removeEventListener('resize', onResize)
})
```

### 11.3 setInitData()（初始数据计算）

```javascript
const setInitData = () => {
  // 1. 构建内部窗口列表（添加active字段）
  defaultData.windowListInner = props.windowList.map((item, index) => ({
    ...item,
    active: props.initSelectWindowTimeLineIndex === index
  }))

  // 2. 设置初始分辨率
  defaultData.currentZoomIndex =
    (props.initZoomIndex >= 0 && props.initZoomIndex < ZOOM.length)
      ? props.initZoomIndex : 5

  // 3. 计算初始startTimestamp
  defaultData.startTimestamp =
    (props.initTime
      ? (typeof props.initTime === 'number'
        ? props.initTime
        : new Date(props.initTime).getTime())
      : new Date(dayjs().format('YYYY-MM-DD 00:00:00')).getTime())
    - totalMS.value / 2

  // 4. 边界修正
  fixStartTimestamp()
}
```

### 11.4 init()（Canvas初始化）

```javascript
const init = () => {
  let { width, height } = timeLineContainer.value.getBoundingClientRect()
  defaultData.width = width
  // 有多窗口时Canvas高度固定，否则撑满容器
  defaultData.height = props.windowList.length > 1 ? props.baseTimeLineHeight : height

  // 必须设置DOM属性（非CSS！否则Canvas会模糊）
  canvas.value.width = defaultData.width
  canvas.value.height = defaultData.height

  defaultData.ctx = canvas.value.getContext('2d')
  defaultData.showWindowList = true
}
```

### 11.5 reset()（状态归零）

```javascript
const reset = () => {
  defaultData.width = 0
  defaultData.height = 0
  defaultData.ctx = null
  defaultData.currentZoomIndex = 0
  defaultData.currentTime = 0
  defaultData.startTimestamp = 0
  defaultData.mousedown = false
  defaultData.mousedownX = 0
  defaultData.mousedownCacheStartTimestamp = 0
}
```

### 11.6 onResize（响应式适配）

```javascript
let onResize = () => {
  init()   // 重新获取尺寸
  draw()   // 重新绘制
  try {
    WindowListItemRef.value.forEach((item) => item.init())
  } catch (error) {
    console.error(error)
  }
}
```

### 11.7 watch 监听

```javascript
// timeSegments 数据变化时完整重渲染
watch(() => props.timeSegments, reRender, { deep: true })
```

### 11.8 extendZOOM 处理（组件创建时执行）

```javascript
// 在 <script setup> 顶层，组件创建时立即执行
props.extendZOOM.forEach((item) => {
  ZOOM.push(item.zoom)
  ZOOM_HOUR_GRID.push(item.zoomHourGrid)
  MOBILE_ZOOM_HOUR_GRID.push(item.mobileZoomHourGrid)
})
```

⚠️ **注意**：这会修改模块级数组（全局共享）。多实例场景下会累积push。

---

## 第十二部分：模板结构

### 12.1 TimeLine.vue 模板

```html
<template>
  <div
    class="timeLineContainer"
    ref="timeLineContainer"
    :style="{ backgroundColor: backgroundColor }"
    @touchstart="onTouchstart"
    @touchmove="onTouchmove"
    @mousedown="onMousedown"
    @mouseout="onMouseout"
    @mousemove="onMousemove"
    @mouseleave="onMouseleave"
  >
    <canvas
      class="canvas"
      ref="canvas"
      @mousewheel.stop.prevent="onMouseweel"
    ></canvas>

    <div
      class="windowList"
      ref="windowList"
      v-if="defaultData.showWindowList && windowList && windowList.length > 1"
      @scroll="onWindowListScroll"
    >
      <WindowListItemCom
        v-for="(item, index) in defaultData.windowListInner"
        ref="WindowListItemRef"
        :key="index"
        :index="index"
        :data="item"
        :totalMS="totalMS"
        :startTimestamp="defaultData.startTimestamp"
        :width="defaultData.width"
        :active="item.active"
        :multiSegmentActiveColor="multiSegmentActiveColor"
        @click_window_timeSegments="triggerClickWindowTimeSegments"
        @click="toggleActive(index)"
      ></WindowListItemCom>
    </div>
  </div>
</template>
```

---

## 第十三部分：样式系统（SCSS）

### 13.1 主组件样式

```scss
<style lang="scss" scoped>
.timeLineContainer {
  width: 100%;
  height: 100%;
  cursor: pointer;
  display: flex;
  flex-direction: column;

  .canvas {
    flex-grow: 0;
    flex-shrink: 0;  // Canvas 不被压缩，保持设定高度
  }

  .windowList {
    width: 100%;
    height: 100%;          // 填满Canvas下方剩余空间
    overflow: auto;
    overflow-x: hidden;
    border-top: 1px solid rgba(153, 153, 153, 1);
    display: flex;
    flex-direction: column;

    &::-webkit-scrollbar {
      display: none;       // 隐藏滚动条
    }
  }
}
</style>
```

### 13.2 布局说明

```
┌─────────────────────── 容器 (flex column, 100%) ────────────────────────┐
│                                                                          │
│  ┌── Canvas (flex-shrink:0, 高度=baseTimeLineHeight或容器全高) ────────┐ │
│  │  绘制区：刻度 + 时间段 + 中心线 + hover                             │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌── windowList (flex-grow:1, overflow-y:auto) ─────────────────────────┐ │
│  │  WindowListItem 1  (30px 高)                                         │ │
│  │  WindowListItem 2  (30px 高)                                         │ │
│  │  WindowListItem 3  (30px 高)                                         │ │
│  │  ...                                                                  │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

- **单窗口模式**：Canvas高度=容器全高，无windowList
- **多窗口模式**：Canvas高度=baseTimeLineHeight(50px)，下方是可滚动的窗口列表

---

## 第十四部分：Vite 打包配置完整代码

### 14.1 vite.config.js

```javascript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import dts from 'vite-plugin-dts'
import VitePluginStyleInject from 'vite-plugin-style-inject'
import { resolve } from 'path'

export default defineConfig(({ command, mode }) => {
  const config = {
    plugins: [
      vue(),
      dts({
        tsconfigPath: 'tsconfig.app.json',
        rollupTypes: true,
        copyDtsFiles: true
      }),
      vueJsx()
    ],
    build: {
      outDir: 'lib',
      rollupOptions: {
        external: ['vue'],
        output: {
          exports: 'named',
          globals: { vue: 'Vue' }
        }
      },
      lib: {
        entry: resolve(__dirname, './src/components/TimeLine/index.ts'),
        name: 'vue3-time-line',
        fileName: 'vue3-time-line'
      }
    },
    resolve: {
      alias: {
        '@': fileURLToPath(new URL('./src', import.meta.url))
      }
    }
  }

  if (command === 'build') {
    if (mode === 'lib') {
      config.plugins.push(VitePluginStyleInject())
    } else if (mode === 'demo') {
      delete config.build.lib
      config.base = './'
      config.build.outDir = 'dist'
      config.build.rollupOptions = {
        output: {
          entryFileNames: '[name].js',
          chunkFileNames: '[name].js',
          assetFileNames: '[name].[ext]'
        }
      }
    }
  }
  return config
})
```

### 14.2 package.json 关键配置

```json
{
  "name": "@scope/vue3-time-line",
  "version": "1.0.0",
  "main": "./lib/vue3-time-line.umd.js",
  "module": "./lib/vue3-time-line.mjs",
  "types": "./lib/vue3-time-line.d.ts",
  "exports": {
    ".": {
      "import": "./lib/vue3-time-line.mjs",
      "require": "./lib/vue3-time-line.umd.js"
    }
  },
  "files": ["lib"],
  "peerDependencies": { "vue": "^3.3.0" },
  "dependencies": { "dayjs": "^1.11.10" },
  "scripts": {
    "dev": "vite",
    "build": "vite build --mode lib",
    "build:demo": "vite build --mode demo"
  }
}
```

### 14.3 打包设计决策说明

| 决策 | 做法 | 原因 |
|------|------|------|
| Vue外部化 | `external: ['vue']` | 使用者项目已有Vue，不重复打包 |
| dayjs打包进来 | 不在external中 | 确保独立可用 |
| CSS注入JS | `VitePluginStyleInject` | 用户不需要单独import CSS |
| 类型合并 | `dts({ rollupTypes: true })` | 输出单个.d.ts文件 |
| 双格式 | ESM(.mjs) + UMD(.umd.js) | 兼容import和require |
| 双模式构建 | lib模式(组件) / demo模式(演示站) | 一套配置两种用途 |



---

## 第十五部分：完整使用示例

### 15.1 基础用法（最简）

```vue
<template>
  <div style="height: 50px">
    <TimeLine ref="TimelineRef" @timeChange="timeChange" />
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'
import dayjs from 'dayjs'

const TimelineRef = ref(null)
const defaultData = reactive({ time: Date.now(), timer: null })
const showTime = computed(() => dayjs(defaultData.time).format('YYYY-MM-DD HH:mm:ss'))

const timeChange = (t) => { defaultData.time = t }

onMounted(() => {
  defaultData.timer = setInterval(() => {
    defaultData.time += 1000
    TimelineRef.value.setTime(defaultData.time)
  }, 1000)
})
onBeforeUnmount(() => clearInterval(defaultData.timer))
</script>
```

### 15.2 显示时间段 + 点击事件

```vue
<template>
  <div style="height: 50px">
    <TimeLine
      ref="Timeline2"
      :initTime="'2021-01-15 00:00:00'"
      :timeSegments="timeSegments"
      @timeChange="timeChange"
      @click_timeSegments="onClickSegment"
      @click_timeline="onClickTimeLine"
      @dragTimeChange="onDragEnd"
    />
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'

const Timeline2 = ref(null)
const timeSegments = reactive([
  {
    name: '录像1',
    beginTime: new Date('2021-01-13 10:00:00').getTime(),
    endTime: new Date('2021-01-14 23:00:00').getTime(),
    color: '#1a94bc',
    startRatio: 0.65,
    endRatio: 0.9
  },
  {
    name: '录像2',
    beginTime: new Date('2021-01-15 02:00:00').getTime(),
    endTime: new Date('2021-01-15 18:00:00').getTime(),
    color: '#1a94bc',
    startRatio: 0.65,
    endRatio: 0.9
  }
])

const timeChange = (t) => { /* 实时时间 */ }
const onClickSegment = (arr, time, date, x) => {
  console.log('点击了时间段:', arr[0].name, '时间:', date)
}
const onClickTimeLine = (time, date, x) => {
  console.log('点击了空白:', date)
}
const onDragEnd = (time) => {
  console.log('拖拽结束, 当前时间:', time)
}
</script>
```

### 15.3 多窗口时间轴

```vue
<template>
  <div style="height: 200px">
    <TimeLine
      :initTime="'2021-01-15 00:00:00'"
      :timeSegments="mainSegments"
      :windowList="windowList"
      @click_window_timeSegments="onWindowClick"
      @change_window_time_line="onWindowChange"
    />
  </div>
</template>

<script setup>
const mainSegments = [
  { beginTime: new Date('2021-01-13 10:00:00').getTime(),
    endTime: new Date('2021-01-14 23:00:00').getTime(),
    color: '#FA3239', startRatio: 0.65, endRatio: 0.9 }
]

const windowList = [
  {
    name: '窗口1',
    timeSegments: [
      { name: '片段A', beginTime: new Date('2021-01-13 10:00:00').getTime(),
        endTime: new Date('2021-01-14 23:00:00').getTime(),
        color: '#FA3239', startRatio: 0.1, endRatio: 0.9 }
    ]
  },
  {
    name: '窗口2',
    timeSegments: [
      { name: '片段B', beginTime: new Date('2021-01-15 02:00:00').getTime(),
        endTime: new Date('2021-01-15 18:00:00').getTime(),
        color: '#FFCC00', startRatio: 0.1, endRatio: 0.9 }
    ]
  },
  { name: '窗口3' },
  { name: '窗口4' }
]

const onWindowClick = (segments, index, item) => {
  console.log('窗口时间段点击:', segments[0].name, '窗口:', index)
}
const onWindowChange = (index, item) => {
  console.log('切换到窗口:', index, item.name)
}
</script>
```

### 15.4 自定义元素定位 (watchTime)

```vue
<template>
  <div style="height: 200px; position: relative;">
    <TimeLine ref="TL" :initTime="'2021-01-02 00:00:00'" :windowList="windows" />
    <i class="marker" ref="marker" style="position:fixed; font-size:24px; color:red;">📍</i>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const TL = ref(null)
const marker = ref(null)
const windows = [{ name: '1' }, { name: '2' }, { name: '3' }]

onMounted(() => {
  TL.value.watchTime('2021-01-01 23:30:00', (x, y) => {
    if (x === -1 || y === -1) {
      marker.value.style.display = 'none'
    } else {
      marker.value.style.display = 'block'
      marker.value.style.left = x + 'px'
      marker.value.style.top = y + 'px'
    }
  })
})
</script>
```

### 15.5 年/年月模式

```vue
<!-- 年模式：10年跨度 -->
<TimeLine :enableZoom="false" :initZoomIndex="10" :timeSegments="segments" />

<!-- 年月模式：365天跨度 -->
<TimeLine :enableZoom="false" :initZoomIndex="9" :timeSegments="segments" />
```

### 15.6 自定义分辨率 (extendZOOM)

```vue
<template>
  <div style="height: 50px">
    <TimeLine
      :enableZoom="false"
      :enableDrag="false"
      :showDateAtZero="false"
      :initZoomIndex="11"
      :initTime="dayjs().format('YYYY-MM-DD 12:00:00')"
      :customShowTime="customShowTime"
      :extendZOOM="[{ zoom: 25, zoomHourGrid: 0.5 }]"
      :formatTime="formatTime"
      :hoverTimeFormat="hoverTimeFormat"
    />
  </div>
</template>

<script setup>
import dayjs from 'dayjs'

const customShowTime = (date, zoomIndex) => {
  if (zoomIndex === 11) {
    return date.getHours() % 2 === 0 && date.getMinutes() === 0
  }
}

const formatTime = (time) => {
  if (time.isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) return '24:00'
  if (time.hour() === 0 && time.minute() === 0) return time.format('HH:mm')
}

const hoverTimeFormat = (time) => {
  if (dayjs(time).isBefore(dayjs().startOf('day')) ||
      dayjs(time).isAfter(dayjs().endOf('day'))) return ''
  return dayjs(time).format('HH:mm:ss')
}
</script>
```

### 15.7 编程式控制

```javascript
// 跳转到指定时间
timelineRef.value.setTime('2021-01-01 00:00:00')
timelineRef.value.setTime(1609459200000)

// 切换分辨率
timelineRef.value.setZoom(0)   // 最精确：半小时
timelineRef.value.setZoom(10)  // 最宏观：10年

// 强制重渲染
timelineRef.value.reRender()
```



---

## 第十六部分：AI 开发提示词模板

### 16.1 基础版（快速开发）

```
请开发一个基于 Vue 3 + Canvas 2D 的视频回放时间轴组件：
- 技术栈：Vue 3 script setup + Canvas 2D + dayjs + SCSS
- 核心状态：startTimestamp（左端时间戳），所有绘制基于此值
- 公式：totalMS = ZOOM[zoomIndex] * 3600000; PX_PER_MS = width / totalMS
- 刻度对齐：msOffset = msPerGrid - (startTimestamp % msPerGrid)
- 绘制顺序：时间段fillRect → 刻度线stroke → 中心线stroke
- 拖拽：newStart = cacheStart - round(diffX / PX_PER_MS)
- 缩放锚点：startTimestamp = currentTime - newTotalMS / 2
- hitTest：ctx.beginPath + ctx.rect + ctx.isPointInPath
- 每次交互：修改startTimestamp → clearRect → draw()
```

### 16.2 完整版（还原全部功能）

```
请开发一个完整的视频监控回放时间轴 Vue 3 组件库，要求如下：

## 文件结构
src/components/TimeLine/
├── TimeLine.vue (主组件，~600行)
├── WindowListItem.vue (子窗口组件)
├── constant.ts (ZOOM/GRID/RULE数组)
└── index.ts (Vue插件入口)

## 常量定义
- ONE_HOUR_STAMP = 3600000
- ZOOM = [0.5, 1, 2, 6, 12, 24, 72, 360, 720, 8760, 87600]
- ZOOM_HOUR_GRID = [1/60, 1/60, 2/60, 1/6, 0.25, 0.5, 1, 4, 4, 720, 7200]
- 每个ZOOM对应一个显示规则函数

## 26个Props
initTime, timeRange, initZoomIndex(默认5), showCenterLine, centerLineStyle,
textColor, hoverTextColor, lineColor, lineHeightRatio{date,time,none,hover},
showHoverTime, hoverTimeFormat, timeSegments, backgroundColor, multiSegmentActiveColor,
enableZoom, enableDrag, windowList, baseTimeLineHeight(50), initSelectWindowTimeLineIndex,
isMobile, maxClickDistance(3), roundWidthTimeSegments(true), customShowTime,
showDateAtZero, extendZOOM, formatTime

## 8个Events
timeChange, mousedown, mouseup, dragTimeChange, click_timeSegments,
click_timeline, change_window_time_line, click_window_timeSegments

## 4个Expose
setTime(t), setZoom(index), watchTime(time,cb,windowIndex?), reRender()

## 核心算法
1. totalMS = ZOOM[i] * 3600000
2. PX_PER_MS = width / totalMS
3. 刻度对齐: msOffset = msPerGrid - (startTimestamp % msPerGrid)
4. 拖拽: newStart = cacheStart - round(diffX / PX_PER_MS)
5. 缩放锚点: start = currentTime - newTotalMS / 2
6. hitTest: beginPath + rect + isPointInPath
7. 四舍五入: Math.round(x), Math.max(1, w)
8. 年月修正: adjustMsOffset对齐到年初/月初

## 打包
Vite lib模式, external vue, ESM+UMD+d.ts, style-inject
```

### 16.3 增量功能提示词

#### 只加时间段功能
```
在现有Canvas时间轴上添加时间段绘制：
数据格式: { beginTime, endTime, color, startRatio(0.6), endRatio(0.9) }
绘制: ctx.fillRect, 坐标Math.round防间隙, 最小1px宽
左裁剪: x<0时x=0, w用endTime-startTimestamp计算
hitTest: ctx.rect + isPointInPath (path模式不填充)
watch deep监听变化时reRender
```

#### 只加多窗口功能
```
添加多窗口时间轴：
条件: windowList.length > 1 时显示
布局: 主Canvas(50px) + 下方windowList(可滚动,每个30px)
子组件: 独立Canvas, 共享startTimestamp/totalMS/width
同步: 父draw()时调用每个子组件.draw()
事件: click冒泡切换active, click_window_timeSegments冒泡转发
```

---

## 第十七部分：开发注意事项与陷阱

### 17.1 性能陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| hover频繁全量重绘 | 每次mousemove都clearRect+draw | 可优化为双Canvas |
| 大量timeSegments卡顿 | forEach遍历所有段 | 增加二分查找可视范围 |
| timeChange事件洪水 | hover也触发draw→emit | 业务层throttle |
| resize抖动 | 连续触发init+draw | debounce处理 |

### 17.2 常见Bug

| 陷阱 | 原因 | 解决 |
|------|------|------|
| Canvas模糊 | 只设了CSS宽高，未设DOM属性 | `canvas.width = px`（非style） |
| 时间段间隙 | 浮点数亚像素 | `roundWidthTimeSegments=true` |
| mouseup丢失 | 鼠标滑出Canvas | 监听`window.mouseup` |
| 拖动中setTime跳动 | 定时器和拖动冲突 | setTime开头检查mousedown |
| extendZOOM累积 | push到模块级数组 | 多实例会重复push |
| touchend无touches | 触摸结束后touches为空 | 用changedTouches |
| 0点刻度不对齐 | 年/月模式天数不固定 | adjustMsOffset修正 |

### 17.3 架构改进建议

| 方面 | 当前 | 建议 |
|------|------|------|
| 绘制 | 全量重绘 | 双Canvas(静态+动态) |
| 状态 | 单reactive | 拆分为多个composable |
| 检索 | 全量遍历 | 区间树/二分查找 |
| 事件 | mouse+touch分开 | 统一PointerEvent |
| 类型 | JS混用 | 完整TypeScript |
| 常量扩展 | 全局数组push | 实例级副本 |

---

## 第十八部分：TypeScript 完整类型定义

```typescript
// ============================================================
// types.ts — 完整类型定义（可直接用于新项目）
// ============================================================

export interface TimeSegment {
  name?: string
  beginTime: number
  endTime?: number
  color: string
  startRatio?: number   // 默认 0.6
  endRatio?: number     // 默认 0.9
  [key: string]: any
}

export interface WindowItem {
  name: string
  timeSegments?: TimeSegment[]
  [key: string]: any
}

export interface TimeRange {
  start?: number | string
  end?: number | string
}

export interface CenterLineStyle {
  width?: number    // 默认 2
  color?: string    // 默认 '#fff'
}

export interface LineHeightRatio {
  date?: number     // 默认 0.3
  time?: number     // 默认 0.2
  none?: number     // 默认 0.1
  hover?: number    // 默认 0.3
}

export interface ExtendZoomItem {
  zoom: number
  zoomHourGrid: number
  mobileZoomHourGrid?: number
}

export interface TimeLineProps {
  initTime?: number | string
  timeRange?: TimeRange
  initZoomIndex?: number
  showCenterLine?: boolean
  centerLineStyle?: CenterLineStyle
  textColor?: string
  hoverTextColor?: string
  lineColor?: string
  lineHeightRatio?: LineHeightRatio
  showHoverTime?: boolean
  hoverTimeFormat?: (time: number) => string
  timeSegments?: TimeSegment[]
  backgroundColor?: string
  multiSegmentActiveColor?: string
  enableZoom?: boolean
  enableDrag?: boolean
  windowList?: WindowItem[]
  baseTimeLineHeight?: number
  initSelectWindowTimeLineIndex?: number
  isMobile?: boolean
  maxClickDistance?: number
  roundWidthTimeSegments?: boolean
  customShowTime?: (date: Date, currentZoomIndex: number) => boolean | undefined
  showDateAtZero?: boolean
  extendZOOM?: ExtendZoomItem[]
  formatTime?: (time: import('dayjs').Dayjs) => string | undefined
}

export interface TimeLineExpose {
  setTime: (t: number | string) => void
  setZoom: (index: number) => void
  watchTime: (
    time: number | string,
    callback: (x: number, y: number) => void,
    windowTimeLineIndex?: number
  ) => void
  reRender: () => void
}

export interface TimeLineEmits {
  timeChange: (currentTime: number) => void
  mousedown: (event: Event) => void
  mouseup: (event: Event) => void
  dragTimeChange: (currentTime: number) => void
  click_timeSegments: (
    segments: TimeSegment[],
    time: number,
    date: string,
    x: number
  ) => void
  click_timeline: (time: number, date: string, x: number) => void
  change_window_time_line: (index: number, item: WindowItem) => void
  click_window_timeSegments: (
    segments: TimeSegment[],
    index: number,
    item: WindowItem
  ) => void
}
```

---

## 第十九部分：数据流图与函数索引

### 19.1 整体数据流

```
┌─────────── 用户交互 ───────────┐
│ mousedown / touchstart          │
│ mousemove / touchmove           │
│ mouseup / touchend              │
│ mousewheel                      │
└─────────┬───────────────────────┘
          ↓
┌─────────── 状态更新 ───────────┐
│ startTimestamp (★核心)          │
│ currentZoomIndex                │
│ mousedown / mousedownX          │
└─────────┬───────────────────────┘
          ↓
┌─────────── 重绘流程 ───────────┐
│ clearCanvas()                   │
│ draw()                          │
│   ├── drawTimeSegments()        │
│   ├── addGraduations()          │
│   ├── drawMiddleLine()          │
│   ├── emit('timeChange')        │
│   ├── WindowListItem[].draw()   │
│   └── updateWatchTime()         │
└─────────────────────────────────┘
          ↓
┌─────────── 输出 ──────────────┐
│ Canvas 像素渲染                 │
│ 事件回调 (timeChange等)         │
│ watchTime 位置回调              │
└─────────────────────────────────┘
```

### 19.2 TimeLine.vue 函数索引

| 函数 | 职责 | 调用时机 |
|------|------|---------|
| setInitData | 计算初始时间/分辨率/窗口列表 | onMounted |
| fixStartTimestamp | 边界钳位检查 | setInitData/drag/setTime |
| init | 获取容器尺寸,创建context | onMounted/onResize/reRender |
| draw | 主绘制调度 | 任何状态变化后 |
| drawTimeSegments | 绘制/检测时间段 | draw内/hitTest |
| addGraduations | 绘制刻度线和文字 | draw内 |
| drawMiddleLine | 绘制中心指示线 | draw内 |
| drawLine | 画线工具 | 各绘制函数内 |
| clearCanvas | 清空画布 | 每次重绘前 |
| graduationTitle | 格式化刻度文字 | addGraduations内 |
| checkShowTime | 判断是否显示时间文字 | addGraduations内 |
| hoverShow | 绘制hover效果 | onPointermove/setTime |
| updateWatchTime | 更新观察者坐标 | draw内 |
| onPointerdown | 按下统一处理 | onMousedown/onTouchstart |
| onPointermove | 移动统一处理 | onMousemove/onTouchmove |
| onPointerup | 松开统一处理 | onMouseup/onTouchend |
| drag | 拖拽逻辑 | onPointermove内 |
| onMouseweel | 滚轮缩放 | Canvas mousewheel |
| onClick | 点击分发 | onPointerup内 |
| getClickTimeSegments | hitTest检测 | onClick内 |
| getClientOffset | 坐标转换 | 所有事件处理 |
| setTime | [Expose] 设置时间 | 外部调用 |
| setZoom | [Expose] 设置分辨率 | 外部调用 |
| watchTime | [Expose] 注册观察 | 外部调用 |
| reRender | [Expose] 完整重渲染 | 外部调用/watch |
| reset | 状态归零 | reRender内 |
| toggleActive | 切换窗口选中 | WindowListItem click |
| onWindowListScroll | 窗口滚动 | windowList scroll |
| onResize | 尺寸重适应 | window resize |

---

## 第二十部分：典型应用场景配置表

| 场景 | 关键配置 |
|------|---------|
| 24小时监控回放 | initZoomIndex=5, initTime=当天0点, timeSegments=录像段 |
| 多通道NVR回放 | windowList=[各通道], timeSegments=总录像段 |
| 只读展示（不可拖拽） | enableDrag=false, enableZoom=false |
| 移动端使用 | isMobile=true |
| 只显示当天 | extendZOOM=[{zoom:25}], enableDrag=false, initZoomIndex=11 |
| 长期数据概览 | initZoomIndex=9(年月)/10(年), enableZoom=false |
| 精确到秒的操作 | initZoomIndex=0(半小时), 配合setTime每秒推进 |
| 自定义时间范围限制 | timeRange={start:'...', end:'...'} |
| 时间段点击跳转 | @click_timeSegments="handler" |
| 时间轴点击定位 | @click_timeline="handler" |

---

*文档完毕。本文档共覆盖：4个文件完整代码、26个Props、8个Events、4个Expose方法、19个内部函数实现、11级分辨率系统、5个核心数学公式、7个完整示例、3套AI提示词模板、12个常见陷阱、完整TypeScript类型定义。*
