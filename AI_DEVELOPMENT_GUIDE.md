# Vue3 Timeline 全方位深度分析与 AI 开发核心文档

> 本文档是对 `@boyzcf/vue3-time-line` 项目的完整逆向工程分析，可直接作为 AI 开发类似时间轴组件的技术规格书。

---

## 目录

1. [项目总览](#第一部分项目总览)
2. [技术栈深度分析](#第二部分技术栈深度分析)
3. [组件插件注册系统](#第三部分组件插件注册系统)
4. [常量系统完整解析](#第四部分常量系统完整解析)
5. [主组件 Props 完整定义](#第五部分主组件-props-完整定义26个属性)
6. [Events 事件系统](#第六部分events-事件系统8个事件)
7. [Expose 方法系统](#第七部分expose-暴露方法4个方法)
8. [内部状态管理](#第八部分内部状态管理)
9. [核心数学模型](#第九部分核心数学模型)
10. [绘制系统完整流程](#第十部分绘制系统完整流程)
11. [交互系统完整流程](#第十一部分交互系统完整流程)
12. [多窗口子组件 WindowListItem](#第十二部分多窗口子组件-windowlistitem)
13. [生命周期与响应式](#第十三部分生命周期与响应式)
14. [样式系统](#第十四部分样式系统)
15. [Demo 示例全集](#第十五部分demo-示例全集)
16. [AI 开发完整提示词模板](#第十六部分ai-开发完整提示词模板)
17. [开发注意事项与陷阱](#第十七部分开发注意事项与陷阱)

---


## 第一部分：项目总览

### 1.1 项目定位

这是一个**基于 Vue 3 + Canvas 2D 的视频监控回放时间轴组件**，发布到 npm 上供其他项目使用。典型应用场景是安防监控、视频回放系统中的时间选择与时间段可视化。

### 1.2 目录结构详解

```
vue3-time-line/
├── lib/                          # 构建产物（发布到 npm 的内容）
│   ├── vue3-time-line.mjs        # ESM 格式产物
│   ├── vue3-time-line.umd.js     # UMD 格式产物
│   └── vue3-time-line.d.ts       # TypeScript 类型声明
├── src/
│   ├── components/TimeLine/      # 核心组件源码
│   │   ├── TimeLine.vue          # 主时间轴组件（~600行）
│   │   ├── WindowListItem.vue    # 多窗口子时间轴组件
│   │   ├── constant.ts           # 常量配置（ZOOM/规则）
│   │   ├── index.ts              # Vue 插件入口
│   │   └── package.json          # 组件级 package.json
│   ├── views/                    # 示例页面
│   │   ├── Base.vue              # 基础用法
│   │   ├── Segment.vue           # 时间段示例
│   │   ├── MultiSegment.vue      # 多轴示例
│   │   ├── Custom.vue            # 自定义元素示例
│   │   ├── Year.vue              # 年级别示例
│   │   ├── YearMonth.vue         # 年月级别示例
│   │   └── CustomZoom.vue        # 自定义分辨率示例
│   ├── App.vue                   # Demo 入口（radio切换示例）
│   └── main.ts                   # 应用入口
├── package.json                  # 项目配置
├── vite.config.js                # 构建配置（双模式：lib/demo）
├── tsconfig.json                 # TS 配置
└── index.html                    # HTML 入口
```

### 1.3 功能清单

| # | 功能 | 说明 |
|---|------|------|
| 1 | Canvas时间刻度绘制 | 根据分辨率动态绘制刻度线和时间文本 |
| 2 | 时间段可视化 | 用彩色矩形表示录像片段 |
| 3 | 鼠标拖拽平移 | 左右拖动改变当前时间 |
| 4 | 滚轮缩放分辨率 | 11级缩放（半小时~10年） |
| 5 | 时间范围限制 | 限定时间轴可拖动的边界 |
| 6 | 鼠标悬停时间显示 | hover时在鼠标位置显示精确时间 |
| 7 | 时间段点击检测 | 精确判断点击是否命中某个时间段 |
| 8 | 多窗口时间轴 | 一个主轴+N个子轴，对应多个播放窗口 |
| 9 | 窗口时间轴选中切换 | 点击切换active状态 |
| 10 | 自定义元素定位(watchTime) | 追踪时间点的实时像素坐标 |
| 11 | 移动端触摸支持 | touchstart/touchmove/touchend |
| 12 | 可扩展分辨率 | extendZOOM追加自定义分辨率 |
| 13 | 自定义时间格式化 | formatTime/hoverTimeFormat |
| 14 | 自定义显示规则 | customShowTime控制哪些时间显示文字 |
| 15 | 响应式适配 | window.resize时重新初始化Canvas |
| 16 | 编程式控制 | setTime/setZoom/reRender/watchTime |

---


## 第二部分：技术栈深度分析

### 2.1 运行时依赖

| 依赖 | 版本 | 作用 | 选型理由 |
|------|------|------|----------|
| `vue` | ^3.3.4 | 组件框架 | 目标就是 Vue3 组件库 |
| `dayjs` | ^1.11.10 | 时间格式化/计算 | 轻量（2KB gzip），API兼容moment |

### 2.2 开发依赖

| 依赖 | 作用 |
|------|------|
| `vite` ^4.4.9 | 构建工具，支持 lib 模式打包 |
| `@vitejs/plugin-vue` | Vue SFC 编译 |
| `@vitejs/plugin-vue-jsx` | JSX 支持 |
| `vite-plugin-dts` ^4.5.3 | 从 TS/Vue 源码生成 .d.ts 类型声明 |
| `vite-plugin-style-inject` | 将 CSS 注入 JS 文件（库模式不输出独立CSS） |
| `typescript` ^5.8.3 | 类型检查 |
| `sass` ^1.69.3 | SCSS 编译 |
| `eslint` + `prettier` | 代码规范 |
| `vitest` | 单元测试框架 |

### 2.3 核心技术选型理由

| 决策 | 理由 |
|------|------|
| **Canvas 而非 DOM** | 时间轴刻度数量可达数百个，DOM节点过多性能差；Canvas单帧绘制效率高 |
| **全量重绘策略** | 每次交互 clearRect + 重绘全部，代码简单，对小Canvas(50px高)性能足够 |
| **dayjs 而非 Date** | 格式化、比较、解析更方便，体积极小 |
| **Vue 插件模式** | 全局注册一次即可使用，符合 Vue 组件库发布规范 |
| **CSS-in-JS (style-inject)** | 库模式打包不需要用户额外引入 CSS 文件 |
| **Composition API** | 更好的逻辑复用和TypeScript支持 |

### 2.4 Vite 构建配置详解

```javascript
// vite.config.js 完整逻辑解析
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
        rollupTypes: true,      // 合并所有类型声明到单个文件
        copyDtsFiles: true      // 确保复制所有关联的 .d.ts 文件
      }),
      vueJsx()
    ],
    build: {
      outDir: 'lib',
      rollupOptions: {
        external: ['vue'],      // vue 不打包进来（peerDependency）
        output: {
          exports: 'named',     // 命名导出
          globals: { vue: 'Vue' }  // UMD 模式全局变量映射
        }
      },
      lib: {
        entry: resolve(__dirname, './src/components/TimeLine/index.ts'),
        name: 'vue3-time-line',
        fileName: 'vue3-time-line'
        // 自动输出: vue3-time-line.mjs (ESM) + vue3-time-line.umd.js (UMD)
      }
    },
    resolve: {
      alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) }
    }
  }

  if (command === 'build') {
    if (mode === 'lib') {
      // `npm run build` → 库模式，打包组件供npm发布
      config.plugins.push(VitePluginStyleInject())  // CSS注入到JS中
    } else if (mode === 'demo') {
      // `npm run build:demo` → 演示站点模式
      delete config.build.lib       // 删除lib配置变成普通SPA
      config.base = './'            // 相对路径（GitHub Pages友好）
      config.build.outDir = 'dist'  // 输出到dist
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

**关键设计决策**：
1. `external: ['vue']` — vue 作为 peerDependency，不打包进库
2. `VitePluginStyleInject` — 只在 lib 模式使用，将 SCSS 编译后的 CSS 注入 JS
3. 双模式构建 — 同一个 vite.config 通过 mode 参数支持库打包和 demo 打包
4. `dts({ rollupTypes: true })` — 将多个 .d.ts 合并为单个文件

---


## 第三部分：组件插件注册系统

### 3.1 插件入口 index.ts

```typescript
import Vue3TimeLineCom from './TimeLine.vue'

interface IOptions {
  comName?: string  // 允许自定义全局组件名称
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

### 3.2 类型声明文件 (vue3-time-line.d.ts)

```typescript
declare interface IOptions {
    comName?: string;
}
declare interface ITimeLine {
    install(app: any, options?: IOptions): void;
}
declare const Vue3TimeLine: ITimeLine;
export default Vue3TimeLine;
```

### 3.3 使用方式

```typescript
// 方式1：全局注册
import TimeLine from '@boyzcf/vue3-time-line'
app.use(TimeLine, { comName: 'TimeLine' })

// 方式2：自定义组件名
app.use(TimeLine, { comName: 'VideoTimeLine' })

// 模板中使用
// <TimeLine @timeChange="..." :timeSegments="..." />
```

### 3.4 package.json exports 配置

```json
{
  "main": "./lib/vue3-time-line.umd.js",
  "module": "./lib/vue3-time-line.mjs",
  "types": "./lib/vue3-time-line.d.ts",
  "exports": {
    ".": {
      "import": "./lib/vue3-time-line.mjs",
      "require": "./lib/vue3-time-line.umd.js"
    }
  },
  "files": ["lib"]
}
```

---

## 第四部分：常量系统完整解析

### 4.1 文件：constant.ts

```typescript
// 一小时的毫秒数
export const ONE_HOUR_STAMP = 60 * 60 * 1000  // 3,600,000

// 时间分辨率数组：整个时间轴表示的时间范围（单位：小时）
export const ZOOM = [0.5, 1, 2, 6, 12, 24, 72, 360, 720, 8760, 87600]

// PC端：每个分辨率对应的最小格代表多少小时
export const ZOOM_HOUR_GRID = [1/60, 1/60, 2/60, 1/6, 0.25, 0.5, 1, 4, 4, 720, 7200]

// 移动端：每格代表更多小时（格子更稀疏）
export const MOBILE_ZOOM_HOUR_GRID = [1/20, 1/30, 1/20, 1/3, 0.5, 2, 4, 4, 4, 720, 7200]

// PC端：时间文字显示规则函数数组
export const ZOOM_DATE_SHOW_RULE = [...]

// 移动端：时间文字显示规则函数数组
export const MOBILE_ZOOM_DATE_SHOW_RULE = [...]
```

### 4.2 ZOOM 时间分辨率完整对照表

| 索引 | ZOOM(小时) | 含义 | 实际跨度 | PC格数 | 移动端格数 | PC每格 | 移动端每格 |
|------|-----------|------|---------|--------|-----------|--------|-----------|
| 0 | 0.5 | 半小时 | 30分钟 | 30格 | 10格 | 1分钟 | 3分钟 |
| 1 | 1 | 1小时 | 60分钟 | 60格 | 30格 | 1分钟 | 2分钟 |
| 2 | 2 | 2小时 | 120分钟 | 60格 | 40格 | 2分钟 | 3分钟 |
| 3 | 6 | 6小时 | 360分钟 | 36格 | 18格 | 10分钟 | 20分钟 |
| 4 | 12 | 12小时 | 720分钟 | 48格 | 24格 | 15分钟 | 30分钟 |
| 5 | 24 | 1天(默认) | 1440分钟 | 48格 | 12格 | 30分钟 | 2小时 |
| 6 | 72 | 3天 | 4320分钟 | 72格 | 18格 | 1小时 | 4小时 |
| 7 | 360 | 15天 | 21600分钟 | 90格 | 90格 | 4小时 | 4小时 |
| 8 | 720 | 30天 | 43200分钟 | 180格 | 180格 | 4小时 | 4小时 |
| 9 | 8760 | 365天 | 年月模式 | 12.2格 | 12.2格 | 30天 | 30天 |
| 10 | 87600 | 3650天 | 年模式 | 12.2格 | 12.2格 | 300天 | 300天 |

**格数公式**：`gridNum = ZOOM[i] / ZOOM_HOUR_GRID[i]`

### 4.3 ZOOM_DATE_SHOW_RULE 显示规则详解

```typescript
export const ZOOM_DATE_SHOW_RULE = [
  // index 0 (半小时): 全部显示时间文字
  () => true,

  // index 1 (1小时): 只在分钟是5的倍数时显示
  (date: Date) => date.getMinutes() % 5 === 0,

  // index 2 (2小时): 只在分钟是10的倍数时显示
  (date: Date) => date.getMinutes() % 10 === 0,

  // index 3 (6小时): 整点和半点显示
  (date: Date) => date.getMinutes() === 0 || date.getMinutes() === 30,

  // index 4 (12小时): 只在整点显示
  (date: Date) => date.getMinutes() === 0,

  // index 5 (1天): 偶数小时的整点显示
  (date: Date) => date.getHours() % 2 === 0 && date.getMinutes() === 0,

  // index 6 (3天): 每3小时的整点显示
  (date: Date) => date.getHours() % 3 === 0 && date.getMinutes() === 0,

  // index 7 (15天): 每12小时显示（0点和12点）
  (date: Date) => date.getHours() % 12 === 0 && date.getMinutes() === 0,

  // index 8 (30天): 全部不显示（只靠0点的日期线段）
  () => false,

  // index 9 (365天/年月模式): 全部显示
  () => true,

  // index 10 (3650天/年模式): 全部显示
  () => true
]
```

### 4.4 移动端规则差异

| 索引 | PC规则 | 移动端规则 |
|------|--------|-----------|
| 4 | 整点显示 | 偶数小时显示 |
| 5 | 偶数小时 | 每4小时 |
| 其余 | 相同 | 相同 |

---


## 第五部分：主组件 Props 完整定义（26个属性）

### Props 总览

| # | 属性名 | 类型 | 默认值 | 版本 |
|---|--------|------|--------|------|
| 1 | initTime | Number/String | '' | - |
| 2 | timeRange | Object | {} | - |
| 3 | initZoomIndex | Number | 5 | - |
| 4 | showCenterLine | Boolean | true | - |
| 5 | centerLineStyle | Object | {width:2,color:'#fff'} | - |
| 6 | textColor | String | 'rgba(151,158,167,1)' | - |
| 7 | hoverTextColor | String | 'rgb(194,202,215)' | - |
| 8 | lineColor | String | 'rgba(151,158,167,1)' | - |
| 9 | lineHeightRatio | Object | {date:0.3,time:0.2,none:0.1,hover:0.3} | - |
| 10 | showHoverTime | Boolean | true | - |
| 11 | hoverTimeFormat | Function | undefined | v0.1.9+ |
| 12 | timeSegments | Array | [] | - |
| 13 | backgroundColor | String | '#262626' | - |
| 14 | multiSegmentActiveColor | String | undefined | - |
| 15 | enableZoom | Boolean | true | - |
| 16 | enableDrag | Boolean | true | - |
| 17 | windowList | Array | [] | - |
| 18 | baseTimeLineHeight | Number | 50 | - |
| 19 | initSelectWindowTimeLineIndex | Number | -1 | - |
| 20 | isMobile | Boolean | false | - |
| 21 | maxClickDistance | Number | 3 | v0.1.2+ |
| 22 | roundWidthTimeSegments | Boolean | true | v0.1.6+ |
| 23 | customShowTime | Function | undefined | v0.1.7+ |
| 24 | showDateAtZero | Boolean | true | v0.1.9+ |
| 25 | extendZOOM | Array | [] | v0.1.9+ |
| 26 | formatTime | Function | undefined | v0.1.9+ |

### 各属性详解

#### 1. initTime
```javascript
initTime: { type: [Number, String], default: '' }
```
- **用途**：设置时间轴初始中心点时间
- **默认行为**：空值 → 当天 00:00:00
- **接受格式**：时间戳 `1610640000000` 或字符串 `'2021-01-15 00:00:00'`
- **内部处理**：
  ```javascript
  const initTimestamp = props.initTime
    ? (typeof props.initTime === 'number' ? props.initTime : new Date(props.initTime).getTime())
    : new Date(dayjs().format('YYYY-MM-DD 00:00:00')).getTime()
  // 中心点 - 半个时间范围 = 时间轴左端点
  startTimestamp = initTimestamp - totalMS / 2
  ```

#### 2. timeRange
```javascript
timeRange: { type: Object, default() { return {} } }
// 格式: { start: '2020-12-19 18:30:00', end: '2021-01-20 10:00:00' }
// 或:   { start: 1608391800000, end: 1611133200000 }
```
- **用途**：限制时间轴中心点的可移动范围
- **限制逻辑**（在拖动和初始化时检查）：
  ```javascript
  const fixStartTimestamp = () => {
    let hfms = totalMS / 2
    let ct = startTimestamp + hfms  // 中心点时间
    if (timeRange.start && ct < timeRange.start) {
      startTimestamp = timeRange.start - hfms  // 钳位到左边界
    }
    if (timeRange.end && ct > timeRange.end) {
      startTimestamp = timeRange.end - hfms    // 钳位到右边界
    }
  }
  ```

#### 3. initZoomIndex
```javascript
initZoomIndex: { type: Number, default: 5 }
```
- **有效范围**：0 ~ (ZOOM.length - 1)，超出回退到5
- **对应关系**：0=半小时, 1=1小时, 2=2小时, 3=6小时, 4=12小时, 5=1天, 6=3天, 7=15天, 8=30天, 9=365天, 10=10年

#### 4. showCenterLine
```javascript
showCenterLine: { type: Boolean, default: true }
```
- **设为false**：不绘制中间竖线，适用于纯展示场景

#### 5. centerLineStyle
```javascript
centerLineStyle: { type: Object, default() { return { width: 2, color: '#fff' } } }
```
- **width**：线宽（Canvas lineWidth，像素）
- **color**：线颜色（CSS颜色值）

#### 6. textColor
```javascript
textColor: { type: String, default: 'rgba(151,158,167,1)' }
```
- **影响范围**：时间轴上所有刻度文字（日期文字、时间文字）

#### 7. hoverTextColor
```javascript
hoverTextColor: { type: String, default: 'rgb(194, 202, 215)' }
```
- **影响范围**：鼠标悬停时显示的时间文字（区别于刻度文字）

#### 8. lineColor
```javascript
lineColor: { type: String, default: 'rgba(151,158,167,1)' }
```
- **影响范围**：所有刻度竖线 + hover指示竖线

#### 9. lineHeightRatio
```javascript
lineHeightRatio: {
  type: Object,
  default() {
    return {
      date: 0.3,   // 0点处日期刻度线高度 = canvasHeight * 0.3
      time: 0.2,   // 有文字的普通时间刻度线高度
      none: 0.1,   // 无文字的小刻度线高度
      hover: 0.3   // hover指示线高度
    }
  }
}
```
- **视觉层次**：date(最长) ≥ hover > time > none(最短)
- **文字定位**：文字top = 线段底部 + 15px

#### 10. showHoverTime
```javascript
showHoverTime: { type: Boolean, default: true }
```
- **false时**：鼠标移动不触发hover显示，只有拖动功能

#### 11. hoverTimeFormat
```javascript
hoverTimeFormat: { type: Function }
// 签名: (time: number) => string
// time 是鼠标所在位置对应的时间戳
```
- **默认格式**：`dayjs(time).format('YYYY-MM-DD HH:mm:ss')`
- **返回空字符串**：不显示时间文字（但仍显示指示线）
- **典型用法**：
  ```javascript
  // 只显示当天的时间
  hoverTimeFormat(time) {
    if (dayjs(time).isBefore(dayjs().format('YYYY-MM-DD 00:00:00'))) return ''
    if (dayjs(time).isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) return ''
    return dayjs(time).format('HH:mm:ss')
  }
  ```

#### 12. timeSegments
```javascript
timeSegments: { type: Array, default: () => [] }
```
- **每项数据结构**：
  ```typescript
  interface TimeSegment {
    name?: string           // 自定义名称（用于点击识别）
    beginTime: number       // 起始时间戳（必填）
    endTime?: number        // 结束时间戳（可选，不传则绘制1px宽线段）
    color: string           // 填充颜色（必填）
    startRatio?: number     // 纵向起始比例（默认0.6）
    endRatio?: number       // 纵向结束比例（默认0.9）
    [key: string]: any      // 可附加任意自定义字段
  }
  ```
- **响应式**：`watch(() => props.timeSegments, reRender, { deep: true })`
- **示例**：
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

#### 13. backgroundColor
```javascript
backgroundColor: { type: String, default: '#262626' }
```
- **实现方式**：通过 `:style="{ backgroundColor }"` 设置到容器div上
- **Canvas本身透明**：Canvas背景透明，底色由容器div提供

#### 14. multiSegmentActiveColor
```javascript
multiSegmentActiveColor: { type: String }
```
- **传递给** WindowListItem 组件的 `multiSegmentActiveColor` prop
- **作用**：选中的窗口时间轴的背景高亮色

#### 15. enableZoom
```javascript
enableZoom: { type: Boolean, default: true }
```
- **false时**：滚轮事件处理函数直接return

#### 16. enableDrag
```javascript
enableDrag: { type: Boolean, default: true }
```
- **false时**：mousemove中不执行drag函数；mouseup中不emit dragTimeChange

#### 17. windowList
```javascript
windowList: { type: Array, default: () => [] }
```
- **每项结构**：
  ```typescript
  interface WindowItem {
    name: string                    // 窗口名称（显示序号用）
    timeSegments?: TimeSegment[]    // 该窗口的时间段
    [key: string]: any              // 任意自定义字段
  }
  ```
- **显示条件**：`windowList.length > 1`
- **内部副作用**：Canvas高度变为 `baseTimeLineHeight`

#### 18. baseTimeLineHeight
```javascript
baseTimeLineHeight: { type: Number, default: 50 }
```
- **仅在** `windowList.length > 1` 时生效
- **逻辑**：`height = windowList.length > 1 ? baseTimeLineHeight : 容器全高`

#### 19. initSelectWindowTimeLineIndex
```javascript
initSelectWindowTimeLineIndex: { type: Number, default: -1 }
```
- **-1**：不选中任何窗口
- **0~N**：初始选中第N个窗口（添加active样式）

#### 20. isMobile
```javascript
isMobile: { type: Boolean, default: false }
```
- **true时的变化**：
  1. mouse事件忽略，只处理touch事件
  2. 使用 MOBILE_ZOOM_HOUR_GRID（格子更大）
  3. 使用 MOBILE_ZOOM_DATE_SHOW_RULE（文字更少）
  4. 全局监听 touchend 而非 mouseup
  5. mouseleave时不重置mousemoveX

#### 21. maxClickDistance
```javascript
maxClickDistance: { type: Number, default: 3 }
```
- **判定逻辑**：mousedown到mouseup的X、Y移动量均≤此值 → 点击
- **大于此值** → 拖拽

#### 22. roundWidthTimeSegments
```javascript
roundWidthTimeSegments: { type: Boolean, default: true }
```
- **true时**：时间段绘制坐标 `x = Math.round(x)`, `w = Math.round(w)`
- **目的**：防止相邻时间段之间出现亚像素间隙

#### 23. customShowTime
```javascript
customShowTime: { type: Function }
// 签名: (date: Date, currentZoomIndex: number) => boolean | any
```
- **三值返回**：
  - `true`：强制显示
  - `false`：强制隐藏
  - 其他值(undefined)：走内置规则
- **调用位置**：addGraduations中对每个刻度时间点调用

#### 24. showDateAtZero
```javascript
showDateAtZero: { type: Boolean, default: true }
```
- **true**：0点刻度绘制最长线段 + 显示 `MM-DD` 格式日期
- **false**：0点和其他时间一样处理

#### 25. extendZOOM
```javascript
extendZOOM: { type: Array, default() { return [] } }
// 每项: { zoom: 25, zoomHourGrid: 0.5, mobileZoomHourGrid: 2 }
```
- **处理时机**：组件script setup顶层执行（非响应式，只执行一次）
- **⚠️ 必须配合 customShowTime**：否则内置规则数组越界报错
- **索引**：追加后的索引从11开始（0~10是内置）

#### 26. formatTime
```javascript
formatTime: { type: Function }
// 签名: (time: dayjs.Dayjs) => string | ''
// 注意：参数是dayjs对象，不是时间戳！
```
- **返回空值**：走内置格式化规则
- **内置规则**：
  ```javascript
  if (yearMode) return 'YYYY'        // 如 "2021"
  if (yearMonthMode) return 'YYYY-MM' // 如 "2021-01"
  if (0点) return 'MM-DD'             // 如 "01-15"
  else return 'HH:mm'                 // 如 "14:30"
  ```
- **典型用法**（24点显示为24:00而非00:00）：
  ```javascript
  formatTime(time) {
    if (time.isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) return '24:00'
    if (time.hour() === 0 && time.minute() === 0) return time.format('HH:mm')
  }
  ```

---


## 第六部分：Events 事件系统（8个事件）

### 事件总览

| # | 事件名 | 触发时机 | 回调参数 |
|---|--------|---------|---------|
| 1 | timeChange | 每次draw()后 | (currentTime: number) |
| 2 | mousedown | 鼠标/触摸按下 | (event: Event) |
| 3 | mouseup | 鼠标/触摸松开 | (event: Event) |
| 4 | dragTimeChange | 拖拽结束 | (currentTime: number) |
| 5 | click_timeSegments | 点击到时间段 | (segments[], time, date, x) |
| 6 | click_timeline | 点击空白区域 | (time, date, x) |
| 7 | change_window_time_line | 切换窗口选中 | (index, item) |
| 8 | click_window_timeSegments | 点击窗口时间段 | (segments[], index, item) |

### 各事件详解

#### 1. timeChange
```javascript
emits('timeChange', defaultData.currentTime)
// currentTime = startTimestamp + totalMS / 2（中心点时间戳）
```
- **触发频率**：非常高！拖动每帧、hover每次移动、缩放每次都触发
- **典型用途**：实时显示当前时间、同步视频播放位置
- **⚠️ 注意**：在hover时也会触发（因为hoverShow内部调用了draw()）

#### 2. mousedown
```javascript
emits('mousedown', e)
```
- **触发时机**：鼠标按下/触摸开始时
- **参数**：原生事件对象
- **用途**：用于外部判断用户开始交互

#### 3. mouseup
```javascript
emits('mouseup', e)
```
- **触发时机**：鼠标松开/触摸结束时（无论是点击还是拖拽结束）
- **⚠️ 注意**：点击事件时也会触发（先触发click逻辑，再emit mouseup）

#### 4. dragTimeChange
```javascript
emits('dragTimeChange', defaultData.currentTime)
```
- **触发条件**：`mousedown=true && enableDrag=true && 移动距离>maxClickDistance`
- **含义**：拖动操作结束后的最终时间
- **典型用途**：拖动结束后请求对应时间的视频数据

#### 5. click_timeSegments
```javascript
emits('click_timeSegments', timeSegments, time, date, x)
```
- **参数详解**：
  - `timeSegments`：点击命中的时间段数组（可能命中多个重叠的）
  - `time`：点击位置对应的时间戳
  - `date`：格式化后的时间字符串 `'YYYY-MM-DD HH:mm:ss'`
  - `x`：点击位置相对时间轴左侧的像素距离
- **触发条件**：点击位置在某个时间段的矩形区域内
- **检测方式**：Canvas `ctx.isPointInPath(x, y)`

#### 6. click_timeline
```javascript
emits('click_timeline', time, date, x)
```
- **触发条件**：点击位置**不在**任何时间段上
- **参数**：同 click_timeSegments 的后三个参数
- **典型用途**：点击时间轴空白处跳转到该时间

#### 7. change_window_time_line
```javascript
emits('change_window_time_line', index, defaultData.windowListInner[index])
```
- **触发时机**：点击某个窗口时间轴切换选中状态
- **参数**：
  - `index`：窗口索引（从0开始）
  - `item`：该窗口的完整数据对象

#### 8. click_window_timeSegments
```javascript
emits('click_window_timeSegments', data, index, item)
```
- **触发时机**：点击到某个窗口时间轴中的时间段
- **参数**：
  - `data`：命中的时间段数组
  - `index`：窗口索引
  - `item`：窗口数据

### 事件触发流程图

```
mousedown → 记录位置 → emit('mousedown')
     ↓
mousemove → 拖动/hover
     ↓
mouseup → 计算移动距离
     ├── 距离 ≤ maxClickDistance → 判定为点击
     │    ├── 命中时间段 → emit('click_timeSegments')
     │    └── 未命中 → emit('click_timeline')
     └── 距离 > maxClickDistance → 判定为拖拽
          └── emit('dragTimeChange')
     最后 → emit('mouseup')
```

---

## 第七部分：Expose 暴露方法（4个方法）

```javascript
defineExpose({ setTime, setZoom, watchTime, reRender })
```

### 7.1 setTime(t)

```javascript
const setTime = (t) => {
  if (defaultData.mousedown) return  // 正在拖动时忽略
  let ts = typeof t === 'number' ? t : new Date(t).getTime()
  defaultData.startTimestamp = ts - totalMS.value / 2
  fixStartTimestamp()  // 检查时间范围边界
  clearCanvas(defaultData.width, defaultData.height)
  draw()
  // 如果鼠标在时间轴上，补绘hover效果
  if (defaultData.mousemoveX !== -1 && !props.isMobile) {
    hoverShow(defaultData.mousemoveX, true)  // noDraw=true，不再清除重绘
  }
}
```
- **参数**：时间戳数字或时间字符串
- **用途**：外部控制时间轴跳转到指定时间（如定时器每秒推进）
- **典型用法**：
  ```javascript
  // 每秒推进时间
  setInterval(() => {
    currentTime += 1000
    timelineRef.value.setTime(currentTime)
  }, 1000)
  ```

### 7.2 setZoom(index)

```javascript
const setZoom = (index) => {
  defaultData.currentZoomIndex = (index >= 0 && index < ZOOM.length) ? index : 5
  clearCanvas(defaultData.width, defaultData.height)
  // 以当前中心时间为锚点重新计算startTimestamp
  defaultData.startTimestamp = defaultData.currentTime - totalMS.value / 2
  draw()
}
```
- **参数**：分辨率索引（0~10 + 扩展索引）
- **锚点逻辑**：缩放前后保持中心点时间不变
- **典型用法**：
  ```javascript
  // 下拉框切换分辨率
  timelineRef.value.setZoom(6)  // 切换到3天视图
  ```

### 7.3 watchTime(time, callback, windowTimeLineIndex?)

```javascript
const watchTime = (time, callback, windowTimeLineIndex) => {
  if (!time || !callback) return
  defaultData.watchTimeList.push({
    time: typeof time === 'number' ? time : new Date(time).getTime(),
    callback,
    windowTimeLineIndex: typeof windowTimeLineIndex === 'number' ? windowTimeLineIndex - 1 : -1
  })
}
```
- **参数**：
  - `time`：要观察的时间点
  - `callback`：`(x: number, y: number) => void`，返回像素坐标
  - `windowTimeLineIndex`：可选，指定在第几个窗口时间轴上（从1开始）
- **回调规则**：
  - 时间点在可视范围内：返回 (x+left, top) 相对视口的绝对坐标
  - 时间点不在可视范围：返回 (-1, -1)
- **更新时机**：每次draw()后自动调用 `updateWatchTime()`
- **典型用法**：
  ```javascript
  // 在时间轴上显示一个图标
  timelineRef.value.watchTime('2021-01-01 23:30:00', (x, y) => {
    if (x === -1) {
      icon.style.display = 'none'
    } else {
      icon.style.display = 'block'
      icon.style.left = x + 'px'
      icon.style.top = y + 'px'
    }
  })
  ```

### 7.4 reRender()

```javascript
const reRender = () => {
  nextTick(() => {
    clearCanvas(defaultData.width, defaultData.height)
    reset()          // 重置所有内部状态
    setInitData()    // 根据props重新计算初始数据
    init()           // 重新获取容器尺寸、创建Canvas context
    draw()           // 重新绘制
  })
}
```
- **用途**：完全重置并重新渲染组件
- **使用场景**：容器尺寸变化、props大幅变更后

---


## 第八部分：内部状态管理

### 8.1 reactive 状态对象

```javascript
const defaultData = reactive({
  width: 0,                        // Canvas宽度（像素）
  height: 0,                       // Canvas高度（像素）
  ctx: null,                       // CanvasRenderingContext2D
  currentZoomIndex: 0,             // 当前分辨率索引
  currentTime: 0,                  // 当前中心点时间（时间戳）
  startTimestamp: 0,               // 时间轴左端对应的时间戳（核心状态！）
  mousedown: false,                // 鼠标是否按下
  mousedownX: 0,                   // 按下时的X坐标（相对容器）
  mousedownY: 0,                   // 按下时的Y坐标（相对容器）
  mousedownCacheStartTimestamp: 0, // 按下时缓存的startTimestamp
  showWindowList: false,           // 是否显示多窗口列表
  windowListInner: [],             // 内部窗口列表数据（带active状态）
  mousemoveX: -1,                  // 鼠标当前X位置，-1表示不在时间轴上
  watchTimeList: []                // 观察的时间点列表
})
```

### 8.2 关键 computed

```javascript
// 整个时间轴代表的总毫秒数
const totalMS = computed(() => ZOOM[currentZoomIndex] * ONE_HOUR_STAMP)

// timeRange转为时间戳格式
const timeRangeTimestamp = computed(() => {
  let t = {}
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

// 根据isMobile选择对应的配置数组
const ACT_ZOOM_HOUR_GRID = computed(() => props.isMobile ? MOBILE_ZOOM_HOUR_GRID : ZOOM_HOUR_GRID)
const ACT_ZOOM_DATE_SHOW_RULE = computed(() => props.isMobile ? MOBILE_ZOOM_DATE_SHOW_RULE : ZOOM_DATE_SHOW_RULE)

// 特殊模式判断
const yearMonthMode = computed(() => currentZoomIndex === 9)
const yearMode = computed(() => currentZoomIndex === 10)
```

### 8.3 模板 ref

```javascript
const timeLineContainer = ref(null)  // 最外层容器div
const canvas = ref(null)              // Canvas元素
const WindowListItemRef = ref([])     // 子窗口组件数组ref
```

---

## 第九部分：核心数学模型

### 9.1 时间-像素映射（最核心的公式）

```
┌──────────────────── Canvas (width px) ────────────────────┐
│                                                            │
│ startTimestamp              currentTime         endTime    │
│ ◄──────────── totalMS ──────────────────────────────────► │
│                    ↑ (width/2)                             │
│                 中心线                                      │
└────────────────────────────────────────────────────────────┘
```

**基础公式**：
```javascript
totalMS = ZOOM[currentZoomIndex] * 3600000      // 时间轴总毫秒数
PX_PER_MS = width / totalMS                      // 每毫秒对应的像素数
currentTime = startTimestamp + totalMS / 2        // 中心点时间

// 时间戳 → 像素X坐标
x = (timestamp - startTimestamp) * PX_PER_MS

// 像素X坐标 → 时间戳
timestamp = startTimestamp + x / PX_PER_MS
```

### 9.2 刻度对齐算法

确保刻度线对齐到"整数时间点"（如整分钟、整小时）：

```javascript
// 一格代表的毫秒数
const msPerGrid = ZOOM_HOUR_GRID[zoomIndex] * ONE_HOUR_STAMP

// 起始偏移：从startTimestamp到下一个对齐点的距离
const msOffset = msPerGrid - (startTimestamp % msPerGrid)
const pxOffset = (msOffset / msPerGrid) * pxPerGrid

// 第i格的X坐标
x[i] = pxOffset + i * pxPerGrid

// 第i格对应的时间
time[i] = startTimestamp + msOffset + i * msPerGrid
```

**为什么需要偏移**：
- startTimestamp 通常不会正好落在"整分钟"上
- 比如 startTimestamp = 10:00:37，msPerGrid = 1分钟
- msOffset = 60000 - (37000) = 23000ms（23秒后是10:01:00）
- 这样第一个刻度就对齐到了 10:01:00

### 9.3 年/月模式特殊处理

年模式和月模式下，月份天数不固定，不能简单用等间距：

```javascript
if (yearMode) {
  // 将时间对齐到年初 1月1日
  adjustMsOffset = currentStartTimestamp -
    new Date(`${year}-01-01 00:00:00`).getTime()
} else if (yearMonthMode) {
  // 将时间对齐到月初 1日
  adjustMsOffset = currentStartTimestamp -
    new Date(`${year}-${month}-01 00:00:00`).getTime()
}
// 修正X坐标
x = pxOffset + i * pxPerGrid - (adjustMsOffset / msPerGrid) * pxPerGrid
// 修正时间
graduationTime = currentStartTimestamp - adjustMsOffset
```

### 9.4 拖拽位移计算

```javascript
const drag = (currentX) => {
  const PX_PER_MS = width / totalMS
  const diffX = currentX - mousedownX  // 鼠标移动的像素差

  // 向右拖 → diffX正 → 时间回退（看更早的时间）
  // 向左拖 → diffX负 → 时间前进（看更晚的时间）
  let newStartTimestamp = mousedownCacheStartTimestamp - Math.round(diffX / PX_PER_MS)

  // 边界钳位
  let centerTime = newStartTimestamp + totalMS / 2
  if (timeRange.start && centerTime < timeRange.start) {
    newStartTimestamp = timeRange.start - totalMS / 2
  }
  if (timeRange.end && centerTime > timeRange.end) {
    newStartTimestamp = timeRange.end - totalMS / 2
  }

  startTimestamp = newStartTimestamp
  clearCanvas()
  draw()
}
```

### 9.5 缩放锚点计算

```javascript
const onMousewheel = (event) => {
  // delta < 0 缩小（看更大范围），delta > 0 放大（看更小范围）
  if (delta < 0) currentZoomIndex++ (有上限)
  if (delta > 0) currentZoomIndex-- (有下限)

  // 关键：以当前中心时间为锚点
  // 缩放后 totalMS 变了，但 currentTime 不变
  startTimestamp = currentTime - totalMS / 2  // 重新计算左端点
}
```

---

## 第十部分：绘制系统完整流程

### 10.1 draw() 主绘制函数（绘制顺序）

```javascript
const draw = () => {
  // 顺序很重要！先画的在底层
  drawTimeSegments()    // 1. 先绘制时间段（在最底层）
  addGraduations()      // 2. 再绘制刻度（覆盖在时间段上方）
  drawMiddleLine()      // 3. 最后绘制中心线（在最上层）

  // 更新当前时间
  currentTime = startTimestamp + totalMS / 2
  emits('timeChange', currentTime)

  // 通知子窗口重绘
  WindowListItemRef.value.forEach(item => item.draw())

  // 更新watchTime位置
  updateWatchTime()
}
```

### 10.2 drawTimeSegments() 时间段绘制

```javascript
const drawTimeSegments = (callback, path) => {
  const PX_PER_MS = width / totalMS

  props.timeSegments.forEach((item) => {
    // 可见性判断：时间段是否与当前视口有交集
    if (item.beginTime > startTimestamp + totalMS) return  // 完全在右边
    // （注意：没有判断完全在左边的情况，因为后面x<0时会裁剪）

    let hasEndTime = item.endTime >= startTimestamp

    ctx.beginPath()
    let x = (item.beginTime - startTimestamp) * PX_PER_MS
    let w

    if (x < 0) {
      // 时间段起点在左边界之外 → 从0开始绘制
      x = 0
      w = hasEndTime ? (item.endTime - startTimestamp) * PX_PER_MS : 1
    } else {
      w = hasEndTime ? (item.endTime - item.beginTime) * PX_PER_MS : 1
    }

    let heightStartRatio = item.startRatio ?? 0.6
    let heightEndRatio = item.endRatio ?? 0.9

    // 四舍五入避免亚像素间隙
    if (props.roundWidthTimeSegments) {
      x = Math.round(x)
      w = Math.round(w)
    }
    w = Math.max(1, w)  // 最小1px

    if (path) {
      // 路径模式：只记录rect路径，用于hitTest
      ctx.rect(x, height * heightStartRatio, w, height * (heightEndRatio - heightStartRatio))
    } else {
      // 绘制模式：实际填充颜色
      ctx.fillStyle = item.color
      ctx.fillRect(x, height * heightStartRatio, w, height * (heightEndRatio - heightStartRatio))
    }

    callback && callback(item)  // hitTest时的回调
  })
}
```

### 10.3 addGraduations() 刻度绘制

```javascript
const addGraduations = () => {
  ctx.beginPath()

  // 计算格子参数
  const gridNum = ZOOM[currentZoomIndex] / ACT_ZOOM_HOUR_GRID[currentZoomIndex]
  const msPerGrid = ACT_ZOOM_HOUR_GRID[currentZoomIndex] * ONE_HOUR_STAMP
  const pxPerGrid = width / gridNum
  const msOffset = msPerGrid - (startTimestamp % msPerGrid)
  const pxOffset = (msOffset / msPerGrid) * pxPerGrid

  for (let i = 0; i < gridNum; i++) {
    let currentStartTimestamp = startTimestamp + msOffset + i * msPerGrid

    // 年/月模式修正
    let adjustMsOffset = 0
    if (yearMode) {
      adjustMsOffset = currentStartTimestamp -
        new Date(`${year}-01-01 00:00:00`).getTime()
    } else if (yearMonthMode) {
      adjustMsOffset = currentStartTimestamp -
        new Date(`${year}-${month}-01 00:00:00`).getTime()
    }

    let x = pxOffset + i * pxPerGrid - (adjustMsOffset / msPerGrid) * pxPerGrid
    let graduationTime = currentStartTimestamp - adjustMsOffset
    let date = new Date(graduationTime)
    let h = 0

    // 三级判断：0点日期 > 显示时间 > 不显示时间
    if (props.showDateAtZero && date.getHours() === 0 && date.getMinutes() === 0) {
      // 0点：长线段 + 日期文字
      h = height * (lineHeightRatio.date ?? 0.3)
      ctx.fillStyle = props.textColor
      ctx.fillText(graduationTitle(graduationTime), x - 13, h + 15)
    } else if (checkShowTime(date)) {
      // 显示时间：中等线段 + 时间文字
      h = height * (lineHeightRatio.time ?? 0.2)
      ctx.fillStyle = props.textColor
      ctx.fillText(graduationTitle(graduationTime), x - 13, h + 15)
    } else {
      // 不显示：短线段
      h = height * (lineHeightRatio.none ?? 0.1)
    }

    drawLine(x, 0, x, h, 1, props.lineColor)
  }
}
```

### 10.4 drawMiddleLine() 中心线绘制

```javascript
const drawMiddleLine = () => {
  if (!props.showCenterLine) return
  ctx.beginPath()
  const { width: lineWidth, color } = props.centerLineStyle
  const x = width / 2
  drawLine(x, 0, x, height, lineWidth, color)
}
```

### 10.5 drawLine() 基础线段绘制

```javascript
const drawLine = (x1, y1, x2, y2, lineWidth = 1, color = '#fff') => {
  ctx.beginPath()
  ctx.strokeStyle = color
  ctx.lineWidth = lineWidth
  ctx.moveTo(x1, y1)
  ctx.lineTo(x2, y2)
  ctx.stroke()
}
```

### 10.6 hoverShow() 悬停时间显示

```javascript
const hoverShow = (x, noDraw) => {
  const PX_PER_MS = width / totalMS
  const time = startTimestamp + x / PX_PER_MS  // 鼠标位置对应的时间

  if (!noDraw) {
    clearCanvas(width, height)
    draw()  // 先完整重绘（会清除上一帧的hover）
  }

  // 绘制hover指示线
  const h = height * (lineHeightRatio.hover ?? 0.3)
  drawLine(x, 0, x, h, 1, props.lineColor)

  // 绘制hover时间文字
  ctx.fillStyle = props.hoverTextColor
  const t = props.hoverTimeFormat
    ? props.hoverTimeFormat(time)
    : dayjs(time).format('YYYY-MM-DD HH:mm:ss')
  const w = ctx.measureText(t).width
  ctx.fillText(t, x - w / 2, h + 20)  // 居中于指示线下方
}
```

### 10.7 graduationTitle() 时间格式化

```javascript
const graduationTitle = (datetime) => {
  let time = dayjs(datetime)

  // 优先使用自定义格式化
  if (props.formatTime) {
    let res = props.formatTime(time)
    if (res) return res
  }

  // 内置规则
  if (yearMode) return time.format('YYYY')           // "2021"
  if (yearMonthMode) return time.format('YYYY-MM')   // "2021-01"
  if (time.hour() === 0 && time.minute() === 0 && time.millisecond() === 0) {
    return time.format('MM-DD')                       // "01-15"
  }
  return time.format('HH:mm')                         // "14:30"
}
```

### 10.8 checkShowTime() 时间显示判断

```javascript
const checkShowTime = (date) => {
  // 先检查自定义规则
  if (props.customShowTime) {
    let res = props.customShowTime(date, currentZoomIndex)
    if (res === true) return true
    if (res === false) return false
    // undefined/null → 继续走内置规则
  }
  // 内置规则
  return ACT_ZOOM_DATE_SHOW_RULE[currentZoomIndex](date)
}
```

---


## 第十一部分：交互系统完整流程

### 11.1 事件绑定策略

```
容器div绑定：touchstart, touchmove, mousedown, mouseout, mousemove, mouseleave
Canvas绑定：mousewheel（stop.prevent修饰符，阻止页面滚动）
全局window绑定：mouseup/touchend（确保拖出Canvas仍能松开）, resize
```

### 11.2 PC端鼠标交互完整流程

#### mousedown → onMousedown → onPointerdown
```javascript
const onMousedown = (e) => {
  if (props.isMobile) return  // 移动端忽略鼠标事件
  onPointerdown(e)
}

const onPointerdown = (e) => {
  let pos = getClientOffset(e)        // 获取相对容器的坐标
  e.target.style.cursor = 'grabbing'  // 改变鼠标样式
  defaultData.mousedownX = pos[0]
  defaultData.mousedownY = pos[1]
  defaultData.mousedown = true
  defaultData.mousedownCacheStartTimestamp = startTimestamp  // 缓存当前位置
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
    drag(x)       // 按下状态 → 执行拖拽
  } else if (props.showHoverTime) {
    hoverShow(x)  // 非按下状态 → 显示hover时间
  }
}
```

#### mouseup（全局window） → onMouseup → onPointerup
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

  // 判断是点击还是拖拽
  if (Math.abs(pos[0] - defaultData.mousedownX) <= props.maxClickDistance &&
      Math.abs(pos[1] - defaultData.mousedownY) <= props.maxClickDistance) {
    reset()
    onClick(...pos)  // 点击
    return
  }

  if (defaultData.mousedown && props.enableDrag) {
    reset()
    emits('dragTimeChange', defaultData.currentTime)  // 拖拽结束
  } else {
    reset()
  }
  emits('mouseup', e)
}
```

#### mouseout → onMouseout
```javascript
const onMouseout = () => {
  clearCanvas(width, height)
  draw()  // 清除hover效果，重新绘制干净的时间轴
}
```

#### mouseleave → onMouseleave
```javascript
const onMouseleave = () => {
  defaultData.mousemoveX = -1  // 标记鼠标已离开
}
```

### 11.3 移动端触摸交互

#### touchstart → onTouchstart
```javascript
const onTouchstart = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]  // 取第一个触摸点
  onPointerdown(e)   // 复用通用按下逻辑
}
```

#### touchmove → onTouchmove
```javascript
const onTouchmove = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]
  onPointermove(e)
}
```

#### touchend（全局window）
```javascript
let onTouchend = (e) => {
  if (!props.isMobile) return
  e = e.touches[0]
  onPointerup(e)
}
```

### 11.4 滚轮缩放

```javascript
const onMouseweel = (event) => {
  if (!props.enableZoom) return

  let e = window.event || event
  let delta = Math.max(-1, Math.min(1, e.wheelDelta || -e.detail))

  if (delta < 0) {
    // 向下滚 → 缩小（看更大范围）
    if (currentZoomIndex + 1 >= ZOOM.length - 1) {
      currentZoomIndex = ZOOM.length - 1
    } else {
      currentZoomIndex++
    }
  } else if (delta > 0) {
    // 向上滚 → 放大（看更小范围）
    if (currentZoomIndex - 1 <= 0) {
      currentZoomIndex = 0
    } else {
      currentZoomIndex--
    }
  }

  clearCanvas(width, height)
  // 以currentTime为锚点重算startTimestamp
  startTimestamp = currentTime - totalMS / 2
  draw()
}
```

### 11.5 点击事件处理

```javascript
const onClick = (x, y) => {
  const PX_PER_MS = width / totalMS
  let time = startTimestamp + x / PX_PER_MS
  let date = dayjs(time).format('YYYY-MM-DD HH:mm:ss')

  // 检测是否点击到了时间段
  let timeSegments = getClickTimeSegments(x, y)
  if (timeSegments && timeSegments.length > 0) {
    emits('click_timeSegments', timeSegments, time, date, x)
  } else {
    emits('click_timeline', time, date, x)
  }
}
```

### 11.6 时间段点击检测（Canvas hitTest）

```javascript
const getClickTimeSegments = (x, y) => {
  let inItems = []
  // 重新绘制所有时间段的路径（不实际填充），检测点击
  drawTimeSegments((item) => {
    if (ctx.isPointInPath(x, y)) {
      inItems.push(item)
    }
  }, true)  // path=true → 只创建rect路径不填充
  return inItems
}
```

**原理**：
1. `drawTimeSegments(callback, path=true)` 为每个时间段调用 `ctx.rect()` 而非 `ctx.fillRect()`
2. 每次 rect 后立即调用 `ctx.isPointInPath(x, y)` 检测点击坐标是否在该路径内
3. 注意每个时间段都 `ctx.beginPath()` 了，所以 isPointInPath 只检测当前路径

### 11.7 坐标转换工具

```javascript
const getClientOffset = (e) => {
  if (!timeLineContainer.value || !e) return [0, 0]
  let { left, top } = timeLineContainer.value.getBoundingClientRect()
  return [e.clientX - left, e.clientY - top]
  // 将浏览器视口坐标转为容器内相对坐标
}
```

---

## 第十二部分：多窗口子组件 WindowListItem

### 12.1 组件职责

每个 WindowListItem 代表一个"播放窗口"的时间轴，独立拥有自己的 Canvas，绘制该窗口的时间段。

### 12.2 模板结构

```html
<template>
  <div class="windowListItem" :class="{ active }" ref="windowListItem" @click="onClick">
    <span class="order">{{ index + 1 }}</span>  <!-- 窗口序号 -->
    <canvas class="windowListItemCanvas" ref="canvas"></canvas>  <!-- 独立Canvas -->
  </div>
</template>
```

### 12.3 Props

```javascript
const props = defineProps({
  index: { type: Number },                    // 窗口索引
  data: { type: Object, default: () => ({}) }, // 窗口数据(含timeSegments)
  totalMS: { type: Number },                   // 从父组件同步的totalMS
  startTimestamp: { type: Number },            // 从父组件同步的startTimestamp
  width: { type: Number },                     // 从父组件同步的Canvas宽度
  active: { type: Boolean, default: false },   // 是否选中
  multiSegmentActiveColor: { type: String, default: '#333' }  // 选中背景色
})
```

### 12.4 核心逻辑

```javascript
// 初始化
const init = () => {
  let { height } = windowListItem.value.getBoundingClientRect()
  defaultData.height = height - 1  // -1 留给border
  canvas.value.width = props.width
  canvas.value.height = defaultData.height
  defaultData.ctx = canvas.value.getContext('2d')
}

// 绘制（由父组件在draw()中调用）
const draw = () => {
  nextTick(() => {
    clearCanvas()
    drawTimeSegments()
  })
}

// 时间段绘制（逻辑与父组件相同）
const drawTimeSegments = (callback, path) => {
  if (!props.data.timeSegments || props.data.timeSegments.length <= 0) return
  const PX_PER_MS = props.width / props.totalMS
  props.data.timeSegments.forEach((item) => {
    // 可见性判断 + 坐标计算 + 绘制（同主组件）
  })
}

// 点击事件
const onClick = (e) => {
  emits('click', e)  // 触发选中切换
  // 检测是否点击到时间段
  let { left, top } = windowListItem.value.getBoundingClientRect()
  let x = e.clientX - left
  let y = e.clientY - top
  let timeSegments = getClickTimeSegments(x, y)
  if (timeSegments.length > 0) {
    emits('click_window_timeSegments', timeSegments, props.index, props.data)
  }
}
```

### 12.5 Expose

```javascript
defineExpose({ draw, getRect })
// draw: 由父组件调用触发重绘
// getRect: 返回DOM位置信息（用于watchTime定位）
```

### 12.6 样式

```scss
.windowListItem {
  width: 100%;
  height: 30px;             // 每个窗口固定30px高
  position: relative;
  border-bottom: 1px solid rgba(153, 153, 153, 1);
  user-select: none;

  &.active {
    background-color: v-bind(multiSegmentActiveColor);  // 选中高亮
  }

  .order {
    position: absolute;
    width: 30px; height: 30px;
    display: flex; justify-content: center; align-items: center;
    color: #fff;
    border-right: 1px solid rgba(153, 153, 153, 1);
  }
}
```

### 12.7 父子组件数据流

```
父组件 TimeLine.vue
  │
  ├── props传递：totalMS, startTimestamp, width（保持时间同步）
  ├── ref调用：WindowListItemRef[i].draw()（触发重绘）
  │
  └── 事件冒泡：
      click → 父组件 toggleActive(index) → 切换选中状态
      click_window_timeSegments → 父组件转发为 emit('click_window_timeSegments')
```

---


## 第十三部分：生命周期与响应式

### 13.1 onMounted 初始化流程

```javascript
onMounted(() => {
  setInitData()   // 1. 设置初始数据（时间、分辨率、窗口列表）
  init()          // 2. 获取容器尺寸、创建Canvas context
  draw()          // 3. 首次绘制

  // 4. 全局事件监听
  if (props.isMobile) {
    window.addEventListener('touchend', onTouchend)
  } else {
    window.addEventListener('mouseup', onMouseup)
  }
  window.addEventListener('resize', onResize)
})
```

### 13.2 onBeforeUnmount 清理

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

### 13.3 setInitData() 详解

```javascript
const setInitData = () => {
  // 1. 构建内部窗口列表（添加active字段）
  defaultData.windowListInner = props.windowList.map((item, index) => ({
    ...item,
    active: props.initSelectWindowTimeLineIndex === index
  }))

  // 2. 设置初始分辨率（有效性检查）
  defaultData.currentZoomIndex =
    (props.initZoomIndex >= 0 && props.initZoomIndex < ZOOM.length) ? props.initZoomIndex : 5

  // 3. 计算初始startTimestamp
  const initTime = props.initTime
    ? (typeof props.initTime === 'number' ? props.initTime : new Date(props.initTime).getTime())
    : new Date(dayjs().format('YYYY-MM-DD 00:00:00')).getTime()
  defaultData.startTimestamp = initTime - totalMS.value / 2

  // 4. 边界修正
  fixStartTimestamp()
}
```

### 13.4 init() 详解

```javascript
const init = () => {
  // 获取容器实际尺寸
  let { width, height } = timeLineContainer.value.getBoundingClientRect()
  defaultData.width = width
  // Canvas高度：有多窗口时用baseTimeLineHeight，否则用容器全高
  defaultData.height = props.windowList.length > 1 ? props.baseTimeLineHeight : height

  // 设置Canvas元素尺寸（必须设置width/height属性，非CSS）
  canvas.value.width = defaultData.width
  canvas.value.height = defaultData.height

  // 获取2D绑制上下文
  defaultData.ctx = canvas.value.getContext('2d')

  // 触发窗口列表显示
  defaultData.showWindowList = true
}
```

### 13.5 watch 监听

```javascript
// timeSegments变化时完整重新渲染
watch(() => props.timeSegments, reRender, { deep: true })
```

### 13.6 onResize 响应式处理

```javascript
let onResize = () => {
  init()   // 重新获取尺寸、重建Canvas
  draw()   // 重新绘制
  // 子窗口也重新初始化
  try {
    WindowListItemRef.value.forEach(item => item.init())
  } catch (error) {
    console.error(error)
  }
}
```

### 13.7 reset() 状态重置

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

### 13.8 extendZOOM 处理时机

```javascript
// 在 script setup 顶层执行（组件创建时，只执行一次）
props.extendZOOM.forEach((item) => {
  ZOOM.push(item.zoom)
  ZOOM_HOUR_GRID.push(item.zoomHourGrid)
  MOBILE_ZOOM_HOUR_GRID.push(item.mobileZoomHourGrid)
})
```
- **⚠️ 注意**：这会修改 constant.ts 导出的数组（引用类型，全局共享）
- **潜在问题**：如果页面上有多个 TimeLine 实例，extendZOOM 会被多次push

---

## 第十四部分：样式系统

### 14.1 主组件样式

```scss
.timeLineContainer {
  width: 100%;
  height: 100%;
  cursor: pointer;
  display: flex;
  flex-direction: column;  // 纵向布局：Canvas在上，窗口列表在下

  .canvas {
    flex-grow: 0;     // 不拉伸
    flex-shrink: 0;   // 不压缩（保持设定高度）
  }

  .windowList {
    width: 100%;
    height: 100%;        // 填满剩余空间
    overflow: auto;       // 窗口多时可滚动
    overflow-x: hidden;   // 横向不滚动
    border-top: 1px solid rgba(153, 153, 153, 1);
    display: flex;
    flex-direction: column;

    &::-webkit-scrollbar {
      display: none;  // 隐藏滚动条
    }
  }
}
```

### 14.2 子窗口样式

```scss
.windowListItem {
  width: 100%;
  height: 30px;           // 每个窗口固定30px
  position: relative;
  border-bottom: 1px solid rgba(153, 153, 153, 1);
  user-select: none;

  &.active {
    background-color: v-bind(multiSegmentActiveColor);  // 动态绑定的CSS变量
  }

  .order {
    position: absolute;
    width: 30px; height: 30px;
    display: flex; justify-content: center; align-items: center;
    color: #fff;
    border-right: 1px solid rgba(153, 153, 153, 1);
  }
}
```

### 14.3 样式注入机制

使用 `vite-plugin-style-inject`，构建时将编译后的CSS代码注入到JS模块中：
```javascript
// 构建后的JS中会包含类似：
;(function(){
  const style = document.createElement('style')
  style.textContent = `.timeLineContainer{...}`
  document.head.appendChild(style)
})()
```
用户使用时无需手动引入CSS文件。

---

## 第十五部分：Demo 示例全集

### 15.1 基础用法 (Base.vue)

```javascript
// 功能：最简使用 + 定时器推进时间 + 重新渲染 + 跳转 + 切换分辨率
<TimeLine ref="TimelineRef" @timeChange="timeChange" />

onMounted(() => {
  // 每秒推进时间
  timer = setInterval(() => {
    time += 1000
    TimelineRef.value.setTime(time)
  }, 1000)
})

// 跳转到指定时间
TimelineRef.value.setTime('2021-01-01 00:00:00')

// 切换分辨率
TimelineRef.value.setZoom(6)

// 重新渲染
TimelineRef.value.reRender()
```

### 15.2 时间段显示 (Segment.vue)

```javascript
// 功能：显示时间段 + 点击时间段 + 点击空白 + 拖拽结束
<TimeLine
  :initTime="'2021-01-15 00:00:00'"
  :timeSegments="timeSegments"
  @timeChange="timeChange"
  @click_timeSegments="click_timeSegments"
  @click_timeline="onClickTimeLine"
  @dragTimeChange="onDragTimeChange"
/>

const timeSegments = [
  {
    name: '时间段1',
    beginTime: new Date('2021-01-13 10:00:00').getTime(),
    endTime: new Date('2021-01-14 23:00:00').getTime(),
    color: '#1a94bc',
    startRatio: 0.65,
    endRatio: 0.9
  },
  {
    name: '时间段2',
    beginTime: new Date('2021-01-15 02:00:00').getTime(),
    endTime: new Date('2021-01-15 18:00:00').getTime(),
    color: '#1a94bc',
    startRatio: 0.65,
    endRatio: 0.9
  }
]

const click_timeSegments = (arr, time, date, x) => {
  alert('点击了：' + arr[0].name)
}
```

### 15.3 多窗口时间轴 (MultiSegment.vue)

```javascript
// 功能：主轴+多个子轴 + 基础时间段 + 窗口时间段 + 窗口选中切换
<TimeLine
  :initTime="'2021-01-15 00:00:00'"
  :timeSegments="timeSegments3"          // 主轴时间段
  :windowList="windowList"               // 多窗口配置
  @click_timeSegments="..."
  @click_window_timeSegments="..."
/>

// 容器高度200px → 主轴50px（默认baseTimeLineHeight）+ 窗口区150px
// 每个窗口30px → 6个窗口=180px，会出现滚动

const windowList = [
  {
    name: '窗口1',
    timeSegments: [
      { name: '窗口1的时间段1', beginTime: ..., endTime: ..., color: '#FA3239', startRatio: 0.1, endRatio: 0.9 },
      { name: '窗口1的时间段2', beginTime: ..., endTime: ..., color: '#00AEFF', startRatio: 0.1, endRatio: 0.9 }
    ]
  },
  { name: '窗口2', timeSegments: [...] },
  { name: '窗口3' },  // 空窗口也可以
  { name: '窗口4' },
  { name: '窗口5' },
  { name: '窗口6' }
]
```

### 15.4 自定义元素定位 (Custom.vue)

```javascript
// 功能：watchTime追踪时间点位置，定位自定义图标
<TimeLine ref="Timeline4Ref" :initTime="..." :windowList="..." />
<i class="icon" ref="flagIcon" style="position:fixed" />
<i class="icon" ref="carIcon" style="position:fixed" />

onMounted(() => {
  // 监听页面滚动时更新位置
  window.addEventListener('scroll', () => {
    Timeline4Ref.value.updateWatchTime()
  })

  // 观察时间点1：在基础时间轴上
  Timeline4Ref.value.watchTime('2021-01-01 23:30:00', (x, y) => {
    if (x === -1 || y === -1) {
      flagIcon.value.style.display = 'none'
    } else {
      flagIcon.value.style.display = 'block'
      flagIcon.value.style.left = x + 'px'
      flagIcon.value.style.top = (y + 24) + 'px'  // 偏移避免遮挡
    }
  })

  // 观察时间点2：在第2个窗口时间轴上（第三个参数=2）
  Timeline4Ref.value.watchTime('2021-01-02 02:30:00', (x, y) => {
    if (x === -1 || y === -1) {
      carIcon.value.style.display = 'none'
    } else {
      carIcon.value.style.display = 'block'
      carIcon.value.style.left = x + 'px'
      carIcon.value.style.top = y + 'px'
    }
  }, 2)  // 指定在第2个窗口时间轴
})
```

### 15.5 年模式 (Year.vue)

```javascript
// 功能：10年跨度的时间段展示
<TimeLine
  :enableZoom="false"     // 禁止缩放
  :initZoomIndex="10"     // 年模式（87600小时=10年）
  :timeSegments="timeSegments"
/>

const timeSegments = [
  { beginTime: new Date('2021-01-13').getTime(), endTime: new Date('2025-01-14').getTime(), color: '#FA3239', startRatio: 0.65, endRatio: 0.9 },
  { beginTime: new Date('2008-01-01').getTime(), endTime: new Date('2021-01-15').getTime(), color: '#836ABB', startRatio: 0.65, endRatio: 0.9 }
]
```

### 15.6 年月模式 (YearMonth.vue)

```javascript
// 功能：365天跨度，刻度为月份
<TimeLine :enableZoom="false" :initZoomIndex="9" :timeSegments="..." />
```

### 15.7 自定义分辨率 (CustomZoom.vue)

```javascript
// 功能：扩展25小时分辨率 + 只显示当天时间 + 禁止拖拽和缩放
<TimeLine
  :enableZoom="false"
  :enableDrag="false"
  :showDateAtZero="false"
  :initZoomIndex="11"                    // 扩展的第一个索引
  :initTime="dayjs().format('YYYY-MM-DD 12:00:00')"  // 12点为中心
  :customShowTime="customShowTime"
  :extendZOOM="extendZOOM"
  :formatTime="formatTime"
  :hoverTimeFormat="hoverTimeFormat"
  @click_timeline="click_timeline"
/>

const extendZOOM = [{ zoom: 25, zoomHourGrid: 0.5 }]

// 自定义显示规则：只在索引11时处理
const customShowTime = (date, zoomIndex) => {
  if (zoomIndex === 11) {
    return date.getHours() % 2 === 0 && date.getMinutes() === 0
  }
}

// 格式化时间轴文字
const formatTime = (time) => {
  if (time.isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) return '24:00'
  if (time.hour() === 0 && time.minute() === 0 && time.millisecond() === 0) {
    return time.format('HH:mm')
  }
}

// 格式化hover时间
const hoverTimeFormat = (time) => {
  if (dayjs(time).isBefore(dayjs().format('YYYY-MM-DD 00:00:00')) ||
      dayjs(time).isAfter(dayjs().format('YYYY-MM-DD 23:59:59'))) {
    return ''  // 超出当天不显示
  }
  return dayjs(time).format('HH:mm:ss')
}
```

### 15.8 App.vue 示例切换器

```javascript
// 使用radio按钮 + 动态组件切换不同示例
<div class="switch-com">
  <div v-for="(item, i) in list" :key="i">
    <input type="radio" v-model="activeComp" :value="item.value" />
    <label>{{ item.name }}</label>
  </div>
</div>
<component :is="activeComp" />

const list = [
  { name: '基础用法', value: Base },
  { name: '显示时间段', value: Segment },
  { name: '多个时间轴', value: MultiSegment },
  { name: '显示自定义元素', value: Custom },
  { name: '显示到年', value: Year },
  { name: '显示到年月', value: YearMonth },
  { name: '自定义时间分辨率', value: CustomZoom }
]
```

---


## 第十六部分：AI 开发完整提示词模板

### 16.1 基础版提示词（快速开发）

```
请帮我开发一个基于 Vue 3 + Canvas 2D 的视频回放时间轴组件。

## 技术栈
- Vue 3 Composition API (script setup)
- Canvas 2D Context API
- dayjs 用于时间格式化
- TypeScript（可选）
- SCSS scoped 样式
- Vite lib 模式打包为 Vue 插件

## 核心数学模型
- 状态核心：startTimestamp（时间轴左端点时间戳）
- totalMS = ZOOM[currentZoomIndex] * 3600000（时间轴总毫秒数）
- PX_PER_MS = canvasWidth / totalMS（每毫秒对应像素数）
- currentTime = startTimestamp + totalMS / 2（中心点时间）
- 时间→像素：x = (timestamp - startTimestamp) * PX_PER_MS
- 像素→时间：timestamp = startTimestamp + x / PX_PER_MS

## 功能需求
1. Canvas绘制时间刻度线和文本，支持11级分辨率（半小时到10年）
2. 绘制彩色时间段（fillRect），支持自定义颜色和高度比例
3. 鼠标拖拽平移时间轴，滚轮缩放分辨率
4. 时间段点击检测（ctx.isPointInPath）
5. 中心竖线指示当前时间
6. hover时显示鼠标所在时间
7. 支持时间范围限制（timeRange.start/end）
8. 刻度对齐算法：msOffset = msPerGrid - (startTimestamp % msPerGrid)
9. 绘制顺序：时间段→刻度→中心线（层级正确）
10. 每次交互流程：修改startTimestamp → clearRect → 重绘全部

## 打包要求
- Vue 插件模式（app.use(TimeLine, { comName: 'xxx' })）
- 输出 ESM + UMD + d.ts
- CSS注入JS（无需单独引入CSS）
- external: ['vue']
```

### 16.2 完整版提示词（还原全部功能）

```
请帮我开发一个完整的视频监控回放时间轴 Vue 3 组件库，要求如下：

## 项目结构
src/components/TimeLine/
  ├── TimeLine.vue          主时间轴组件
  ├── WindowListItem.vue    多窗口子时间轴
  ├── constant.ts           常量（ZOOM/GRID/RULE数组）
  └── index.ts              Vue插件入口

## constant.ts 定义
- ONE_HOUR_STAMP = 3600000
- ZOOM = [0.5, 1, 2, 6, 12, 24, 72, 360, 720, 8760, 87600]
- ZOOM_HOUR_GRID = [1/60, 1/60, 2/60, 1/6, 0.25, 0.5, 1, 4, 4, 720, 7200]
- MOBILE_ZOOM_HOUR_GRID = [1/20, 1/30, 1/20, 1/3, 0.5, 2, 4, 4, 4, 720, 7200]
- ZOOM_DATE_SHOW_RULE：每个分辨率对应一个(date)=>boolean判断函数
- MOBILE_ZOOM_DATE_SHOW_RULE：移动端对应版本

## Props（26个）
1. initTime: Number|String（初始中心时间，默认当天0点）
2. timeRange: Object { start, end }（时间限制范围）
3. initZoomIndex: Number（默认5，即24小时）
4. showCenterLine: Boolean（默认true）
5. centerLineStyle: Object { width:2, color:'#fff' }
6. textColor: String（刻度文字颜色）
7. hoverTextColor: String（hover文字颜色）
8. lineColor: String（刻度线颜色）
9. lineHeightRatio: Object { date:0.3, time:0.2, none:0.1, hover:0.3 }
10. showHoverTime: Boolean（默认true）
11. hoverTimeFormat: Function (time)=>string
12. timeSegments: Array [{ beginTime, endTime, color, startRatio, endRatio }]
13. backgroundColor: String（默认'#262626'）
14. multiSegmentActiveColor: String
15. enableZoom: Boolean（默认true）
16. enableDrag: Boolean（默认true）
17. windowList: Array [{ name, timeSegments }]
18. baseTimeLineHeight: Number（默认50）
19. initSelectWindowTimeLineIndex: Number（默认-1）
20. isMobile: Boolean（默认false）
21. maxClickDistance: Number（默认3px，区分点击和拖拽）
22. roundWidthTimeSegments: Boolean（默认true，四舍五入防间隙）
23. customShowTime: Function (date, zoomIndex)=>boolean|undefined
24. showDateAtZero: Boolean（默认true）
25. extendZOOM: Array [{ zoom, zoomHourGrid, mobileZoomHourGrid }]
26. formatTime: Function (dayjsObj)=>string|''

## Events（8个）
1. timeChange(currentTime) — 每次draw后
2. mousedown(event) — 按下
3. mouseup(event) — 松开
4. dragTimeChange(currentTime) — 拖拽结束
5. click_timeSegments(segments[], time, date, x) — 点击时间段
6. click_timeline(time, date, x) — 点击空白
7. change_window_time_line(index, item) — 切换窗口
8. click_window_timeSegments(segments[], index, item) — 点击窗口时间段

## Expose方法（4个）
1. setTime(t) — 设置当前时间（拖动中忽略）
2. setZoom(index) — 设置分辨率（以当前时间为锚点）
3. watchTime(time, callback, windowIndex?) — 观察时间点位置
4. reRender() — 完整重置重渲染

## 核心算法
1. 刻度对齐：msOffset = msPerGrid - (startTimestamp % msPerGrid)
2. 年/月模式修正：adjustMsOffset对齐到年初/月初
3. 拖拽：newStart = cacheStart - round(diffX / PX_PER_MS)
4. 缩放锚点：startTimestamp = currentTime - newTotalMS / 2
5. hitTest：drawTimeSegments(callback, path=true) + ctx.isPointInPath
6. 时间段宽度最小1px：w = Math.max(1, w)
7. 左边界裁剪：x<0时设x=0，w用endTime-startTimestamp计算

## 布局
- 外层flex column，Canvas不伸缩
- 多窗口时Canvas高度=baseTimeLineHeight，剩余空间给windowList
- 单窗口时Canvas高度=容器全高
- 每个WindowListItem固定30px高

## 样式
- scoped SCSS
- 深色主题默认（背景#262626，文字rgba(151,158,167,1)）
- 隐藏windowList滚动条
- cursor: pointer/grabbing切换
```

### 16.3 增量功能提示词

#### 只开发时间段功能
```
在已有的Vue3 Canvas时间轴基础上，添加时间段绘制功能：
- 数据格式：{ beginTime, endTime, color, startRatio(默认0.6), endRatio(默认0.9) }
- 绘制用ctx.fillRect
- 坐标四舍五入防间隙
- 最小宽度1px
- 左边界裁剪处理
- 点击检测用ctx.rect + isPointInPath
- watch深监听timeSegments变化时reRender
```

#### 只开发多窗口功能
```
在已有的Vue3时间轴基础上，添加多窗口时间轴功能：
- 触发条件：windowList.length > 1
- 布局：主Canvas + 下方可滚动的窗口列表
- 每个窗口独立Canvas，共享startTimestamp和totalMS
- 主组件draw()时调用每个子组件的draw()
- 子组件点击事件冒泡到父组件
- 支持active选中状态切换
```

#### 只开发watchTime功能
```
添加watchTime时间点观察功能：
- API: watchTime(time, callback, windowIndex?)
- 内部维护watchTimeList数组
- 每次draw()后遍历列表计算每个时间点的像素位置
- 超出可视范围返回(-1,-1)
- 在范围内返回(x+canvasLeft, y)相对视口坐标
- windowIndex可指定定位到第几个窗口时间轴的y坐标
```

---

## 第十七部分：开发注意事项与陷阱

### 17.1 性能相关

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| hover时频繁重绘 | 每次mousemove都clearRect+draw | 可优化为双Canvas（底层静态+顶层动态） |
| 大量时间段卡顿 | forEach遍历所有段 | 可增加二分查找可视范围段 |
| resize性能 | 整个init+draw流程 | 可debounce处理 |
| timeChange事件频率过高 | hover也触发draw()→emit | 业务层应throttle处理 |

### 17.2 常见Bug与规避

| 陷阱 | 说明 | 规避 |
|------|------|------|
| Canvas模糊 | 未设置canvas.width/height属性 | 必须设置DOM属性而非CSS |
| 时间段间隙 | 浮点数亚像素渲染 | roundWidthTimeSegments=true |
| 全局数组污染 | extendZOOM直接push到模块级数组 | 多实例会累积，应在onBeforeUnmount中pop |
| mouseup丢失 | 鼠标滑出Canvas后松开 | 监听window.mouseup |
| 拖动中setTime | 定时器setTime与拖动冲突 | setTime开头检查mousedown状态 |
| 年/月模式刻度不均匀 | 月份天数不等 | adjustMsOffset修正 |
| touchend无touches | 触摸结束时touches为空 | 需要使用changedTouches |

### 17.3 架构改进建议

| 方面 | 当前方式 | 可优化为 |
|------|---------|---------|
| 绘制策略 | 全量重绘 | 脏区域重绘或双Canvas |
| 状态管理 | 单个reactive对象 | 可拆分为多个composable |
| 时间段检索 | 全量遍历 | 区间树或二分查找 |
| 事件系统 | 直接window监听 | 可用PointerEvent统一触摸/鼠标 |
| 类型安全 | script setup无类型 | 添加完整TypeScript类型 |
| 常量扩展 | 修改全局数组 | 使用实例级副本 |
| 测试 | 无 | Canvas测试用jest-canvas-mock |

### 17.4 典型应用场景

| 场景 | 关键配置 |
|------|---------|
| 24小时监控回放 | initZoomIndex=5, timeSegments=录像段 |
| 多通道NVR回放 | windowList=各通道, timeSegments=总录像段 |
| 只读时间线展示 | enableDrag=false, enableZoom=false |
| 移动端使用 | isMobile=true |
| 只显示当天 | extendZOOM=[{zoom:25}], enableDrag=false |
| 长期数据概览 | initZoomIndex=9/10(年/10年模式) |
| 精确到秒的操作 | initZoomIndex=0(半小时), 配合setTime |

### 17.5 关键设计模式总结

1. **全量重绘模式**：任何状态变化 → clearRect → 依次绘制所有图层
2. **锚点缩放**：缩放时保持中心时间不变
3. **拖拽缓存**：mousedown时缓存startTimestamp，避免累积误差
4. **路径重用检测**：绘制和hitTest共用drawTimeSegments函数（path参数切换）
5. **观察者模式**：watchTime注册回调，每帧自动通知
6. **Props深监听**：timeSegments变化触发完整reRender
7. **事件分发**：按距离区分click/drag，优先检测timeSegments点击

---

## 附录A：完整源码文件引用

### TimeLine.vue 核心函数索引

| 函数名 | 行为 |
|--------|------|
| setInitData | 设置初始数据（时间、分辨率、窗口列表） |
| fixStartTimestamp | 根据timeRange边界修正startTimestamp |
| init | 获取容器尺寸，创建Canvas context |
| draw | 主绘制入口（顺序：时间段→刻度→中心线） |
| updateWatchTime | 遍历watchTimeList计算像素位置 |
| drawMiddleLine | 绘制中心竖线 |
| addGraduations | 绘制所有刻度线和文字 |
| checkShowTime | 判断某时间点是否显示文字 |
| drawTimeSegments | 绘制/检测时间段 |
| graduationTitle | 格式化刻度文字 |
| drawLine | Canvas画线工具函数 |
| onPointerdown | 统一按下处理 |
| onPointermove | 统一移动处理 |
| onPointerup | 统一松开处理（区分click/drag） |
| drag | 拖拽逻辑（计算新startTimestamp） |
| hoverShow | hover时间显示 |
| onMouseweel | 滚轮缩放 |
| onClick | 点击处理（检测时间段→分发事件） |
| getClickTimeSegments | Canvas hitTest |
| getClientOffset | 坐标转换（视口→容器） |
| clearCanvas | 清除画布 |
| setTime | [Expose] 设置当前时间 |
| setZoom | [Expose] 设置分辨率 |
| watchTime | [Expose] 注册时间观察 |
| reRender | [Expose] 完整重渲染 |
| reset | 重置所有状态 |
| toggleActive | 切换窗口选中 |
| onWindowListScroll | 窗口列表滚动时更新watchTime |
| onResize | 容器尺寸变化处理 |

---

## 附录B：数据流图

```
┌─────────── 用户交互 ───────────┐
│ mousedown/touchstart            │
│ mousemove/touchmove             │
│ mouseup/touchend                │
│ mousewheel                      │
└─────────┬───────────────────────┘
          ↓
┌─────────── 状态更新 ───────────┐
│ startTimestamp (核心)           │
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
│   ├── WindowListItem.draw()     │
│   └── updateWatchTime()         │
└─────────────────────────────────┘
          ↓
┌─────────── 输出 ──────────────┐
│ Canvas像素渲染                  │
│ 事件回调（timeChange等）        │
│ watchTime位置回调               │
└─────────────────────────────────┘
```

---

## 附录C：TypeScript 完整类型定义（建议新项目使用）

```typescript
// types.ts
export interface TimeSegment {
  name?: string
  beginTime: number
  endTime?: number
  color: string
  startRatio?: number  // 默认0.6
  endRatio?: number    // 默认0.9
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
  width?: number   // 默认2
  color?: string   // 默认'#fff'
}

export interface LineHeightRatio {
  date?: number    // 默认0.3
  time?: number    // 默认0.2
  none?: number    // 默认0.1
  hover?: number   // 默认0.3
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
  formatTime?: (time: import('dayjs').Dayjs) => string | ''
}

export interface TimeLineExpose {
  setTime: (t: number | string) => void
  setZoom: (index: number) => void
  watchTime: (time: number | string, callback: (x: number, y: number) => void, windowTimeLineIndex?: number) => void
  reRender: () => void
}

export interface TimeLineEmits {
  timeChange: (currentTime: number) => void
  mousedown: (event: Event) => void
  mouseup: (event: Event) => void
  dragTimeChange: (currentTime: number) => void
  click_timeSegments: (segments: TimeSegment[], time: number, date: string, x: number) => void
  click_timeline: (time: number, date: string, x: number) => void
  change_window_time_line: (index: number, item: WindowItem) => void
  click_window_timeSegments: (segments: TimeSegment[], index: number, item: WindowItem) => void
}
```

---

*文档生成日期：2026-05-21*
*源项目：https://github.com/SnowBeatRain/vue3-time-line*
*npm包：@boyzcf/vue3-time-line@1.0.1*
