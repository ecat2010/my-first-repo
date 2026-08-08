# 第 6 章 浏览器集成与核心 API

> 本章导读：上一章我们看了 Mermaid 内部的「厨房」是怎么运作的。这一章我们站在使用者的角度，学怎么在网页里把 Mermaid 接进来，并认识它对外提供的几个核心接口。学完之后，你会知道三种集成方式，并能用 `initialize`、`run`、`render`、`parse`、`detectType` 这几个 API 随心所欲地画图、校验图。

## 6.1 三种集成方式概览

把 Mermaid 用在你自己的网页里，常见有三种做法，由「省事程度」从低到高排列：

1. **零 JS 自动渲染**：只要用 `<pre class="mermaid">` 包住图定义文本，再引入 Mermaid 的脚本，页面加载时它就会自动把图画出。这是最简单的方式，默认 `startOnLoad` 为 `true`，即「加载即画」。
2. **API 方式（手动控制）**：先 `initialize({ startOnLoad: false })` 关掉自动渲染，之后再手动调用 `render` 把某段文本画出来。适合你想在按钮点击、数据加载完成后再画图的场景。
3. **自定义选择器 / 节点**：用 `mermaid.run({ querySelector: '.x' })` 或传入 `nodes: [...]` 来指定「只渲染哪些元素」。当你页面里混着 Mermaid 图和其他内容时特别有用。

无论哪种方式，核心都是后面这几个 API。先记住一个坑：自动渲染认的是 **`class="mermaid"`** 这个 CSS 类，不是 `<mermaid>` 这种标签。

## 6.2 核心 API 逐个讲

下面这些接口都挂在主库导出的 `mermaid` 对象上（入口见 `packages/mermaid/src/mermaid.ts`）。注意：下面代码块里的 `mermaid.run()`、`mermaid.initialize()` 都是 **JavaScript API**，属于正常代码，不是终端命令。

### 6.2.1 `mermaid.initialize(config)` —— 设置全局配置

在渲染之前调用，用来设置全局配置，比如主题、安全级别、是否自动加载等。

```javascript
mermaid.initialize({
  startOnLoad: true,
  theme: 'dark',
});
```

它只「登记配置」，不产生任何图。通常整个页面调用一次即可。

### 6.2.2 `mermaid.run(options?)` —— 扫描并渲染（v10+ 推荐）

v10 之后官方推荐的入口。它扫描页面里 `class="mermaid"` 的元素并逐一渲染。`options` 可选，常用字段：

- `querySelector`：要扫描的选择器，默认 `'.mermaid'`；
- `nodes`：直接传一组 DOM 节点数组；
- `suppressErrors`：出错时是否静默。

```javascript
// 只渲染 class 为 "my-diagram" 的元素
mermaid.run({ querySelector: '.my-diagram' });
```

### 6.2.3 `mermaid.render(id, text, container?)` —— 渲染成 SVG 字符串

把一段图定义文本渲染成图片，返回 `{ svg, bindFunctions }`。`id` 是给这张图起的唯一标识，`text` 是图定义文本，`container` 可选（某些交互图需要挂在某个 DOM 容器上）。

```javascript
const { svg, bindFunctions } = await mermaid.render('myId', 'graph LR; A-->B');
document.getElementById('holder').innerHTML = svg;
bindFunctions?.(document.getElementById('holder'));
```

`svg` 是画好的 SVG 代码，`bindFunctions` 是可选的「绑定交互函数」，有可点击节点时才需要调用。

### 6.2.4 `mermaid.parse(text)` —— 只校验不画图

只检查语法对不对，**不真的画**。返回 `{ diagramType }` 告诉你这是哪种图。写编辑器、做语法检查很有用。

```javascript
const result = await mermaid.parse('graph LR; A-->B');
console.log(result.diagramType); // 例如 "flowchart"
```

如果语法有错，它会抛出异常，你可以用 `try/catch` 接住并提示用户。

### 6.2.5 `mermaid.detectType(text)` —— 探测图类型

传入文本，返回它属于哪种图。和 `parse` 类似但更轻量，只做「认类型」这件事。

```javascript
const type = mermaid.detectType('sequenceDiagram\nA->>B: hi');
```

### 6.2.6 关于 `mermaid.init()`

旧版有个 `mermaid.init()`，现在**已废弃**。请一律用 `initialize()` + `run()` 代替，本章的例子也都是新写法。

## 6.3 安全级别 securityLevel

`securityLevel` 控制 Mermaid 能不能执行 HTML 或点击交互。它默认是 **`strict`**：足够安全，但会禁止 HTML 标签和点击交互。

如果你要做「可点击节点」的演示（比如点击节点跳链接），需要把它放宽：

```javascript
mermaid.initialize({
  securityLevel: 'loose', // 放开交互，仅用于可信内容
});
```

提醒一句：`loose` 会允许图里嵌入 HTML，只应在你信任图内容来源时使用，不要对来历不明的用户输入开这个口子。

## 6.4 完整示例

下面给两个可直接用的完整 HTML 例子。注意这里用的是浏览器 standalone 版（IIFE 全局构建 `mermaid.min.js`），通过 CDN 引入，不需要你本地安装任何东西。

### 示例一：API 方式手动 render 一张图

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>Mermaid API 示例</title>
  <script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
</head>
<body>
  <div id="holder"></div>

  <script>
    mermaid.initialize({ startOnLoad: false, theme: 'forest' });

    (async () => {
      const text = 'graph LR\n  A[开始] --> B{判断}\n  B -->|是| C[执行]\n  B -->|否| D[结束]';
      const { svg, bindFunctions } = await mermaid.render('demo1', text);
      const holder = document.getElementById('holder');
      holder.innerHTML = svg;
      bindFunctions?.(holder);
    })();
  </script>
</body>
</html>
```

**预期结果**：页面打开后，在 `holder` 区域出现一张绿森森（forest 主题）的流程图，包含「开始 → 判断 → 执行/结束」的结构。

### 示例二：自定义选择器用 run 渲染

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>Mermaid run 选择器示例</title>
  <script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
</head>
<body>
  <pre class="my-diagram">sequenceDiagram
    用户->>系统: 请求数据
    系统-->>用户: 返回结果</pre>

  <script>
    mermaid.initialize({ startOnLoad: false });
    mermaid.run({ querySelector: '.my-diagram' });
  </script>
</body>
</html>
```

**预期结果**：页面加载后，`mermaid.run` 扫描到 `class="my-diagram"` 的元素，把它渲染成一张时序图，显示「用户 → 系统：请求数据」与「系统 → 用户：返回结果」两条消息。

> 补充：如果你想要 ESM 写法（适合现代打包器），CDN 地址用 `https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs`，用 `import mermaid from '...'` 引入即可，接口完全一致。

## 本章小结

本章我们学会了在浏览器里「驾驭」Mermaid：

- 三种集成方式：**零 JS 自动渲染**、**API 手动控制**、**自定义选择器/节点**；
- 核心 API：`initialize`（设配置）、`run`（扫描渲染，v10+ 推荐）、`render`（出 SVG）、`parse`（只校验）、`detectType`（探类型）；旧 `init()` 已废弃；
- `securityLevel` 默认 `strict`，要演示可点击节点改 `loose`；
- 用 `class="mermaid"`（不是 `<mermaid>` 标签）触发自动渲染；
- 通过 CDN 引入 standalone 版即可零安装使用。

现在你既懂了 Mermaid 内部怎么运作（第 5 章），又会了在网页里调用它（本章）。万事俱备，下一步就是真正去写各种图的语法了。下一章开始，我们将进入《07-流程图flowchart语法详解.md》，从流程图、时序图一路学到思维导图、甘特图等 30 多种图的具体写法。
