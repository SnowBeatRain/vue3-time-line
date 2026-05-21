# Skill: Vue3 组件库打包与发布

## 概述
指导 AI 如何从零搭建一个 Vue 3 组件库项目，包含打包配置、类型声明生成、Vue 插件注册模式和 npm 发布配置。

---

## 项目结构模板

```
my-component/
├── src/
│   └── components/MyComponent/
│       ├── MyComponent.vue     # 主组件
│       ├── index.ts            # Vue 插件入口
│       ├── constant.ts         # 常量配置
│       └── package.json        # 组件级 package.json（可选）
├── lib/                        # 构建产物
│   ├── my-component.mjs        # ESM
│   ├── my-component.umd.js     # UMD
│   └── my-component.d.ts       # 类型声明
├── package.json
├── vite.config.js
├── tsconfig.json
└── tsconfig.app.json
```

---

## Vue 插件注册模式

```typescript
// index.ts
import MyComponentVue from './MyComponent.vue'

interface IOptions {
  comName?: string
}

const MyComponent = {
  install(app: any, options?: IOptions) {
    const name = options?.comName || 'MyComponent'
    app.component(name, MyComponentVue)
  }
}
export default MyComponent
```

使用方式：
```typescript
import MyComponent from 'my-component'
app.use(MyComponent, { comName: 'MyComponent' })
```

---

## Vite lib 模式配置

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import dts from 'vite-plugin-dts'
import VitePluginStyleInject from 'vite-plugin-style-inject'
import { resolve } from 'path'

export default defineConfig(({ command, mode }) => {
  const config = {
    plugins: [
      vue(),
      dts({ tsconfigPath: 'tsconfig.app.json', rollupTypes: true, copyDtsFiles: true }),
    ],
    build: {
      outDir: 'lib',
      rollupOptions: {
        external: ['vue'],
        output: { exports: 'named', globals: { vue: 'Vue' } }
      },
      lib: {
        entry: resolve(__dirname, './src/components/MyComponent/index.ts'),
        name: 'my-component',
        fileName: 'my-component'
      }
    }
  }

  if (mode === 'lib') {
    config.plugins.push(VitePluginStyleInject())
  } else if (mode === 'demo') {
    delete config.build.lib
    config.base = './'
    config.build.outDir = 'dist'
  }
  return config
})
```

---

## package.json 配置

```json
{
  "name": "@scope/my-component",
  "version": "1.0.0",
  "main": "./lib/my-component.umd.js",
  "module": "./lib/my-component.mjs",
  "types": "./lib/my-component.d.ts",
  "exports": {
    ".": {
      "import": "./lib/my-component.mjs",
      "require": "./lib/my-component.umd.js"
    }
  },
  "files": ["lib"],
  "peerDependencies": {
    "vue": "^3.3.0"
  },
  "scripts": {
    "build": "vite build --mode lib",
    "build:demo": "vite build --mode demo",
    "dev": "vite"
  }
}
```

---

## 关键决策点

| 决策 | 推荐做法 | 原因 |
|------|---------|------|
| CSS 处理 | vite-plugin-style-inject | 用户无需手动引入 CSS |
| 类型声明 | vite-plugin-dts + rollupTypes | 合并为单个 .d.ts 文件 |
| Vue 外部化 | external: ['vue'] | 避免打包两份 Vue |
| 输出格式 | ESM + UMD | 兼容 import 和 require |
| 组件名 | 通过 options 可配置 | 避免命名冲突 |

---

## 注意事项

1. Canvas 等 DOM 尺寸必须在 onMounted 后获取（getBoundingClientRect）
2. 全局事件（window.addEventListener）必须在 onBeforeUnmount 中清理
3. defineExpose 暴露的方法对外部通过 ref 调用
4. watch + deep: true 监听复杂 props 变化
5. nextTick 确保 DOM 更新后再操作 Canvas
