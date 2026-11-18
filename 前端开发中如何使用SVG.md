# 前端开发中的 SVG 使用指南

SVG（Scalable Vector Graphics）是基于 XML 的矢量图形格式，也是 DOM 的一部分。它既能当图片用，又能当 DOM 节点用，这是它与 PNG/JPG 最大的区别。

## 一、引入 SVG 的几种方式

### 1. `<img>` 标签

```html
<img src="logo.svg" alt="Logo" width="120" height="40">
```

- 优点：简单，可被浏览器缓存，不污染 DOM
- 缺点：**无法用外部 CSS/JS 控制内部元素**，也不能改颜色，内部脚本不会执行
- 适用：Logo、插画等静态图片

### 2. CSS 背景

```css
.icon {
  background: url("icon.svg") no-repeat center / contain;
}
```

限制与 `<img>` 类似，可用 `data URI` 内联减少请求：

```css
.icon {
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' ...%3E");
}
```

注意：data URI 里必须带 `xmlns`，`#` 要转义成 `%23`。

### 3. 内联 SVG（最灵活）

```html
<svg width="100" height="100" viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg">
  <circle cx="50" cy="50" r="40" fill="tomato" />
</svg>
```

- 优点：可用 CSS 和 JS 完全控制，支持动画与交互，零额外请求
- 缺点：会增加 HTML 体积，不能单独缓存
- 适用：图标、需要交互或动画的图形、数据可视化

### 4. `<object>` / `<embed>` / `<iframe>`

```html
<object type="image/svg+xml" data="chart.svg"></object>
```

SVG 内部脚本可以运行，也能通过 `contentDocument` 访问，但现在用得很少，通常被内联方案取代。

### 5. Sprite + `<use>`（图标系统常用）

```html
<!-- 定义（通常放在页面顶部，隐藏） -->
<svg style="display:none">
  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="M3 12l9-9 9 9v9H3z"/>
  </symbol>
</svg>

<!-- 使用 -->
<svg class="icon"><use href="#icon-home"/></svg>
```

也可以引用外部文件：`<use href="sprite.svg#icon-home"/>`（需同源）。

注意：新代码用 `href`，老的 `xlink:href` 已废弃。

### 6. CSS mask（单色图标的好方案）

```css
.icon {
  width: 24px; height: 24px;
  background-color: currentColor;
  mask: url(icon.svg) no-repeat center / contain;
}
```

这样既能像 `<img>` 一样引用文件，又能通过 `background-color` 改色。

## 二、核心概念

### viewBox：最重要的属性

```html
<svg width="200" height="200" viewBox="0 0 100 100">
```

`viewBox="minX minY width height"` 定义内部坐标系。上面的例子里，100×100 的坐标系被缩放填满 200×200 的显示区域。

- 只设 `viewBox`、不设宽高 → 自适应容器，CSS 控制尺寸，**响应式首选**
- `preserveAspectRatio` 控制宽高比不一致时的对齐与缩放（默认 `xMidYMid meet`）

### 常用基础元素

| 元素 | 说明 |
|---|---|
| `<rect>` `<circle>` `<ellipse>` `<line>` `<polyline>` `<polygon>` | 基本形状 |
| `<path>` | 万能路径，`d` 属性用 M/L/C/Q/A/Z 等指令绘制 |
| `<text>` `<tspan>` | 文本 |
| `<g>` | 分组，便于统一变换和样式 |
| `<defs>` | 定义可复用对象（渐变、滤镜、clipPath 等） |
| `<use>` | 复用定义好的元素 |
| `<symbol>` | 带 viewBox 的可复用模板 |

### 填充与描边

```html
<path d="M10 80 Q 52 10 95 80" fill="none" stroke="#333" stroke-width="2"
      stroke-linecap="round" stroke-linejoin="round" />
```

`fill`、`stroke`、`stroke-width`、`stroke-dasharray`、`opacity` 等既是属性，也能写进 CSS。

## 三、用 CSS 控制 SVG

### 1. 用 `currentColor` 继承文字颜色

```html
<svg class="icon" viewBox="0 0 24 24" fill="currentColor">
  <path d="..."/>
</svg>
```

```css
.btn { color: #666; }
.btn:hover { color: #1677ff; }  /* 图标跟着变色 */
.icon { width: 1em; height: 1em; vertical-align: -0.125em; }
```

这是图标颜色管理最简洁的方式，建议图标统一用 `currentColor`。

### 2. CSS 变量穿透 `<use>` 的 Shadow DOM

`<use>` 引用的内容处于 shadow tree，外部选择器选不到内部元素，但**CSS 变量和继承属性可以穿透**：

```html
<symbol id="icon-heart" viewBox="0 0 24 24">
  <path d="..." fill="var(--icon-primary, currentColor)"/>
  <circle fill="var(--icon-secondary, #ccc)" .../>
</symbol>
```

```css
.icon-theme { --icon-primary: #e11d48; --icon-secondary: #fecdd3; }
```

做双色图标和主题切换时很实用。

### 3. 常见样式差异

- 用 `fill`/`stroke` 而不是 `color`/`background`/`border`
- 不支持 `margin`、`padding` 作用于内部元素
- `transform-origin` 默认是 SVG 坐标系原点，需要时加 `transform-box: fill-box; transform-origin: center;`

```css
.gear {
  transform-box: fill-box;
  transform-origin: center;
  animation: spin 2s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
```

## 四、SVG 动画

### 1. CSS 动画（首选）

经典的**描边动画**：

```html
<svg viewBox="0 0 100 100">
  <path class="draw" d="M10 50 Q 50 0 90 50 T 170 50" fill="none" stroke="#333" stroke-width="3"/>
</svg>
```

```css
.draw {
  stroke-dasharray: 300;   /* 设为路径总长度，可用 path.getTotalLength() 获取 */
  stroke-dashoffset: 300;
  animation: draw 2s ease forwards;
}
@keyframes draw { to { stroke-dashoffset: 0; } }
```

也可以给 `path` 设 `pathLength="1"`，这样 dasharray 直接写 1，不用计算真实长度。

### 2. SMIL（SVG 原生动画）

```html
<circle r="10" cx="20" cy="50">
  <animate attributeName="cx" from="20" to="180" dur="2s" repeatCount="indefinite"/>
</circle>
```

不需要 JS，写 loading 动画很方便。Chrome 曾宣布要废弃但后来撤回，目前各浏览器都支持，不过社区更倾向 CSS 或 JS 方案。

### 3. JavaScript 动画库

- **GSAP**：功能强大，有 MorphSVG、DrawSVG 等插件
- **anime.js**、**Framer Motion**（React）、**Motion One**
- 路径变形（morphing）要求前后路径的点数、结构一致，否则需要借助专门的插值库（如 flubber）

### 4. 性能提示

- 优先动画 `transform` 和 `opacity`
- 动画 `d`、`width` 等属性会触发重绘，路径复杂时可能卡顿
- 大量元素的动画建议考虑 Canvas

## 五、JavaScript 操作 SVG

SVG 元素需要用**命名空间**创建：

```js
const NS = 'http://www.w3.org/2000/svg';
const svg = document.createElementNS(NS, 'svg');
svg.setAttribute('viewBox', '0 0 100 100');

const circle = document.createElementNS(NS, 'circle');
circle.setAttribute('cx', 50);
circle.setAttribute('cy', 50);
circle.setAttribute('r', 30);
svg.appendChild(circle);
document.body.appendChild(svg);
```

常用 API：

```js
path.getTotalLength();            // 路径总长度
path.getPointAtLength(100);       // 路径上某长度处的点坐标
el.getBBox();                     // 元素包围盒（文字宽度测量也常用）
svg.createSVGPoint();             // 屏幕坐标 ↔ SVG 坐标转换
```

事件绑定和普通 DOM 一样，`addEventListener('click', ...)` 即可。需要点击穿透时用 `pointer-events` 控制。

## 六、主流框架中的使用

### React

**方式 A：作为组件导入（SVGR）**

```jsx
import Logo from './logo.svg?react';   // Vite + vite-plugin-svgr
// 或 CRA / Webpack: import { ReactComponent as Logo } from './logo.svg';

<Logo className="logo" width={40} />
```

**方式 B：直接写 JSX**

```jsx
const Icon = ({ size = 24, ...props }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="currentColor" {...props}>
    <path d="..." />
  </svg>
);
```

JSX 中属性要用驼峰：`stroke-width` → `strokeWidth`，`class` → `className`，`fill-rule` → `fillRule`。

### Vue

```js
// vite.config.js
import svgLoader from 'vite-svg-loader';
export default { plugins: [vue(), svgLoader()] };
```

```vue
<script setup>
import IconHome from './home.svg?component';
</script>
<template><IconHome class="icon" /></template>
```

### 通用方案：Sprite 自动化

- Vite：`vite-plugin-svg-icons`
- Webpack：`svg-sprite-loader`
- 把 `icons/` 目录下的 SVG 自动合并成 symbol sprite，用 `<Icon name="home" />` 统一调用

### 图标库

Lucide、Heroicons、Iconify、Ant Design Icons、Material Symbols 等，大多以 SVG 组件形式提供，支持按需引入（tree-shaking）。

## 七、优化

### 1. 用 SVGO 压缩

设计工具（Figma、Illustrator、Sketch）导出的 SVG 往往带有大量冗余信息，用 SVGO 能减小一半甚至更多：

```bash
npx svgo input.svg -o output.svg
```

配置示例：

```js
// svgo.config.js
module.exports = {
  plugins: [
    { name: 'preset-default', params: { overrides: { removeViewBox: false } } },
    'removeDimensions',  // 去掉 width/height，保留 viewBox 方便响应式
  ],
};
```

> **注意**：不要让 SVGO 删掉 `viewBox`，否则无法缩放。多个内联 SVG 里的 `id` 可能冲突（渐变、clipPath 等），要启用 `prefixIds` 或手动加前缀。

### 2. 体积与性能

- 节点数过多（比如几千个）会拖慢 DOM，改用 Canvas 或 WebGL
- 简化路径精度（小数位保留 1–2 位足够）
- 开启 gzip/brotli，SVG 作为文本压缩比很高
- 小图标内联，大插画用 `<img>` 以便缓存

### 3. 渲染问题

- 1px 线条模糊：坐标取 0.5 对齐像素，或设置 `shape-rendering="crispEdges"`
- 高 DPI 屏幕通常无需处理，SVG 本身就是矢量

## 八、无障碍（a11y）

```html
<!-- 有意义的图形 -->
<svg role="img" aria-labelledby="t d" viewBox="...">
  <title id="t">销售额趋势图</title>
  <desc id="d">2024 年各季度销售额逐步增长</desc>
  ...
</svg>

<!-- 纯装饰图标 -->
<svg aria-hidden="true" focusable="false" ...>...</svg>

<!-- 图标按钮：用文字说明按钮用途 -->
<button aria-label="关闭"><svg aria-hidden="true">...</svg></button>
```

要点：

- 装饰性图标隐藏，避免读屏软件干扰
- 有语义的图形提供 `<title>` / `<desc>`
- 不要只靠颜色传达信息

## 九、进阶特性

### 渐变与图案

```html
<defs>
  <linearGradient id="g" x1="0" y1="0" x2="1" y2="1">
    <stop offset="0" stop-color="#6366f1"/>
    <stop offset="1" stop-color="#ec4899"/>
  </linearGradient>
</defs>
<rect width="100" height="100" fill="url(#g)"/>
```

### 裁剪与蒙版

```html
<clipPath id="c"><circle cx="50" cy="50" r="40"/></clipPath>
<image href="photo.jpg" clip-path="url(#c)" width="100" height="100"/>
```

`mask` 支持半透明（按亮度或透明度），`clipPath` 是硬裁剪。

### 滤镜

```html
<filter id="blur"><feGaussianBlur stdDeviation="3"/></filter>
<circle r="20" filter="url(#blur)"/>
```

`feTurbulence`、`feDisplacementMap` 等可做噪点、水波、扭曲效果。滤镜较耗性能，慎用于大面积或持续动画。

### foreignObject

在 SVG 内嵌入 HTML：

```html
<foreignObject x="10" y="10" width="200" height="100">
  <div xmlns="http://www.w3.org/1999/xhtml">可以放 HTML</div>
</foreignObject>
```

适合图表里的富文本标签。导出为图片时可能有兼容问题。

## 十、数据可视化

SVG 非常适合中小规模的数据可视化，因为每个图形元素都是 DOM 节点，可以绑定事件、做 CSS 动画：

- **D3.js**：底层可视化库，数据与 DOM 绑定，自由度最高
- **ECharts**、**Highcharts**、**Recharts**、**Visx**：封装好的图表库，部分支持 SVG 渲染模式
- 数据量过大（上万个元素）时，Canvas/WebGL 性能更优

D3 简单示例：

```js
d3.select('svg')
  .selectAll('rect')
  .data([30, 80, 45, 60])
  .join('rect')
  .attr('x', (d, i) => i * 30)
  .attr('y', d => 100 - d)
  .attr('width', 25)
  .attr('height', d => d)
  .attr('fill', 'steelblue');
```

## 十一、常见坑

1. **缺少 `viewBox`**：无法缩放，响应式失效
2. **`<img>` 引入的 SVG 无法被 CSS 改色**：需要改色就内联或用 mask
3. **跨域问题**：`<use href="https://other.com/sprite.svg#id">` 受同源限制
4. **id 冲突**：多个内联 SVG 带相同 `id`（如 `gradient1`），后者会错用前者
5. **Safari 兼容**：部分 `transform-origin`、`filter` 行为与 Chrome 不一致，需实测
6. **文字渲染**：SVG 中的 `<text>` 不会自动换行，需手动拆行或用 `foreignObject`
7. **服务端渲染 / 邮件**：邮件客户端大多不支持内联 SVG，需降级为 PNG
8. **`svg` 默认是 inline 元素**：下方可能出现空隙，设 `display: block` 或 `vertical-align: middle`
9. **安全**：不要直接渲染用户上传的 SVG，里面可能含 `<script>` 和事件属性（XSS）。需要用 DOMPurify 等净化，或只通过 `<img>` 展示

## 十二、选型建议

| 场景 | 推荐方案 |
|---|---|
| Logo、静态插画 | `<img src>` |
| UI 图标，需要换色 | 内联 SVG / 组件化 / Sprite，配合 `currentColor` |
| 大量重复图标 | Sprite + `<use>` 或图标库 |
| 复杂动画、交互 | 内联 SVG + CSS/GSAP |
| 单色图标但想用文件引用 | CSS `mask` |
| 中小规模图表 | SVG（D3 / ECharts SVG 模式） |
| 海量数据点、实时渲染 | Canvas / WebGL |
| 用户上传内容展示 | `<img>` 或净化后再内联 |
