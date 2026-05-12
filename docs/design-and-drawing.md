# 设计与绘制说明

本文档记录 `index.html` 的设计策略、CSS 绘图方法、交互结构和碳核算实现，方便后续继续改造成完整产品。

## 1. 设计目标

这个页面不是传统后台仪表盘，而是一个售前演示型工具箱。目标是让企业客户在 30 秒内理解三件事：

- 能碳管理可以从最简单的活动数据开始。
- 输入电、气、油、绿电和产值后，可以立即得到碳排、碳强度和目标进度。
- 小工具适合讲清逻辑，正式落地需要 EMS 系统完成采集、存证、核查和报表闭环。

## 2. 视觉风格

参考对象是 `https://shoppu.mochi.at/`，迁移时提炼了以下特征：

- 奶油色纸张背景。
- 黑色粗描边。
- 硬投影，而不是柔和阴影。
- 红色作为强提醒和价格贴纸。
- 黄色作为促销贴纸和按钮高亮。
- 商品货架式功能卡片。
- 横向滚动公告条。
- 复古目录感，而不是 SaaS 默认蓝白大屏。

核心变量集中在 CSS `:root`：

```css
:root {
  --cream: #fff3d8;
  --paper: #fffaf0;
  --ink: #111111;
  --red: #ef3d2f;
  --green: #2f8f5b;
  --blue: #2867b2;
  --yellow: #ffd447;
  --line: 3px solid var(--ink);
  --shadow: 7px 7px 0 var(--ink);
  --radius: 8px;
}
```

后续按钮、卡片、输入框、图表、报告卡都复用这组变量，保证页面统一。

## 3. 页面结构

页面按“先吸引、再理解、再动手、再输出”的顺序组织：

```text
header.topbar       顶部品牌和导航
.marquee            横向滚动公告
.hero               首屏介绍和 CSS 工厂插图
.kpis               四个核心指标
#tools              工具货架，展示能力边界
#calculator         输入区和图表区
#report             自动报告摘要
#code               核心构建代码
footer              EMS 落地说明
```

这种结构适合公众号、售前演示和现场沟通。用户不需要先理解标准术语，先看到“这个工具能算什么”。

## 4. CSS 绘图方法

页面没有使用图片绘制工厂，而是用 HTML 元素和 CSS 组合完成。这样做的好处是可离线运行、可缩放、可快速改色。

### 4.1 工厂主体

工厂区域是 `.factory-body`，使用边框和背景网格制造工业感：

```css
.factory-body {
  position: relative;
  border: var(--line);
  background:
    linear-gradient(90deg, rgba(17,17,17,.1) 1px, transparent 1px) 0 0 / 26px 100%,
    #d7ead9;
  overflow: hidden;
}
```

### 4.2 烟囱

烟囱是一个绝对定位矩形，用左右和顶部边框塑形：

```css
.chimney {
  position: absolute;
  left: 10%;
  bottom: 0;
  width: 58px;
  height: 70%;
  border-right: var(--line);
  border-left: var(--line);
  border-top: var(--line);
  background: #f2a65a;
}
```

### 4.3 厂房屋顶

屋顶通过 `transform: skewY(-9deg)` 做成斜面：

```css
.roof {
  position: absolute;
  left: 28%;
  bottom: 45%;
  width: 58%;
  height: 28%;
  border: var(--line);
  background: var(--red);
  transform: skewY(-9deg);
}
```

### 4.4 厂房窗户

窗户不是 SVG，而是 `.plant` 内部的多个 `.window` 元素，用 CSS Grid 自动排布：

```css
.plant {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 10px;
  padding: 16px;
}

.window {
  border: 2px solid var(--ink);
  background: var(--blue);
  min-height: 34px;
}
```

### 4.5 烟雾动画

烟雾由三个圆形 `i` 标签组成，使用 `@keyframes puff` 做上下浮动：

```css
.smoke i {
  width: 26px;
  height: 26px;
  border: 2px solid var(--ink);
  border-radius: 50%;
  background: #fff;
  animation: puff 2.8s ease-in-out infinite;
}

@keyframes puff {
  0%, 100% { transform: translateY(0); opacity: .65; }
  50% { transform: translateY(-12px); opacity: 1; }
}
```

## 5. 图表绘制方法

### 5.1 环形图

环形图用 `conic-gradient` 绘制结构占比，再用伪元素盖住中间形成 donut：

```css
.donut {
  background: conic-gradient(var(--red) 0 40%, var(--blue) 40% 82%, var(--green) 82% 100%);
}

.donut::after {
  content: "";
  position: absolute;
  inset: 36px;
  border: var(--line);
  border-radius: 50%;
  background: var(--paper);
}
```

JavaScript 根据 Scope 1 和 Scope 2 的占比动态改写背景：

```js
document.getElementById("donut").style.background =
  `conic-gradient(var(--red) 0 ${s1}%, var(--blue) ${s1}% ${s1 + s2}%, var(--green) ${s1 + s2}% 100%)`;
```

### 5.2 柱状图

每条柱子是一个 `.bar-track` 加内部 `.bar-fill`。只改变 `width`，避免重排复杂布局：

```js
function setBar(id, ratio) {
  document.getElementById(id).style.width = `${Math.max(4, Math.min(ratio * 100, 100))}%`;
}
```

## 6. 交互逻辑

所有输入字段监听 `input` 和 `change` 事件，用户每次修改都会触发 `calculate()`：

```js
ids.forEach(id => document.getElementById(id).addEventListener("input", calculate));
ids.forEach(id => document.getElementById(id).addEventListener("change", calculate));
```

月度、季度、年度切换通过倍率实现：

```js
modeMultiplier = button.dataset.mode === "quarter" ? 3 : button.dataset.mode === "year" ? 12 : 1;
```

这样可以让同一组月度输入快速扩展到不同演示口径。

## 7. 碳核算逻辑

演示版采用最小核算模型：

```js
const factors = {
  electricity: 0.0005703,
  gas: 0.002162,
  diesel: 0.00263
};
```

计算步骤：

```js
const electricityEmissionGross = electricity * factors.electricity;
const greenOffset = electricityEmissionGross * greenRate;
const electricityEmission = Math.max(electricityEmissionGross - greenOffset, 0);
const gasEmission = gas * factors.gas;
const dieselEmission = diesel * factors.diesel;
const scope1 = gasEmission + dieselEmission;
const scope2 = electricityEmission;
const total = scope1 + scope2;
const intensity = total / revenue;
```

这里的设计原则是“简单、透明、可讲解”。正式项目中需要加入组织边界、运营边界、凭证、数据来源、排放因子版本、核查口径和审计记录。

## 8. 后续扩展建议

- 把排放因子抽到 JSON 文件，支持地区和年份切换。
- 增加导出 Excel / PDF 功能。
- 增加历史月份台账和趋势图。
- 增加产品碳足迹模式。
- 增加光伏、储能、绿证、碳配额模块。
- 接入后端数据库，保存企业、设备、能源台账和核算记录。
- 将单页 HTML 拆成 Vite / React / Vue 项目，便于长期维护。
