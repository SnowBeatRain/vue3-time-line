# Skill: Canvas 交互式可视化组件开发模式

## 概述
指导 AI 开发基于 Canvas 2D 的交互式可视化组件（时间轴、波形图、甘特图等），包含绘制循环、事件处理、坐标转换和性能优化的通用模式。

---

## Canvas 组件通用架构

```
┌─────────────────────────────────────┐
│ 容器 div（flex, 100%宽高）          │
│   ├── Canvas（flex-shrink:0）        │
│   └── 其他 DOM 区域（可选）          │
└─────────────────────────────────────┘
```

### 初始化流程
```javascript
onMounted(() => {
  const { width, height } = container.getBoundingClientRect()
  canvas.width = width       // 必须设DOM属性，非CSS
  canvas.height = height
  ctx = canvas.getContext('2d')
  draw()
  window.addEventListener('mouseup', onMouseup)
  window.addEventListener('resize', onResize)
})
```

---

## 全量重绘模式

```javascript
function update() {
  ctx.clearRect(0, 0, width, height)
  drawBackground()   // 背景层
  drawData()         // 数据层
  drawUI()           // UI层（指示线等）
}
```

适用场景：Canvas面积小（<500px高），数据量适中（<1000个元素）

---

## 坐标转换

### 浏览器坐标 → Canvas 坐标
```javascript
function getCanvasPos(event) {
  const { left, top } = canvas.getBoundingClientRect()
  return [event.clientX - left, event.clientY - top]
}
```

### 数据坐标 → 像素坐标（线性映射）
```javascript
function dataToPixel(value, dataMin, dataMax, pixelMin, pixelMax) {
  return pixelMin + (value - dataMin) / (dataMax - dataMin) * (pixelMax - pixelMin)
}
```

---

## 事件处理模式

### 点击/拖拽区分
```javascript
const MAX_CLICK_DISTANCE = 3
onPointerDown: 记录 mousedownX/Y + 缓存状态
onPointerUp: distance <= 3 → onClick, 否则 → onDragEnd
```

### 全局 mouseup
mouseup 必须在 window 上监听，防止拖出 Canvas 后无法松开。

### cursor 切换
默认 `pointer`，按下 `grabbing`，松开恢复 `pointer`

---

## hitTest 点击检测

### ctx.isPointInPath（推荐）
```javascript
ctx.beginPath()
ctx.rect(x, y, w, h)
if (ctx.isPointInPath(clickX, clickY)) { /* 命中 */ }
```

每个元素都需要 `beginPath()` 隔离路径。

---

## 缩放模式

### 中心锚点缩放
```javascript
onWheel(delta) {
  changeZoomLevel(delta)
  offset = currentCenter - newRange / 2
}
```

---

## 性能优化策略

| 策略 | 做法 |
|------|------|
| 双Canvas | 底层静态 + 顶层动态（hover） |
| 可见性裁剪 | 只绘制视口范围内的元素 |
| requestAnimationFrame | 连续动画用 rAF |
| 四舍五入 | Math.round 防亚像素模糊和间隙 |

---

## 移动端适配

1. 触摸事件用 `e.touches[0]` 获取坐标
2. touchend 用 `changedTouches`（touches为空）
3. 格子间距更大，文字密度更低
4. 全局用 touchend 替代 mouseup
5. 阻止滚动：`@touchmove.prevent`

---

## Canvas 文字工具

```javascript
const textWidth = ctx.measureText(text).width
ctx.fillText(text, x - textWidth / 2, y)  // 居中
ctx.font = '12px Arial'
ctx.fillStyle = '#999'
```

---

## 通用 drawLine

```javascript
function drawLine(ctx, x1, y1, x2, y2, lineWidth = 1, color = '#fff') {
  ctx.beginPath()
  ctx.strokeStyle = color
  ctx.lineWidth = lineWidth
  ctx.moveTo(x1, y1)
  ctx.lineTo(x2, y2)
  ctx.stroke()
}
```
