# BrainSphere 2D 词条关系图 — 开发文档

> 本文档供其他 AI 快速理解和生成 BrainSphere 项目数据。涵盖每一个字段的含义、两种节点类型的区别、渲染引擎参数、以及开发过程中踩过的坑。

---

## 一、项目概述

BrainSphere 是一个单文件 HTML 应用（`brainmap.html`），支持：
- **N-body 物理模拟**的词条关系可视化
- **两种页面类型**：脑图（mindmap，默认）和 Markdown 百科（article）
- **ZIP 导入/导出**：项目以 ZIP 包形式管理，内含多个 JSON 文件
- **图形化编辑**：编辑模式下可调整节点属性、创建连接、修改代码
- **Markdown 编辑器**：内置简易 Markdown 解析器和编辑界面
- **移动端适配**：虚拟摇杆 + 触控缩放平移

---

## 二、数据模型

### 2.1 全局状态 `STATE`

```javascript
STATE = {
  graph: {},        // 所有页面的键值对 { pageId: PageObject }
  focusId: '',      // 当前焦点页面ID
  viewport: { x: 0, y: 0, scale: 1 },  // 画布视口
  isEditing: false, // 是否处于编辑模式
  selectedNodeId: '', // 编辑模式下选中的节点ID
  projectPath: '',  // 项目文件路径（显示用）
  history: [],      // 浏览历史栈（用于返回）
  animProgress: 0   // 页面切换动画进度
}
```

### 2.2 页面（Page）数据结构

每个页面是一个 JSON 对象，有两种类型：

#### 类型 A：脑图页（默认，`type` 省略或不存在）

```json
{
  "id": "li",
  "word": "礼",
  "description": "礼者，礼节、礼仪也。",
  "children": [
    {
      "word": "冠礼",
      "targetPageId": "guanli",
      "weight": 1,
      "description": "冠礼：男子成人之礼",
      "connection": {
        "label": "成人",
        "style": "solid-arrow",
        "color": "#ff6b6b"
      },
      "sphere": {
        "style": "glass",
        "color": "#ff6b6b",
        "radius": 46
      }
    }
  ],
  "sphere": {
    "radius": 80,
    "style": "glass",
    "color": "#4a90d9"
  }
}
```

#### 类型 B：Markdown 百科页（`type: "article"`）

```json
{
  "id": "stu_001_intro",
  "word": "蔡博鑫 · 个人简介",
  "description": "蔡博鑫 的个人简介",
  "type": "article",
  "markdown": "## 基本信息\n\n| 字段 | 内容 |\n|------|------|\n| 姓名 | 蔡博鑫 |\n...",
  "children": [],
  "sphere": {
    "radius": 40,
    "style": "matte",
    "color": "#6ab0ff"
  }
}
```

### 2.3 字段详解

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 页面唯一标识符。在同一个项目中必须全局唯一。用于 `targetPageId` 跳转指向。 |
| `word` | string | ✅ | 节点的展示名称，显示在球体上。脑图页中也是中心节点的标题。 |
| `description` | string | ❌ | 悬浮提示内容（鼠标悬停节点时显示的 tooltip）。 |
| `type` | string | ❌ | 页面类型。省略或不存在时为 **mindmap**（脑图），设为 `"article"` 时为 Markdown 百科页。 |
| `markdown` | string | ❌ | **仅 article 类型有效**。页面的 Markdown 内容，会被解析为 HTML 显示。 |
| `children` | array | ❌ | 子节点列表。每个子节点定义一条连线和目标页面。**article 类型建议为空数组 `[]`**。 |
| `sphere` | object | ❌ | 中心节点（焦点节点）的球体样式。如果省略，使用 `config.defaults.mainSphere`。 |

#### `children` 数组中每个子节点的字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `word` | string | ✅ | 子节点球体上显示的文字 |
| `targetPageId` | string | ✅ | 点击该子节点后跳转到的目标页面 ID。如果为空字符串 `""`，表示不跳转（纯展示节点）。 |
| `weight` | number | ❌ | **权重，决定节点的排布位置**。按 weight 升序排列，weight 越小越靠近中心。范围建议 1~100。 |
| `description` | string | ❌ | 该子节点的悬浮提示内容 |
| `connection` | object | ❌ | 连线样式定义 |
| `sphere` | object | ❌ | 子节点球体样式定义 |

#### `connection` 对象字段：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `label` | string | `''` | 连线上显示的文字标签 |
| `style` | string | `'solid-arrow'` | 连线样式。可选值见下方「连线样式表」 |
| `color` | string | `'#888888'` | 连线颜色（十六进制） |

#### `sphere` 对象字段：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `radius` | number | 46 | 球体半径（像素） |
| `style` | string | `'glass'` | 球体渲染风格。可选值见下方「球体风格表」 |
| `color` | string | `'#6ab0ff'` | 球体主色（十六进制） |

### 2.4 两种节点类型的核心区别

| 特性 | mindmap（默认） | article |
|------|----------------|---------|
| `type` 字段 | 省略或不存在 | 必须设为 `"article"` |
| 显示方式 | Canvas 2D 渲染 N-body 脑图 | 隐藏 Canvas，显示 Markdown 文章视图 |
| `children` 作用 | 定义子节点和连线 | 建议为空（article 页通常不需要子节点导航） |
| `markdown` 字段 | 无效 | **核心内容**，会被 `parseMarkdown()` 解析为 HTML |
| 点击节点行为 | 导航到 targetPageId | article 页本身不显示在脑图上，而是由其他节点指向它 |
| 典型用途 | 知识层级、关系网络 | 个人简介、详细说明、百科词条 |

**重要**：article 页面的入口通常是在某个 mindmap 页面的 `children` 中，通过 `targetPageId` 指向 article 页面。当用户点击这个子节点时，应用检测到目标页面 `type === 'article'`，就切换到文章视图。

---

## 三、配置文件 `config`

每个项目必须包含一个 `config.json`（在 ZIP 中名为 `config.json`），键名为 `config`：

```json
{
  "_id": "config",
  "rootPageId": "INDEX",
  "transition": "zoom",
  "theme": "dark",
  "defaults": {
    "mainSphere": {
      "radius": 80,
      "style": "glass",
      "color": "#4a90d9",
      "fontSize": 22,
      "fontColor": "#ffffff",
      "fontFamily": "Microsoft YaHei"
    },
    "childSphere": {
      "radius": 46,
      "style": "metal",
      "color": "#6ab0ff",
      "fontSize": 13,
      "fontColor": "#ffffff",
      "fontFamily": "Microsoft YaHei"
    },
    "connection": {
      "style": "solid-arrow",
      "color": "#888888",
      "width": 2,
      "labelFontSize": 12,
      "labelFontColor": "#cccccc"
    }
  }
}
```

### config 字段详解

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | string | 固定为 `"config"`，与文件名一致 |
| `rootPageId` | string | 项目入口页面 ID。打开 ZIP 后首先加载这个页面 |
| `transition` | string | 页面切换动画类型。目前仅支持 `"zoom"` |
| `theme` | string | 主题。 `"dark"` 为深色模式 |
| `defaults.mainSphere` | object | 中心节点的默认样式（当页面未定义 `sphere` 时使用） |
| `defaults.childSphere` | object | 子节点的默认样式（当 child 未定义 `sphere` 时使用） |
| `defaults.connection` | object | 连线的默认样式（当 child 未定义 `connection` 时使用） |

### 样式继承规则

配置通过 `deepMerge(base, override)` 实现深层合并：
1. 页面级别的 `sphere` 覆盖 `config.defaults.mainSphere`
2. 子节点级别的 `sphere` 覆盖 `config.defaults.childSphere`
3. 子节点级别的 `connection` 覆盖 `config.defaults.connection`
4. 未定义的字段回退到配置默认值

### config.json 的导出与导入

**ZIP 导出时**：`saveProject()` 会自动生成 `config.json` 并打包进 ZIP。生成逻辑在 `buildConfigJSON()` 中：
- 去掉所有以 `_` 开头的内部字段（如 `_id`）
- 如果缺少 `rootPageId`，自动设为当前焦点页面 `STATE.focusId`，或回退到 `INDEX`，再回退到第一个页面

**ZIP/URL 导入时**：三种加载方式都会读取 `config.json` 中的 `rootPageId` 作为入口页面：
1. **本地上传 ZIP**：`rootId = graph.config?.rootPageId || Object.keys(graph).find(id => id !== 'config')`
2. **URL 加载 ZIP**：同上，通过 `loadProjectFromURL()` → `loadProject()` 链路
3. **文件夹打开**：通过 `loadProjectViaDirHandle()` 读取 `config.json`

**入口页面查找优先级**（由高到低）：
1. `config.json` 中的 `rootPageId`
2. `INDEX` 页面（如果存在）
3. 第一个非 `config` 的页面

---

## 四、球体风格表

| 风格值 | 视觉效果 |
|--------|---------|
| `glass` | 玻璃质感：半透明高光 + 内发光 + 柔和阴影 |
| `metal` | 金属质感：径向渐变模拟金属反光 + 强高光 |
| `matte` | 磨砂质感：纯色填充 + 柔和边缘 |
| `wireframe` | 线框质感：仅描边 + 内部半透明 |

---

## 五、连线样式表

| 样式值 | 视觉效果 |
|--------|---------|
| `solid` | 实线 |
| `dashed` | 虚线 |
| `solid-arrow` | 实线 + 末端箭头 |
| `dashed-arrow` | 虚线 + 末端箭头 |

---

## 六、ZIP 项目文件格式

### 6.1 文件结构

一个有效的 BrainSphere 项目 ZIP 包内部结构：

```
project.zip
├── config.json          # 全局配置（_id: "config"）
├── INDEX.json           # 入口页面（由 config.rootPageId 指定）
├── li.json              # 其他脑图页面
├── yue.json
├── stu_001.json         # 同学中心页（mindmap）
├── stu_001_intro.json   # 个人简介（article）
├── stu_001_rel.json     # 关系网（mindmap 或 article）
└── ...
```

### 6.2 每个 JSON 文件的命名规则

- 文件名（不含 `.json` 后缀）= 页面对象的 `id` 字段
- 必须严格一致，否则 ZIP 解析时无法建立映射
- `config.json` 的 `id` 固定为 `config`

### 6.3 加载流程

1. 解压 ZIP，读取所有 `.json` 文件
2. 用文件名（不含后缀）作为键，文件内容为值，构建 `STATE.graph` 对象
3. 从 `STATE.graph['config']` 读取 `rootPageId`
4. 调用 `focusOn(rootPageId)` 加载入口页面

---

## 七、渲染引擎核心参数

### 7.1 布局算法 `computeLayout(pageId)`

1. 中心节点固定在原点 `(0, 0)`
2. 子节点按 `weight` 升序排列
3. 子节点分层排布：
   - 第 1 层最多 6 个，半径 `mainRadius + childRadius + 80`
   - 第 2 层最多 12 个，半径 `上一层层半径 + childRadius * 2 + 60`
   - 每层递增，层内按角度均匀分布
4. 返回节点（连接 label 为 `"返回"` 的节点）固定放在底部

### 7.2 物理模拟参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `springK` | 0.05 | 弹簧刚度 |
| `repulsion` | 3000 | 节点间斥力 |
| `damping` | 0.9 | 速度阻尼 |
| `centerForce` | 0.02 | 向中心回拉的力 |
| `maxSpeed` | 15 | 最大速度限制 |

### 7.3 鼠标悬浮排斥

当鼠标悬停在节点上时，周围节点会被推开：
- 默认模式：`fan`（扇形展开）
- 力度随节点总数动态调整：`forceMultiplier = min(2.5, 0.8 + nodeCount * 0.15)`
- 基础力度：`baseForce = (hasHover ? 120 : 45) * forceMultiplier`
- 最大偏移：220 像素

### 7.4 字体自适应

```javascript
const fontSizeByRadius = node.radius * 0.4;
const adaptiveFontSize = Math.max(8, Math.min(baseFontSize, fontSizeByRadius));
```

- 字体大小不超过球半径的 40%
- 最小不小于 8px
- 文字总宽度限制在球直径的 90% 以内

---

## 八、交互系统

### 8.1 鼠标事件

| 操作 | 行为 |
|------|------|
| 单击节点（非编辑模式） | 导航到节点对应的 `targetPageId` |
| 单击节点（编辑模式） | 选中节点，右侧面板显示属性编辑器 |
| 拖拽空白处 | 平移画布 |
| 拖拽节点 | 移动节点位置（编辑模式） |
| 滚轮 | 缩放画布（0.15x ~ 4x） |
| 右键 | 弹出上下文菜单（新建节点等） |

### 8.2 编辑模式

点击顶部栏「✏️ 编辑」进入编辑模式：
- 右侧显示「节点属性」面板（可折叠）
- 可修改：名称、ID、悬浮提示、连线提示、权重、颜色、球体风格、半径、关系标签
- 可连接至已有节点（下拉框选择）
- 可删除节点
- 新增节点默认：`metal` 风格 + 随机颜色 + `solid-arrow` 连线

### 8.3 触控事件（移动端）

| 手势 | 行为 |
|------|------|
| 单指拖拽 | 平移画布 |
| 双指捏合 | 缩放画布 |
| 单指点击节点 | 导航/选中 |

### 8.4 虚拟摇杆

右下角浮动控件（默认隐藏，点击 `🎮` 按钮显示）：

```
   [▲]
[◄][▼][►]
 [+]  [−]
```

- 按住方向键：持续平移画布
- 按住 `+`/`-`：持续缩放
- 按钮样式半透明，不遮挡内容

---

## 九、Markdown 解析器

内置轻量 Markdown 解析器 `parseMarkdown(md)`，支持：

| Markdown 语法 | 输出 |
|---------------|------|
| `#` ~ `######` | `<h1>` ~ `<h6>` |
| `**粗体**` | `<strong>` |
| `*斜体*` | `<em>` |
| `` `行内代码` `` | `<code>` |
| ```代码块``` | `<pre><code>` |
| `> 引用` | `<blockquote>` |
| `![alt](url)` | `<img>` |
| `[text](url)` | `<a>` |
| `- 列表` / `* 列表` | `<ul><li>` |
| `1. 有序列表` | `<ol><li>` |
| `---` / `***` | `<hr>` |
| `\n\n` | `<p>` |
| `\|` 表格 `\|` | `<table>` |

---

## 十、开发踩坑记录（重要！）

### 坑 1：编辑器属性面板遮挡右侧按钮

**现象**：`right-panel` 里的日志按钮、历史按钮、虚拟摇杆切换按钮点击无反应。

**原因**：`#editor-panel`（z-index 100）在 DOM 中位于 `#right-panel` 之后，且尺寸重叠（都在右侧），导致 `editor-panel` 的空白背景区域拦截了所有点击事件。

**修复**：
```css
#editor-panel {
  pointer-events: none; /* 让空白区域穿透 */
}
#editor-panel input,
#editor-panel select,
#editor-panel textarea,
#editor-panel button {
  pointer-events: auto; /* 表单元素恢复可点击 */
}
```

**教训**：绝对定位且尺寸较大的面板，即使 `display: none`，在显示时也要注意 `pointer-events` 管理，避免遮挡其他交互元素。

---

### 坑 2：虚拟摇杆在 PC 端完全不工作

**现象**：PC 浏览器按 F12 进入手机模拟模式，虚拟摇杆按钮点击无反应。

**原因**：`initMobileControls()` 里有 `if (!isMobileDevice()) { panel.style.display = 'none'; return; }`，导致 PC 端直接 return，按钮事件完全没有绑定。

**修复**：移除移动端检测限制，改为：
```javascript
function initMobileControls() {
  const panel = document.getElementById('mobile-controls');
  if (!panel) return;
  panel.style.display = 'none'; // 默认隐藏，但事件始终绑定
  // ... 绑定所有按钮事件
}
```

**教训**：控件的事件绑定和显示逻辑要分离。不要因为在某种环境下"不需要显示"就跳过事件绑定。

---

### 坑 3：div 元素作为按钮时点击区域不确定

**现象**：虚拟摇杆按钮（`div.mc-btn`）在某些浏览器/环境下点击不灵敏。

**原因**：`div` 元素没有默认的按钮语义，部分浏览器对 `div` 的触控区域处理不一致，尤其是快速连续点击时。

**修复**：将所有可点击控件从 `div` 改为 `button`：
```html
<!-- 之前 -->
<div class="mc-btn" data-dir="up">▲</div>

<!-- 之后 -->
<button class="mc-btn" data-dir="up">▲</button>
```

同时重置 button 默认样式：
```css
#mobile-controls .mc-btn {
  padding: 0;
  margin: 0;
  font-family: inherit;
}
```

**教训**：任何需要用户点击交互的元素，优先使用语义化的 `<button>` 标签，而不是 `<div>`。

---

### 坑 4：JSON 控制字符导致解析失败

**现象**：从 textarea 读取 JSON 文本后，`JSON.parse()` 报错 "Bad control character"。

**原因**：textarea 中的真实换行符（`\n`）在 JSON 字符串中没有被转义，直接传入 `JSON.parse()` 时成为非法控制字符。

**修复**：添加 `safeJSONParse()` 函数：
```javascript
function safeJSONParse(text) {
  try {
    return JSON.parse(text);
  } catch (e) {
    // 逐字符转义控制字符
    let clean = '';
    for (const ch of text) {
      const code = ch.charCodeAt(0);
      if (code < 0x20 && code !== 0x09 && code !== 0x0a && code !== 0x0d) {
        clean += '\\u' + code.toString(16).padStart(4, '0');
      } else {
        clean += ch;
      }
    }
    return JSON.parse(clean);
  }
}
```

**教训**：永远不要直接对从 textarea 获取的文本调用 `JSON.parse()`。用户输入可能包含未转义的控制字符。

---

### 坑 5：折叠后面板工具栏没有靠右

**现象**：属性面板折叠后（32px 宽），toggle 按钮在标题栏中居中显示，视觉上不协调。

**修复**：折叠状态下标题栏的 `align-items` 从 `center` 改为 `flex-end`：
```css
#editor-panel.collapsed #editor-panel-header {
  flex-direction: column;
  gap: 6px;
  align-items: flex-end; /* 之前是 center */
}
```

---

### 坑 6：正则语法错误导致整个 JS 无法解析

**现象**：页面白屏，控制台显示 "Uncaught SyntaxError: Invalid regular expression"。

**原因**：Markdown 解析器中有一行正则 `\*\*\*+` 被错误地放在字符串模板中，实际解析为非法正则。

**修复**：将正则改为正确的字面量形式：
```javascript
// 之前（错误）
html = html.replace(/^(---|\*\*\*+|_{3,})/, '<hr>');

// 实际上代码中应为
html = html.replace(/^(---|\*{3,}|_{3,})/gim, '<hr>');
```

**教训**：在 HTML 文件内嵌的 `<script>` 中，转义字符容易出错。建议用 `node --check` 或 `new Function(js)` 做语法验证。

---

### 坑 7：saveProject 缺少 async 导致 await 报错

**现象**：保存项目时控制台报错 "await is only valid in async functions"。

**修复**：
```javascript
// 之前
function saveProject() {
  await someAsyncOperation();
}

// 之后
async function saveProject() {
  await someAsyncOperation();
}
```

---

### 坑 8：节点无法编辑（编辑模式下单击被识别为拖拽）

**现象**：进入编辑模式后，点击节点无法打开属性面板。

**原因**：`mousedown` 和 `mouseup` 之间计算了鼠标移动距离，但没有阈值判断，导致任何微小的鼠标移动都被判定为拖拽而非点击。

**修复**：在 `mousedown` 时记录起始位置，`mouseup` 时判断移动距离小于 8px 才视为点击：
```javascript
let dragStartPos = null;
canvas.addEventListener('mousedown', (e) => {
  dragStartPos = { x: e.clientX, y: e.clientY };
  // ...
});
canvas.addEventListener('mouseup', (e) => {
  const dx = e.clientX - dragStartPos.x;
  const dy = e.clientY - dragStartPos.y;
  if (Math.sqrt(dx*dx + dy*dy) < 8) {
    // 视为点击
  }
});
```

---

### 坑 9：运算符优先级导致 `rootPageId` 被忽略

**现象**：通过 URL 加载 ZIP 项目时，虽然 `config.json` 中正确设置了 `rootPageId`，但应用总是跳转到 `INDEX` 页面，或者找不到主页。

**原因**：JavaScript 中 `||` 的优先级低于 `? :`，以下代码：
```javascript
const rootId = configData?.rootPageId || STATE.graph['INDEX'] ? 'INDEX' : ...;
```
被解析为：
```javascript
const rootId = (configData?.rootPageId || STATE.graph['INDEX']) ? 'INDEX' : ...;
```
只要 `configData?.rootPageId` 存在（truthy），就会返回 `'INDEX'`，完全忽略了 `rootPageId` 的实际值。

**修复**：加括号明确优先级：
```javascript
const rootId = configData?.rootPageId || (STATE.graph['INDEX'] ? 'INDEX' : Object.keys(STATE.graph).find(id => id !== 'config'));
```

---

### 坑 10：ZIP 导出时缺少 `config.json`

**现象**：本地上传 ZIP 可以正常加载，但通过 URL 加载同款 ZIP 找不到主页。

**原因**：`saveProject()` 在导出 ZIP 时，`if (id === 'config') continue` 跳过了 `config.json`，导致 ZIP 包里没有配置文件。本地上传有额外的回退逻辑（取第一个页面），但 URL 加载依赖 `config.json` 中的 `rootPageId`。

**修复**：导出时生成干净的 `config.json`（去掉内部字段 `_id`）：
```javascript
function buildConfigJSON() {
  const cfg = STATE.graph['config'] || {};
  const clean = {};
  for (const [k, v] of Object.entries(cfg)) {
    if (k.startsWith('_')) continue;
    clean[k] = v;
  }
  if (!clean.rootPageId) {
    clean.rootPageId = STATE.focusId || STATE.graph['INDEX'] ? 'INDEX' : Object.keys(STATE.graph).find(id => id !== 'config');
  }
  return JSON.stringify(clean, null, 2);
}
// ZIP 导出时：zip.file('config.json', buildConfigJSON());
```

---

### 坑 11：`file://` 协议在浏览器中不可用

**现象**：通过路径加载项目时（非文件夹选择器），浏览器报 `file://` 相关错误。

**原因**：`loadProjectViaFetch()` 使用 `file://${path}/config.json` 拼接 URL，现代浏览器出于安全考虑禁止 `fetch()` 访问 `file://` 协议。

**修复**：去掉 `file://`，使用相对/绝对路径直接 `fetch`：
```javascript
const basePath = path.replace(/\\/g, '/').replace(/\/$/, '');
const configUrl = basePath ? `${basePath}/config.json` : 'config.json';
const resp = await fetch(configUrl);
```

---

## 十一、快速生成项目的 Python 模板

```python
import json, zipfile, os, shutil, random

def create_project(output_zip_path, pages_dict):
    """
    pages_dict: { page_id: page_object, ... }
    必须包含 config 页面
    """
    tmp_dir = output_zip_path + "_tmp"
    os.makedirs(tmp_dir, exist_ok=True)
    
    for pid, pdata in pages_dict.items():
        with open(os.path.join(tmp_dir, f"{pid}.json"), "w", encoding="utf-8") as f:
            json.dump(pdata, f, ensure_ascii=False, indent=2)
    
    with zipfile.ZipFile(output_zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
        for f in os.listdir(tmp_dir):
            zf.write(os.path.join(tmp_dir, f), f)
    
    shutil.rmtree(tmp_dir)
    print(f"项目已生成: {output_zip_path}")

# 示例：创建一个简单项目
project = {
    "config": {
        "_id": "config",
        "rootPageId": "home",
        "transition": "zoom",
        "theme": "dark",
        "defaults": {
            "mainSphere": {"radius": 80, "style": "glass", "color": "#4a90d9", "fontSize": 22, "fontColor": "#ffffff"},
            "childSphere": {"radius": 46, "style": "metal", "color": "#6ab0ff", "fontSize": 13, "fontColor": "#ffffff"},
            "connection": {"style": "solid-arrow", "color": "#888888", "width": 2}
        }
    },
    "home": {
        "id": "home",
        "word": "首页",
        "description": "项目首页",
        "children": [
            {
                "word": "子节点A",
                "targetPageId": "page_a",
                "weight": 1,
                "description": "",
                "connection": {"label": "", "style": "solid-arrow", "color": "#ff6b6b"},
                "sphere": {"style": "glass", "color": "#ff6b6b", "radius": 46}
            }
        ],
        "sphere": {"radius": 80, "style": "glass", "color": "#4a90d9"}
    },
    "page_a": {
        "id": "page_a",
        "word": "子节点A",
        "description": "子节点A的详细说明",
        "type": "article",  # ← 这是一个 Markdown 百科页
        "markdown": "## 标题\n\n这里是详细内容...",
        "children": [],
        "sphere": {"radius": 40, "style": "matte", "color": "#6ab0ff"}
    }
}

create_project("my_project.zip", project)
```

---

## 十二、开发检查清单

在生成项目数据后，用以下清单验证：

- [ ] `config.json` 存在且 `_id` 为 `"config"`
- [ ] `config.rootPageId` 指向一个实际存在的页面 ID
- [ ] 所有 `targetPageId` 都指向实际存在的页面（或空字符串表示不跳转）
- [ ] 页面 ID 全局唯一，且与 JSON 文件名一致
- [ ] article 类型的页面 `type` 字段明确设为 `"article"`
- [ ] article 页面的 `markdown` 字段不为空（至少有一些内容）
- [ ] mindmap 页面的 `children` 如果是返回节点，label 设为 `"返回"`
- [ ] 所有颜色值使用完整的 6 位十六进制（如 `#ff6b6b`，不要用 `#fff`）
- [ ] JSON 中不包含非法控制字符（如未转义的换行符）
- [ ] 权重 `weight` 使用数字类型，不要用字符串

---

*文档版本：2026-05-31*  
*对应 BrainSphere 版本：多轮迭代后的稳定版*