# 教程式垂直版面 + Tailwind CSS v4 用法笔记

> 适用场景:**像本项目一样,已经装了 Tailwind v4 的 Next.js 项目。**
> 本项目的样式是「两套混用」——**手写语义 class 管结构,Tailwind 工具类管细节**。
> 这份笔记教会你这套布局,以及 Tailwind 在这里到底怎么用。

---

## 〇、先认清:本项目是「两套样式混用」

| 类型 | 谁提供 | 写在哪 | 例子 |
|---|---|---|---|
| **语义 class** | 你自己手写 | `src/app/globals.css` | `.widget` `.section-header` `.widgets-grid` `.btn` |
| **工具类** | Tailwind v4 | 直接写在 JSX 的 `className` 里 | `flex` `gap-2` `text-center` `mb-4` `md:grid-cols-2` |

在 `Dashboard.tsx` 里两者并排出现:

```jsx
<div className="widget">                       {/* ← 手写语义 class:定义"这是卡片" */}
  <div className="flex gap-2 justify-center">   {/* ← Tailwind 工具类:局部排版 */}
    <button className="btn btn-primary">+</button>
  </div>
</div>
```

**分工原则(最重要的一句话):**

> **会重复、要统一的结构 → 写成语义 class;一次性的局部微调 → 用 Tailwind 工具类。**

---

## 一、Tailwind v4 是怎么接进这个项目的

v4 和老版本(v3)差别很大,**不再需要 `tailwind.config.js`**,全部在 CSS 里搞定:

```css
/* src/app/globals.css —— 只有这一行就启用了整个 Tailwind */
@import "tailwindcss";
```

加上 `postcss.config.mjs` 里挂了插件:

```js
const config = { plugins: ["@tailwindcss/postcss"] };
export default config;
```

就这两处。装好后,你在任何 `className` 里写 `flex`、`mb-4`、`text-center` 都会生效。

### `@theme inline`:把你的 CSS 变量"喂"给 Tailwind

v4 新增了 `@theme`,作用是**让 Tailwind 认识你自定义的设计变量**,从而生成对应的工具类:

```css
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --font-sans: var(--font-geist-sans);
  --font-mono: var(--font-geist-mono);
}
```

举例:因为这里声明了 `--color-background`,你就能直接写 `className="bg-background"`,
Tailwind 会自动生成 `background-color: var(--background)`。**自定义变量和工具类就这样打通了。**

---

## 二、整体布局结构(从外到内 4 层)

版面是竖向流式的,一节一节往下排,形成教程的章节感。结构如下:

```
dashboard-center        ← 页面:竖向流式 + 居中
  └─ tutorial-section   ← 一个章节,节间留白
       └─ section-inner ← 限制最大宽度(900px),避免大屏铺满
            ├─ section-header  ← 序号 + 标题 + 描述 三元素对齐
            └─ widgets-grid    ← 把 1~N 个卡片放进响应式网格
                 └─ widget      ← 卡片本体,内部再用 Tailwind 排版
```

对应 `Dashboard.tsx` 里的 `<Section>` 组件:

```jsx
function Section({ number, title, description, children }) {
  return (
    <div className="tutorial-section">
      <div className="section-inner">
        <div className="section-header">
          <div className="pattern-number">{number}</div>
          <div>
            <div className="section-title">{title}</div>
            <div className="section-description">{description}</div>
          </div>
        </div>
        <div className="widgets-grid">{children}</div>
      </div>
    </div>
  );
}
```

> 注意 `section-header` 里的结构:**序号是一个 div,标题+描述包在另一个 div 里**。
> 这样 flex 的两个"孩子"就是【圆圈】和【文字块】,`align-items: center` 一上,天然对齐。

---

## 三、关键语义 class 逐个讲(为什么这样写)

这些是你手写在 `globals.css` 里、值得抄到下个项目的结构 class。

### 1. CSS 变量 = 主题层(最值得抄走的部分)

颜色全部不写死,定义成变量,分别挂在 `:root`(亮色)和 `.dark`(暗色)上。
组件只引用 `var(--xxx)`,于是**切主题只需切换一个父级 class**:

```css
:root {
  --background: #ffffff;
  --card: #ffffff;
  --border: #e5e7eb;
  --primary: #6366f1;
  --muted-foreground: #64748b;
  /* …更多语义色 */
}
.dark {
  --background: #0f172a;
  --card: #1e293b;
  --border: #334155;
  --primary: #818cf8;
  /* …同名变量,换一套值 */
}
```

> 关键技巧:变量名按**用途**命名(`--card`、`--border`),不要按**颜色**命名(`--gray-100`)。
> 这样换配色时组件代码一行都不用改。

### 2. `dashboard-center` —— 竖向流式 + 居中

```css
.dashboard-center {
  display: flex;
  flex-direction: column;  /* 竖排 → "章节往下走"的阅读节奏 */
  align-items: center;     /* 水平居中 */
  width: 100%;
}
```

### 3. `section-inner` —— 用 `min()` 限制阅读宽度

```css
.section-inner {
  width: min(900px, 100%);  /* 取"900px"和"100%"里较小的那个 */
  margin: 0 auto;
}
```
- 屏 > 900px → 用 900px(留白聚焦);屏 < 900px → 用 100%(占满不溢出)。一行搞定两端。

### 4. `section-header` + `pattern-number` —— 三元素对齐

```css
.section-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;                            /* 用 gap 管间距,比 margin 省心 */
  border-bottom: 2px solid var(--border);  /* 分隔线,强化"章节"感 */
}
.pattern-number {
  display: inline-flex;
  align-items: center;
  justify-content: center;  /* 数字在圆圈里水平+垂直居中 */
  width: 2rem; height: 2rem;
  border-radius: 50%;       /* 正圆 = 宽高相等 + 50% 圆角 */
  background: var(--primary);
}
```

### 5. `widgets-grid` —— 自适应卡片网格

```css
.widgets-grid {
  display: grid;
  grid-template-columns: 1fr;  /* 默认单列 */
  gap: 1.5rem;
}
```
想要"放得下就多列、放不下自动减列",不写媒体查询的黄金写法:
```css
grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
```

### 6. `widget` —— 卡片本体

```css
.widget {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 0.75rem;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0,0,0,.1);
  transition: all 0.2s ease;
  box-sizing: border-box;        /* padding 算进宽度,避免溢出 */
}
.widget:hover {
  transform: translateY(-1px);   /* 轻微上浮 → "可交互"的反馈 */
}
```

---

## 四、卡片内部:Tailwind 工具类怎么用

结构搭好后,卡片**内部**的排版全交给 Tailwind。看本项目的真实代码:

```jsx
<div className="widget">
  <h3>
    <span className="widget-icon">🔢</span>
    Counter Widget
    <span className="pattern-badge">useState</span>
  </h3>

  <div className="text-center mb-4">            {/* 居中 + 下边距 */}
    <div className="text-3xl font-bold my-4">{count}</div>  {/* 大号粗体 + 上下边距 */}
  </div>

  <div className="flex gap-2 justify-center">   {/* 横向排列 + 间距 + 居中 */}
    <button className="btn btn-secondary">-</button>
    <button className="btn btn-primary">+</button>
  </div>
</div>
```

### 本项目最常用的 Tailwind 工具类速查

| 类别 | 工具类 | 作用 |
|---|---|---|
| **布局** | `flex` / `grid` | 开启 flex / grid |
| | `flex-col` | 主轴改为竖向 |
| | `grid-cols-1` `grid-cols-3` | grid 固定列数 |
| **对齐** | `items-center` | 交叉轴居中(垂直) |
| | `justify-center` `justify-between` | 主轴居中 / 两端对齐 |
| | `text-center` | 文字居中 |
| **间距** | `gap-2` `gap-3` `gap-6` | flex/grid 子元素间距 |
| | `mb-4` `mt-4` `my-4` | margin 下 / 上 / 上下 |
| | `p-3` `p-8` `px-3` `py-2` | padding(全/水平/垂直) |
| **字号字重** | `text-xs` `text-sm` `text-2xl` `text-3xl` | 字号 |
| | `font-bold` `font-semibold` `font-medium` | 字重 |
| **尺寸** | `w-full` `h-2` `flex-1` | 宽满 / 定高 / 占满剩余空间 |
| | `max-h-96 overflow-y-auto` | 限高 + 超出滚动 |
| **圆角** | `rounded` `rounded-xl` `rounded-2xl` `rounded-full` | 圆角等级 |
| **动效** | `transition-all duration-300` | 过渡动画 |
| | `hover:scale-[1.02]` | hover 放大(方括号=任意值) |

### 间距单位换算(记住这个就不用查)

Tailwind 数字 × `0.25rem` = 实际值。`gap-2` = `0.5rem`(8px),`mb-4` = `1rem`(16px),`p-3` = `0.75rem`(12px)。

### 响应式前缀:`md:` 是怎么回事

加 `md:` 前缀表示"屏幕 ≥ 768px 时才生效"。本项目 ContactForm 的左右分栏就靠它:

```jsx
<div className="grid grid-cols-1 md:grid-cols-2 gap-6">
  {/* 手机:1 列堆叠;平板及以上:2 列并排 */}
</div>
```
断点:`sm:`640 / `md:`768 / `lg:`1024 / `xl:`1280。**移动优先**——不带前缀的是手机默认值,带前缀的是"更大屏时覆盖"。

### 任意值:用方括号写非预设的数

预设里没有的值,用 `[ ]` 直接写:

```jsx
<div className="hover:scale-[1.02]">   {/* 放大到 1.02 倍 */}
<div className="w-[320px]">            {/* 宽 320px */}
```

### Tailwind 搞不定时,退回 inline style + 变量

涉及主题变量、渐变、动态值时,本项目直接用 `style`(注意是双花括号,外层 JSX 表达式、内层对象):

```jsx
<div
  className="text-sm mb-0"
  style={{ color: "var(--muted-foreground)" }}   {/* 颜色走变量,跟随主题切换 */}
/>

<div style={{ width: `${(done / total) * 100}%` }} />   {/* 进度条:动态宽度 */}
```

> 经验:**能用工具类就用工具类(简洁、统一),只有"需要变量 / 动态计算 / 渐变"时才退回 `style`。**

---

## 五、判断:什么时候写 class,什么时候用 Tailwind

| 写成语义 class(放进 globals.css) | 用 Tailwind 工具类(写在 className) |
|---|---|
| 项目里**反复出现**的东西 | 只是**这一处**的微调 |
| 卡片 `.widget`、按钮 `.btn`、徽章 `.pattern-badge`、输入框 `.input` | 这块要居中、这里加下边距、这排横着放 |
| 需要统一管理 hover / focus / 主题色的 | 临时的 flex 方向、间距、字号 |

**一个实用信号:** 如果你发现**同一串 Tailwind 类在多个地方复制粘贴**(比如按钮那一长串),
就该把它提炼成一个语义 class(`.btn`),或封装成 React 组件(本项目的 `<Button>` 就是这么做的)。

---

## 六、复用清单(下次开 Tailwind 项目照着做)

1. **接入 Tailwind v4**:`globals.css` 顶部 `@import "tailwindcss";` + `postcss.config.mjs` 挂插件。
2. **抄主题变量**:`:root` + `.dark`,用途命名;需要变工具类的变量放进 `@theme inline`。
3. **抄结构 class**:`dashboard-center` / `tutorial-section` / `section-inner` / `section-header` / `widgets-grid` / `widget`。
4. **卡片内部用 Tailwind**:`flex`、`gap-*`、`text-center`、`mb-*` 做局部排版。
5. **要分栏**用 `grid grid-cols-1 md:grid-cols-2`;**要自适应多列**把 `widgets-grid` 改成 `repeat(auto-fit, minmax(320px,1fr))`。
6. **切肤**:给最外层加 / 去 `.dark`,全站颜色自动切换。
7. **重复的一长串工具类**就提炼成语义 class 或 React 组件。

> 一句话总结整套思路:
> **变量管颜色,语义 class 管结构与复用,Tailwind 工具类管局部微调,`md:` 和 `auto-fit` 管响应式。**
