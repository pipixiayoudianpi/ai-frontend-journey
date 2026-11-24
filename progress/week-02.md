# Week 2（2026-06-15 ~ 2026-11-24）

> Day 8 之后中断了一段时间，11 月初恢复，Day 9-14 集中完成。

## 阶段
第 0 阶段：地基补强

## 主题
- TypeScript 入门：6 大核心点 + 用类做数据建模
- Vue 3 实战：Vite + Vue 3 + TS + Pinia + Vue Router + UnoCSS 从零搭一个聊天应用
- Vue 响应式原理：依赖追踪、ref / reactive / shallowRef 的取舍

## 完成情况

### Day 8 — TypeScript 6 大核心点
基础类型与类型推导、数组和对象、函数参数和返回值、interface 与联合类型、可选属性与 readonly、泛型。

综合练习：用户管理模块，挑战题独立写出 `pluck<T, K extends keyof T>(items: T[], key: K): T[K][]`。

### Day 9 — 用类做数据建模
写了 `ConversationManager`：`class`、`private`、`Map<K, V>`、`Date`、`crypto.randomUUID()`，扩展了 delete / getMessages / search。

### Day 10 — 项目脚手架
用 Vite 搭 [conversation-chat](https://github.com/pipixiayoudianpi/conversation-chat)，接入 Pinia、Vue Router、UnoCSS，配置 `@` 路径别名，按 types / stores / composables / views / components / router 分目录。

顺带修好了一个环境问题：Node 是 x64 版本，跑在 arm64 Mac 的 Rosetta 上，导致 Rolldown、oxc、Rollup 的原生二进制反复找不到。换成 arm64 原生 Node 后解决。

### Day 11 — Pinia store 和组件拆分
- 把 ConversationManager 改写成 Pinia Setup Store（state → ref，getter → computed，方法 → 函数）
- 拆出 ConversationList、MessageList，用 `defineProps` / `defineEmits` 的类型写法
- 页面负责和 store 打交道，组件只通过 props 和事件通信

### Day 12 — 让应用能用
- MessageInput：Enter 发送、Shift+Enter 换行，`isComposing` 判断避免中文输入法选字时误发
- 模拟 AI 回复：闭包捕获会话 id，回调里先确认会话还存在
- 自动滚到底部：`watch` 多数据源 + `nextTick`
- 搜索：store 里的 computed 层层叠加，搜索框用 `defineModel` 双向绑定
- 持久化：第一个组合式函数 `useLocalStorage`，`watch` 的 `deep` 选项

### Day 13 — 响应式原理实验
在 ts-warmup 里用 `@vue/reactivity` 的 `effect` 做了 6 个实验和 1 道思考题：依赖追踪、reactive、解构丢失响应式、toRefs、shallowRef、解构出对象拿到的是引用。笔记见 [vue-reactivity.md](../notes/vue-reactivity.md)。

### Day 14 — 用回项目
- `storeToRefs` 正确解构 store
- Vue 3.5 的 props 解构（编译器改写回 `props.xxx`）
- shallowRef 的三个使用场景，ref / reactive / shallowRef / markRaw 怎么选
- 补缺口：第二个组合式函数 `useAutoScroll`；用 `watchEffect` 让浏览器标签页标题跟着当前会话变

### 可见成果
- [conversation-chat](https://github.com/pipixiayoudianpi/conversation-chat)：能新建、切换、删除、搜索会话，发消息、模拟回复、自动滚动，刷新不丢数据。每个功能一次规范的 commit
- [notes/vue-reactivity.md](../notes/vue-reactivity.md)：响应式原理笔记，包含我做错的题和踩过的坑

## Week 2 验收对照

- [x] TS 6 大基础点能不查资料写出来
- [x] 用 Vue 3 + TS + 组合式 API 完整做一个小项目（没有可以重写的旧项目，改为从零做 conversation-chat）
- [x] ref / reactive / computed / watch / watchEffect 用法清楚
- [x] 写过至少 2 个自定义组合式函数（useLocalStorage、useAutoScroll）

## 踩过的坑

### TypeScript
1. **`arr[0]` 的类型是 `T | undefined`**：tsconfig 开了 `noUncheckedIndexedAccess`。优先改类型反映现实，慎用 `!`
2. **`Map.has()` 不能帮 `get()` 窄化类型**：直接 `const x = map.get(k); if (!x) return` 一次搞定
3. **箭头函数 `=> {` 忘写 return**：filter 回调返回 undefined，结果全被过滤掉，而且 TS 不报错
4. **`JSON.parse(...) as T` 是类型系统的盲区**：存进 localStorage 的 Date 读回来变成字符串，类型却还写着 Date。改为存数字时间戳
5. **ESM import 在 CommonJS 文件里报 ts(1295)**：ts-warmup 的 package.json 缺 `"type": "module"`

### Vue
6. **解构 store / reactive 会断开响应式**：要用 `storeToRefs` / `toRefs`
7. **解构出对象拿到的是引用**：对象内部改了看得到，整个替换就看不到。需要过一段时间再用的数据，存 id 再查，不要攥着对象
8. **`watch([messages, messages.length])` 静默失败**：编译器只检查直接传 prop 的写法，数组里的漏检；类型检查也放行；只有浏览器 Console 里有黄色 `[Vue warn]`。watch 的数据源只能是 ref、reactive 对象或函数
9. **Vetur 报"找不到模块"**：Vue 3 项目要用 Vue - Official 插件，Vetur 在这个工作区禁用

### 环境和工具
10. **x64 Node 跑在 arm64 Mac 上**：原生二进制按 `process.arch` 找，安装时和运行时架构对不上就报 "Cannot find native binding"。同一个错出现两次就该找根因，不再打补丁
11. **脚本里 `nvm use` 改不了当前终端**：子进程改不了父进程的环境变量，要新开终端
12. **GitHub 登录账号和 commit 作者是两回事**：个人项目在仓库里单独设 `git config user.email`，全局保持公司邮箱
13. **又一次 typo**：`pnpm tsx prctice.ts`。用 Tab 补全

### 工程习惯
14. **质疑预期**：Claude 给的预期输出说搜 "ts" 能匹配 "TypeScript"，我逐字母检查发现 "typescript" 里根本没有 "ts" 这个子串
15. **注释掉的旧代码直接删**：代码都在 Git 历史里，不用注释保留

## 心智模型沉淀

- **响应式 = 一张依赖表**：读的时候登记（track），改的时候通知（trigger）。模板渲染、computed、watch、watchEffect 底层都是 effect
- **computed 算值，watch 做事**：要一个新值用 computed，要执行动作（滚动、存储、请求）用 watch；只想让副作用一直和数据同步，用 watchEffect
- **容器组件和展示组件**：页面和 store 打交道，组件只管 props 进、事件出，单向数据流
- **不是所有状态都放 store**：只有多个组件共享的才放，输入框草稿留在组件自己里面
- **数据结构按使用方式选**：前端要渲染列表、要序列化，用数组比 Map 合适
- **要存储、要传输的数据只用 JSON 支持的类型**：时间存时间戳，显示时再转 Date
- **先用最简单的写法**：没遇到问题之前不上 shallowRef，过早优化只会让代码变复杂
- **能运行 ≠ 写对了**：类型检查、编译器、运行时警告各拦一部分，写完扫一眼浏览器 Console

## 下周计划

Week 3：从零搭个人博客并部署上线
- 技术栈：Vue 3 + Vite + TS、UnoCSS、Vue Router、Markdown 文章渲染，部署到 Vercel
- 功能：首页、文章列表、文章详情（代码高亮、目录）、暗黑模式、基础 SEO
- 内容：把 Week 1-2 的学习记录整理成前 3 篇博文
