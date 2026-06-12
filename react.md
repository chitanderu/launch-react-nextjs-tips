# React 实战笔记:ContactForm + useCallback / useMemo

> 以本项目 `Dashboard.tsx` 里的 `ContactForm`(PATTERN 6)为案例,
> 讲透 `useCallback`、`useMemo`、依赖数组,以及一个"受控表单 + 多条数据持久化"功能的完整实现。

---

## 〇、依赖数组:那个末尾的方括号是什么

`useCallback`、`useMemo`、`useEffect` 的**第二个参数**都是「依赖数组」:

```jsx
const submissionStats = useMemo(
  () => { return {...}; },   // 第 1 个参数:要缓存的"计算函数"
  [submittedDataList]        // 第 2 个参数:依赖数组 ← 末尾的方括号
);
```

作用:**告诉 React「什么时候才需要重新执行」**。React 盯着方括号里的值,**只有它变了才重算**,否则返回上次缓存结果。

| 写法 | 含义 |
|---|---|
| `[submittedDataList]` | 该值变了才重算 ✅ |
| `[]` 空数组 | 只在首次渲染执行一次,之后永不重算 |
| 不写第二个参数 | 每次渲染都执行 = 等于没优化 |

**铁律:函数体里读到的、会变化的外部变量,都要列进依赖数组。**
这里函数体读了 `submittedDataList`,所以它必须进数组。

---

## 一、useCallback 解决什么问题

### 背景:每次渲染,函数都"重新出生"

React 组件每次重渲染(state 变、父组件刷新),**整个函数体从头跑一遍**,里面定义的函数每次都是**全新对象**(内存地址不同)。

```jsx
function ContactForm() {
  const handleSubmit = (e) => { ... };  // 每次渲染都是"新"函数
}
```

普通按钮无所谓,但两种情况会出问题:

1. **传给 `React.memo` 优化过的子组件** → 子组件误以为 props 变了,白白重渲染
2. **放进 `useEffect` / `useMemo` 的依赖数组** → 依赖每次都"变",反复触发

### useCallback 的作用:把函数"记住"

```jsx
const handleSubmit = useCallback(
  (e) => { ... },     // 你的函数
  [formData, nextId], // 依赖数组
);
```

**依赖不变 → 返回上一次那个一模一样的函数对象**;依赖变了才造新的。
🐍 类比 Python 的 `functools.lru_cache`:输入没变就返回缓存。

---

## 二、本项目 4 个 useCallback 逐个讲

### 1. handleChange —— 受控输入的统一处理器

```jsx
const handleChange = useCallback(
  (e) => {
    const { name, value } = e.target;
    setFormData((prev) => ({ ...prev, [name]: value }));  // 计算属性名
    if (errors[name]) {
      setErrors((prev) => ({ ...prev, [name]: "" }));     // 一打字就清该字段错误
    }
  },
  [errors],  // 函数体读了 errors[name],必须列入
);
```

两个关键技巧:

- **`[name]: value` 计算属性名**:一个 handler 搞定 name/email/message 三个框。靠每个 `<input>` 的 `name="..."`,从 `e.target.name` 取出是哪个字段。**不用写三个 handler。**
- **函数式更新 `(prev) => ...`**:从回调拿"最新的 prev",避免闭包拿到过期旧值。

### 2. handleSubmit —— 校验 + 提交

```jsx
const handleSubmit = useCallback(
  (e) => {
    e.preventDefault();          // 阻止表单默认刷新页面(关键)

    const newErrors = {};        // 收集校验错误
    if (!formData.name.trim()) newErrors.name = "Name is required";
    if (!formData.email.includes("@")) newErrors.email = "Valid email required";
    if (!formData.message.trim()) newErrors.message = "Message is required";
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;                    // 有错就停,不提交
    }

    setIsSubmitting(true);       // 进入"提交中" → 按钮变 Sending、输入框禁用
    setTimeout(() => {           // 模拟 1.5s API 请求
      const newSubmission = { id: nextId, ...formData, submittedAt: new Date().toLocaleString() };
      setSubmittedDataList((prev) => [newSubmission, ...prev]);  // 新的放最前
      setNextId((prev) => prev + 1);
      setFormData({ name: "", email: "", message: "" });         // 清空表单
      setErrors({});
      setIsSubmitting(false);
    }, 1500);
  },
  [formData, nextId],  // 校验读了 formData,生成 id 读了 nextId
);
```

### 3 & 4. 删除函数 —— 空依赖 `[]` 的典范

```jsx
const handleDeleteSubmission = useCallback((id) => {
  setSubmittedDataList((prev) => prev.filter((s) => s.id !== id));
}, []);  // ← 空数组

const handleDeleteAll = useCallback(() => {
  setSubmittedDataList([]);
}, []);  // ← 空数组
```

**为什么能用 `[]`(永不重建)?** 因为没读任何外部 state,只调用 `setSubmittedDataList`,且用**函数式更新 `(prev) => ...`** 从参数拿最新值。
→ 这是函数式更新最大的好处:**让 callback 摆脱依赖,函数永远稳定。**

---

## 三、依赖数组速记规则

| 函数体里用到了什么 | 依赖数组该写什么 |
|---|---|
| 读了某个 state/prop(`errors[name]`、`formData`) | 必须列进去 |
| 只调用 `setXxx` 且用 `(prev) => ...` | 可以留空 `[]` |
| 调用了其它 useCallback 函数 | 把那个函数也列进去 |

**一句话:函数体里读到的外部变量都要进依赖;想摆脱依赖就改用函数式更新。**

---

## 四、完整功能实现教学(从零搭)

### 第 1 步:设计 state(5 个)

```jsx
const [formData, setFormData] = useState({ name: "", email: "", message: "" });
const [errors, setErrors] = useState({});                // 字段名 → 错误信息
const [submittedDataList, setSubmittedDataList] = useState([]);  // 已提交列表
const [isSubmitting, setIsSubmitting] = useState(false); // 提交中?
const [nextId, setNextId] = useState(1);                 // 自增 id 计数器
```

### 第 2 步:受控输入(Controlled Component)

```jsx
<input
  name="name"               // handleChange 靠它区分字段
  value={formData.name}     // 值来自 state(唯一数据源)
  onChange={handleChange}   // 改动写回 state
/>
```

闭环:敲键盘 → onChange → 更新 state → 重渲染 → 输入框显示新值。

### 第 3 步:提交时校验

收集错误进 `newErrors`,有错 `setErrors` 并 `return` 中断;没错才提交。配合 handleChange 里"一打字清错误",体验更顺。

### 第 4 步:用 isSubmitting 做加载态

提交瞬间 `setIsSubmitting(true)`:按钮变 "Sending..."、`disabled`、输入框禁用、顶部显示提示。完成后设回 `false`。**防重复提交 + 给反馈。**

### 第 5 步:存多条 + 自增 id

```jsx
setSubmittedDataList((prev) => [newSubmission, ...prev]);  // 展开运算符:新的在前
setNextId((prev) => prev + 1);
```

用 `nextId` 而非 `list.length` 当 id——删除后 length 会变,可能撞 id。**自增计数器保证唯一**(这正是列表渲染 `key` 要稳定唯一的原因)。

### 第 6 步:用 useMemo 算统计

```jsx
const submissionStats = useMemo(() => {
  return {
    total: submittedDataList.length,
    uniqueEmails: new Set(submittedDataList.map((s) => s.email)).size,  // Set 去重
    avgMessageLength:
      submittedDataList.length > 0
        ? Math.round(
            submittedDataList.reduce((sum, s) => sum + s.message.length, 0) /
              submittedDataList.length,
          )
        : 0,  // 防止除以 0
  };
}, [submittedDataList]);  // 只在列表变化时才重算
```

- `useCallback` 缓存**函数**,`useMemo` 缓存**计算结果**。
- `new Set(...).size` 是去重计数的惯用法。
- `reduce` 求和再除以个数 = 平均值;空列表时返回 0,避免 `NaN`。

### 第 7 步:渲染——三态切换 + group hover

```jsx
{submittedDataList.length > 0 ? (
  /* 有数据:统计卡片 + 列表,map 渲染,key={submission.id} */
) : (
  /* 空状态:友好的 "No Messages Yet" */
)}
```

hover 才出现的删除按钮(Tailwind `group` 技巧):

```jsx
<div className="group ...">
  <Button className="opacity-0 group-hover:opacity-100 ...">✕</Button>
</div>
```
`group` 标记父级,`group-hover:` = "父级被 hover 时"。按钮平时隐藏,鼠标移上去才浮现。

---

## 五、整体数据流

```
用户打字
  → onChange → handleChange → setFormData         (受控输入闭环)
点击提交
  → handleSubmit → 校验失败 → setErrors,停
                 → 校验通过 → setIsSubmitting(true)
                            → (1.5s 后) 加进列表、nextId+1、清空表单、isSubmitting(false)
  → submittedDataList 变化 → useMemo 重算 submissionStats
                          → 列表区重渲染(统计卡片 + 历史)
点击删除
  → handleDeleteSubmission / handleDeleteAll → 过滤/清空 → 重渲染
```

---

## 六、Hook 分工总结

- **useState** — 存会变的数据
- **useCallback** — 缓存**函数**,依赖不变时保持同一个,避免子组件无谓重渲染 / 闭包旧值
- **useMemo** — 缓存**计算结果**,避免每次渲染重算昂贵逻辑
- **函数式更新 `(prev) => ...`** — 让 callback 摆脱 state 依赖,从而能用 `[]` 空依赖,函数永远稳定

> ⚠️ 诚实提醒:本项目把 `useCallback` 用在了普通 DOM 元素(`<input>`、`<form>`)上,
> **性能收益几乎为零**——原生 DOM 不会因函数变化而重渲染,这里主要是**教学演示写法**。
> 真正必要的场景:把函数传给 `React.memo` 包裹的子组件,或放进其它 Hook 的依赖数组时。

---

# PATTERN 7:Advanced Patterns —— 自定义 Hook & 性能优化

> 案例:`NotesWidget`(笔记小组件)+ 自定义 Hook `useLocalStorage`。
> 核心:**把"带状态的逻辑"抽成可复用的自定义 Hook**,再配合 `useMemo`/`useCallback` 优化。

---

## 〇、什么是自定义 Hook

自定义 Hook 就是一个**名字以 `use` 开头、内部调用了其它 Hook 的普通函数**。
它的价值:把"会在多个组件里重复的、带 state 的逻辑"抽出来复用。

```
普通函数      → 复用"无状态"的计算逻辑(比如格式化日期)
自定义 Hook   → 复用"有状态"的逻辑(比如 localStorage 读写、表单处理、订阅)
```

两条铁律:

1. **名字必须以 `use` 开头**(React 靠这个识别它是 Hook,才能检查规则)。
2. **只能在组件顶层 / 其它 Hook 里调用**,不能写在 if、循环、嵌套函数里。

---

## 一、自定义 Hook:useLocalStorage 逐行讲

它的目标:做一个**用法和 `useState` 一模一样**,但数据会**自动存进 localStorage、刷新页面不丢**的 Hook。

```ts
// src/hooks/useLocalStorage.ts
export function useLocalStorage<T>(
  key: string,         // localStorage 的键名
  initialValue: T,     // 没有存过时的默认值
): [T, (value: T) => void] {   // 返回 [值, 设值函数],和 useState 同款
  const [storedValue, setStoredValue] = useState<T>(initialValue);
  const [mounted, setMounted] = useState(false);  // 是否已在浏览器挂载

  // ① 挂载后,从 localStorage 读出已存的值
  useEffect(() => {
    setMounted(true);
    try {
      const item = window.localStorage.getItem(key);
      if (item) {
        setStoredValue(JSON.parse(item));  // 字符串 → 对象/数组
      }
    } catch (error) {
      console.error("Error reading from localStorage:", error);
    }
  }, [key]);   // key 变了就重新读

  // ② 设值时:既更新 state,又写回 localStorage
  const setValue = useCallback(
    (value: T) => {
      try {
        setStoredValue(value);
        if (mounted && typeof window !== "undefined") {
          window.localStorage.setItem(key, JSON.stringify(value));  // 对象/数组 → 字符串
        }
      } catch (error) {
        console.error("Error saving to localStorage:", error);
      }
    },
    [key, mounted],
  );

  return [storedValue, setValue];   // 像 useState 一样返回元组
}
```

### 三个设计要点

**1. 泛型 `<T>`——让 Hook 能存任意类型**
`useLocalStorage<string[]>(...)` 存字符串数组,`useLocalStorage<number>(...)` 存数字。
`<T>` 是占位符,调用时填什么类型,返回值和 `setValue` 参数就都是那个类型,**类型安全**。

**2. 为什么需要 `mounted` 这个开关?——SSR(服务端渲染)问题**
Next.js 会先在**服务器**上渲染组件,而服务器**没有 `window`**(localStorage 属于浏览器)。
所以:
- 初始值统一用 `initialValue`(服务器和浏览器首屏一致,避免 hydration 不匹配报错)。
- 等 `useEffect` 跑了(只在浏览器跑)→ `setMounted(true)`,这之后才真正去读/写 localStorage。

**3. `JSON.stringify` / `JSON.parse` 这对搭档**
localStorage **只能存字符串**。所以存的时候用 `JSON.stringify` 把数组/对象转成字符串,
读的时候用 `JSON.parse` 转回来。`try/catch` 是防止存了坏数据时解析崩溃。

---

## 二、消费端:NotesWidget 怎么用这个 Hook

```jsx
function NotesWidget() {
  // 用起来和 useState 一模一样,但会自动持久化!
  const [notes, setNotes] = useLocalStorage<string[]>("tutorial-notes", []);
  const [newNote, setNewNote] = useState("");
  ...
}
```

**最大的好处:** 调用方完全不用关心 localStorage、SSR、JSON 转换那些细节——
那些复杂逻辑都被**封装在 Hook 内部**了。这就是自定义 Hook 的意义:**隐藏复杂度,暴露简单接口。**

---

## 三、用 useMemo 做"昂贵计算"优化

```jsx
// ❌ BAD:每次渲染都重算(哪怕 notes 没变)
// const noteStats = {
//   total: notes.length,
//   long: notes.filter((note) => note.length > 10).length,
//   avgLength: notes.reduce((s, n) => s + n.length, 0) / notes.length,
// };

// ✅ GOOD:useMemo 只在 notes 变化时重算
const noteStats = useMemo(() => {
  console.log("📊 Calculating note statistics...");  // 只在 notes 变时才打印
  return {
    total: notes.length,
    long: notes.filter((note) => note.length > 10).length,
    avgLength:
      notes.length > 0
        ? Math.round(notes.reduce((sum, n) => sum + n.length, 0) / notes.length)
        : 0,   // 防止除以 0 得 NaN
  };
}, [notes]);   // 依赖数组:notes 变了才重算
```

> 🐍 类比 Python 的 `@lru_cache`。
> 那个 `console.log` 是**教学小心机**:打开控制台,你会发现只有改 notes 时才打印,
> 输入框打字(改的是 `newNote`)时不打印——直观证明 useMemo 真的拦住了无谓计算。

---

## 四、用 useCallback 包裹操作函数

```jsx
const addNote = useCallback(() => {
  if (newNote.trim()) {
    setNotes([...notes, newNote.trim()]);  // 展开旧数组 + 新笔记
    setNewNote("");                        // 清空输入框
  }
}, [notes, newNote, setNotes]);            // 读了这三个,都要列入

const clearNotes = useCallback(() => {
  setNotes([]);
}, [setNotes]);
```

注意 `setNotes` 也被列进依赖了——因为它来自 `useLocalStorage`,严格说每次渲染可能是新函数,
列进去最安全。(本项目里它被 `useCallback` 稳定住了,所以其实不会变。)

---

## 五、完整实现流程(从零搭)

1. **抽 Hook**:把 localStorage 读写逻辑抽进 `useLocalStorage`,对外只暴露 `[值, 设值]`。
2. **消费**:`const [notes, setNotes] = useLocalStorage("tutorial-notes", [])`,像 useState 一样用。
3. **派生数据**:统计信息(总数/长笔记数/平均长度)用 `useMemo`,依赖 `[notes]`。
4. **操作函数**:`addNote` / `clearNotes` 用 `useCallback` 包裹。
5. **渲染**:统计卡片(三宫格)、输入框 + 添加按钮、列表(空态提示 vs `notes.map`)。
6. **持久化生效**:刷新页面,笔记还在——因为每次 `setNotes` 都写进了 localStorage。

---

## 六、数据流

```
首次加载
  → useLocalStorage 的 useEffect 跑 → 从 localStorage 读出旧笔记 → setStoredValue
输入框打字
  → setNewNote(改的是临时输入,不触发 noteStats 重算)
点 Add
  → addNote → setNotes([...notes, 新笔记])
            → useLocalStorage 内部:更新 state + 写入 localStorage
            → notes 变 → useMemo 重算 noteStats → 统计卡片刷新 + 列表刷新
刷新页面
  → 笔记依然在(从 localStorage 恢复)
```

---

## 七、本节小结

- **自定义 Hook** = 名字以 `use` 开头、复用"带状态的逻辑";它把复杂度(localStorage、SSR、JSON)藏在内部,对外只给简单接口。
- **泛型 `<T>`** 让 Hook 类型安全地存任意数据。
- **`mounted` 开关 + `initialValue`** 解决 Next.js SSR 下 `window` 不存在、hydration 不匹配的问题。
- **useMemo** 缓存派生统计,**useCallback** 缓存操作函数,依赖数组都遵循同一条铁律:**函数体读到的会变的外部变量,都要列进去**。
