# 06_前端HTML_CSS_JavaScript语法与作用精讲

这份文档解释前端三件套：

```text
frontend/index.html
frontend/styles.css
frontend/app.js
```

前端负责用户能看到、能点击、能输入的部分。

## 一、HTML 精讲

HTML 的作用是搭建网页结构。

你可以把 HTML 理解成房子的骨架。

CSS 是装修。

JavaScript 是电路和开关。

### 1. `<!DOCTYPE html>`

```html
<!DOCTYPE html>
```

这是文档类型声明。

它告诉浏览器：这个网页使用 HTML5 标准。

如果不写，浏览器可能用旧模式解析页面，页面显示可能不一致。

### 2. `<html lang="zh-CN">`

```html
<html lang="zh-CN">
```

`html` 是整个网页的根标签。

`lang="zh-CN"` 表示页面主要语言是简体中文。

这有助于浏览器、翻译工具、搜索引擎理解页面语言。

### 3. `<head>` 区域

`head` 里面放页面配置，不是主要显示内容。

常见内容包括：

```html
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>学习养成计划 V2</title>
<link rel="stylesheet" href="styles.css" />
```

#### charset

```html
<meta charset="UTF-8" />
```

设置网页编码。

如果没有正确编码，中文可能乱码。

#### viewport

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

让网页在手机、平板、电脑上都能按屏幕宽度显示。

#### title

```html
<title>学习养成计划 V2</title>
```

浏览器标签页显示的标题。

#### link

```html
<link rel="stylesheet" href="styles.css" />
```

引入 CSS 文件。

这表示页面样式写在 `styles.css` 中。

### 4. `<body>` 区域

`body` 是网页真正显示给用户看的内容。

本项目的 `body` 大致结构是：

```html
<body>
  <div class="app">
    <aside class="sidebar">左侧栏</aside>
    <main class="main">主要内容</main>
  </div>
  <div id="toast" class="toast"></div>
  <script src="app.js"></script>
</body>
```

意思是：

- `app` 是整个页面容器。
- `sidebar` 是左侧菜单和提醒。
- `main` 是右侧主体内容。
- `toast` 是右下角提示框。
- `script` 引入 JS 文件。

### 5. class 和 id 的区别

HTML 中经常有：

```html
<div class="metric-card" id="totalCount"></div>
```

#### class

`class` 通常给 CSS 用。

多个元素可以有同一个 class。

例如很多卡片都可以叫 `metric-card`。

#### id

`id` 通常给 JavaScript 用。

一个页面里同一个 id 最好只出现一次。

例如：

```js
document.querySelector("#totalCount")
```

就是通过 id 找到总任务数的位置。

### 6. 表单 form

添加任务区域使用了：

```html
<form id="taskForm" class="panel form-panel">
```

`form` 表示表单。

用户输入任务名称、主题、日期、时间后，点击提交按钮。

JavaScript 会监听表单提交事件，把数据发给 C++ 后端。

### 7. input 输入框

例如：

```html
<input id="titleInput" required placeholder="例如：复习 C++ 结构体" />
```

含义：

- `id="titleInput"`：JS 可以找到它。
- `required`：浏览器要求不能为空。
- `placeholder`：输入框里的提示文字。

### 8. select 下拉框

```html
<select id="priorityInput">
  <option value="3">高</option>
  <option value="2" selected>中</option>
  <option value="1">低</option>
</select>
```

用于选择优先级。

`value="3"` 传给后端的是数字 3。

用户看到的是“高”。

### 9. button 按钮

```html
<button class="primary-btn" type="submit">添加任务</button>
```

按钮类型是 `submit`，表示点击后提交表单。

JavaScript 会拦截这个提交动作，转成请求后端。

### 10. script 引入 JS

```html
<script src="app.js"></script>
```

表示加载 `app.js`。

如果没有这句，页面只是静态页面，按钮不会真正工作。

## 二、CSS 精讲

CSS 负责页面样式。

你可以把 CSS 理解成“给 HTML 穿衣服”。

### 1. CSS 变量

```css
:root {
  --bg: #eef2f8;
  --surface: #ffffff;
  --blue: #2864e8;
}
```

`--bg`、`--surface`、`--blue` 是自定义变量。

使用时写：

```css
background: var(--bg);
```

好处是：

如果想统一修改颜色，只改变量即可。

### 2. 通配选择器

```css
* { box-sizing: border-box; }
```

`*` 表示所有元素。

`box-sizing: border-box` 让宽度计算更直观。

设置后，元素宽度包含 padding 和 border，不容易把布局撑坏。

### 3. body 样式

```css
body {
  margin: 0;
  min-height: 100vh;
  color: var(--ink);
  background: var(--bg);
  font-family: "Microsoft YaHei", Arial, sans-serif;
}
```

解释：

- `margin: 0`：去掉浏览器默认外边距。
- `min-height: 100vh`：页面至少占满一屏高度。
- `color`：默认文字颜色。
- `background`：页面背景色。
- `font-family`：字体。

### 4. Grid 网格布局

```css
.app {
  min-height: 100vh;
  display: grid;
  grid-template-columns: 304px 1fr;
}
```

这表示页面分成两列：

- 左侧 304px。
- 右侧占剩余空间。

这就是左侧栏 + 主内容区的布局。

### 5. Flex 弹性布局

```css
.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}
```

`flex` 适合让元素横向排列。

这里让 logo 和标题横向排列。

`gap` 表示两个元素之间的间距。

### 6. 卡片样式

```css
.side-card, .panel, .metric-card {
  background: var(--surface);
  border: 1px solid var(--line);
  border-radius: 8px;
  box-shadow: var(--shadow);
}
```

这里把侧边卡片、普通面板、统计卡片设置成统一风格。

包括：

- 白色背景。
- 边框。
- 圆角。
- 阴影。

### 7. 响应式布局

```css
@media (max-width: 1080px) {
  .app { grid-template-columns: 1fr; }
}
```

意思是：当屏幕宽度小于 1080px 时，不再左右两列，而是改成一列。

这样手机或小屏幕上不会挤在一起。

## 三、JavaScript 精讲

JavaScript 负责页面交互。

### 1. API 对象

```js
const API = {
  tasks: "/api/tasks",
  stats: "/api/stats",
  reminders: "/api/reminders",
  recommendations: "/api/recommendations",
  resetDemo: "/api/demo/reset"
};
```

这是一个对象。

对象里保存后端接口地址。

这样以后请求任务时写：

```js
API.tasks
```

比每次手写 `"/api/tasks"` 更清楚。

### 2. let 变量

```js
let tasks = [];
let stats = {...};
let recommendations = [];
let currentFilter = "all";
```

`let` 表示变量以后可以改变。

这些变量保存前端当前数据。

- `tasks`：任务列表。
- `stats`：统计数据。
- `recommendations`：推荐任务。
- `currentFilter`：当前筛选条件。

### 3. `$` 简写函数

```js
const $ = selector => document.querySelector(selector);
```

这是箭头函数。

它的作用是简化查找页面元素。

原来要写：

```js
document.querySelector("#taskList")
```

现在可以写：

```js
$("#taskList")
```

### 4. el 对象

```js
const el = {
  taskList: $("#taskList"),
  totalCount: $("#totalCount")
};
```

`el` 保存页面元素。

好处是：

不用每次都重新查找元素。

后面可以直接写：

```js
el.taskList.innerHTML = "...";
```

### 5. async 和 await

```js
async function getJson(url) {
  const response = await fetch(url);
  return response.json();
}
```

`async` 表示这是异步函数。

`await` 表示等待结果。

因为请求后端需要时间，所以不能像普通代码一样马上得到结果。

### 6. fetch 请求后端

```js
fetch("/api/tasks")
```

表示向 C++ 后端请求任务列表。

后端收到请求后返回 JSON。

前端再把 JSON 显示出来。

### 7. POST 提交表单

```js
fetch(url, {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: params.toString()
});
```

意思是：

- 请求方式是 POST。
- 提交的是表单格式。
- 数据放在 body 里。

添加任务、删除任务、完成打卡都用了 POST。

### 8. render 渲染函数

项目中很多函数以 `render` 开头。

例如：

- `renderTasks()`：渲染任务列表。
- `renderMetrics()`：渲染统计卡片。
- `renderCalendar()`：渲染打卡日历。
- `renderAnalysis()`：渲染数据分析。

“渲染”的意思是：

> 把数据变成网页上的内容。

### 9. innerHTML

```js
el.taskList.innerHTML = visible.map(...).join("");
```

`innerHTML` 可以修改某个元素内部的 HTML。

这里把任务数组转换成很多任务卡片，再放到页面上。

### 10. map 函数

```js
tasks.map(task => `...`)
```

`map` 会遍历数组中的每个任务。

每个任务都会转换成一段 HTML 字符串。

最后得到一个新的数组。

### 11. join 函数

```js
.join("")
```

把数组中的多段字符串拼成一个完整字符串。

如果不 join，页面不能直接显示数组。

### 12. filter 函数

```js
tasks.filter(task => !task.completed)
```

筛选未完成任务。

如果任务满足条件，就保留。

否则过滤掉。

### 13. 事件绑定

```js
el.taskForm.addEventListener("submit", addTaskFromForm);
```

意思是：当表单提交时，执行 `addTaskFromForm` 函数。

项目中的按钮点击、导航切换、月份选择都用了事件绑定。

### 14. classList

```js
button.classList.add("active");
```

`classList` 用来增加或删除 CSS 类名。

比如导航按钮被点击后，给它加上 `active`，它就会变成高亮样式。

### 15. 前端整体运行流程

1. 页面加载。
2. JS 执行 `loadAll()`。
3. 前端请求任务、统计、提醒、推荐任务。
4. 后端返回 JSON。
5. 前端调用 render 函数更新页面。
6. 用户点击按钮。
7. JS 调用后端接口。
8. 后端修改数据。
9. 前端重新刷新页面。