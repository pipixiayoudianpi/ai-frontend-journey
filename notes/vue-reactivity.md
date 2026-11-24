# Vue 3 响应式原理笔记

> Week 2 Day 13（2026-11）。在 `ts-warmup` 里用 `@vue/reactivity` 的 `effect` 做了 6 个实验和 1 道思考题。

## 一句话总结

Vue 维护着一张"谁依赖谁"的表：**读的时候登记，改的时候通知**。

- **track（登记）**：effect 运行期间，每读一个响应式数据，就在表里记一笔"这个数据被这个 effect 用到了"
- **trigger（通知）**：数据被修改时，查表找出所有用过它的 effect，重新执行

组件的模板渲染、`computed`、`watch`，底层都是 effect。界面"自动刷新"就是这张表在起作用。

- `computed` = 带缓存的 effect，依赖没变就直接返回上次的结果
- `watch` = 数据变了不重新渲染，而是执行我给的回调

## 六个实验

| 实验 | 代码要点 | 结果 | 结论 |
|---|---|---|---|
| 1 | `effect` 里读 `count.value`，改两次 | 打印 3 次（初始 1 次 + 每改一次 1 次） | effect 创建时立刻执行一次，之后依赖一变就重跑 |
| 2 | effect 只读 `a`，改 `b` 再改 `a` | 改 `b` 不打印，改 `a` 打印 ✓ 我猜对了 | 没被读过的数据不在依赖表里，改了也不触发。Vue 只刷新真正用到它的地方 |
| 3 | `reactive` 对象，改 `state.age` | 打印 | reactive 把对象包成 Proxy，访问属性就能被拦截，不需要 `.value` |
| 4 | `const { age } = state`，再改 `state.age` | 解构的 effect 只在**创建时**打印一次 `age = 26`，改 `state.age` 之后**不再打印**；实验 3 的 effect 打印 ✓ 我猜对了 | 解构那一刻读出的是当时的值（一个普通数字），之后跟 state 毫无联系 |
| 5 | `const { age: ageRef } = toRefs(state)` | 改 `state.age` 或改 `ageRef.value`，两边的 effect 都打印 | toRefs 生成的 ref 不存值，只是转接：读写 `.value` 实际读写的是 `state.age` |
| 6 | `shallowRef({ count: 0 })` | `big.value.count++` 不打印，`big.value = {...}` 打印 ✓ 我猜对了 | shallowRef 只监听 `.value` 有没有被整个替换，里面的对象不是 Proxy。数据其实变了，只是没人收到通知 |

### 怎么判断 effect 有没有响应修改

**effect 创建时一定会先执行一次**，所以"打印过"不代表"响应了"。要看的是**修改之后它有没有再打印**。

我复习时就被实验 4 绕进去过：看到 `[effect-解构] age = 26`，以为它响应了修改。其实那是创建时的那一次，值还是修改前的 26。

调试技巧：在修改前后加标记，夹在中间的输出才是对这次修改的响应。

```ts
console.log("--- 准备修改 ---");
state.age = 27;
console.log("--- 修改完成 ---");
```

### 为什么 ref 要写 `.value`

JS 没办法拦截"读一个普通变量"，只能拦截"读一个对象的属性"。所以数字、字符串这类基本类型，ref 把它装进一个盒子，盒子上有个 `value` 属性，读写 `.value` 时 Vue 才能拦截。**`.value` 就是盒子的把手**。

在 `<script setup>` 里声明的顶层 ref，模板里会自动拆箱，所以模板里不用写 `.value`，script 里要写。

## ❌ 我做错的：思考题

```ts
const user = reactive({ profile: { name: "A" } });
const { profile } = user;
effect(() => {
  console.log(`name = ${profile.name}`);
});
user.profile.name = "B"; // 会打印吗？
user.profile = { name: "C" }; // 会打印吗？
```

- **我的猜测**：两个都会打印
- **实际结果**：第一个打印 B，第二个不打印

**错在哪**：我以为解构出对象和解构出数字不一样，解构出对象就"一直连着"。其实解构出对象时，我拿到的是一个**引用**，指向当时那个对象。

```
解构那一刻：
user.profile ──→ 对象A { name: "A" } ←── profile（我的变量）

user.profile.name = "B"：改的是对象 A 本身
user.profile ──→ 对象A { name: "B" } ←── profile      ✓ 同一个对象，看得到

user.profile = { name: "C" }：user.profile 换成指向新对象 C
user.profile ──→ 对象C { name: "C" }
                 对象A { name: "B" } ←── profile      ✗ 我还拿着旧的
```

effect 登记的是"对象 A 的 name"，对象 A 没变，所以不触发。就算重跑，读到的也还是 B。

**比喻**：解构出对象，等于拿到了一个房间的钥匙。房间里的家具怎么搬我都看得到；但房东把门牌号换成另一个房间，我手里的钥匙开的还是原来那间。

| 解构出来的是 | 我拿到的 | 对象内部被修改 | 整个被替换 |
|---|---|---|---|
| 基本类型（实验 4） | 当时那个值的拷贝 | 不适用 | 看不到 |
| 对象（思考题） | 指向同一个对象的引用 | **看得到** | 看不到 |

**共同点**：解构之后，和原来那个父对象的联系就断了。

**带走的经验**：需要过一段时间再用的数据，**存它的 id，用的时候再去查最新的**，不要长期攥着一个对象引用。Day 12 写模拟回复时，setTimeout 里存 `convId` 再重新查，而不是存 `conv` 对象，就是这个原因。如果存的是对象，会话在等待期间被删或者数组被整体替换，往旧对象上加消息不会报错，界面也不刷新，消息悄悄丢失。

## 💡 我当时没看懂的语法：解构重命名

```ts
const { age: ageRef } = toRefs(state);
// 等价于
const refs = toRefs(state);
const ageRef = refs.age;
```

冒号左边是源对象的属性名，右边是新变量名，读作"把 age 存进 ageRef"。

容易混的地方：花括号在等号**右边**是创建对象（`{ age: 25 }`，冒号右边是值），在等号**左边**是解构（冒号右边是新变量名）。

这里要改名，一是 `age` 这个名字在实验 4 已经用过，`const` 不能重复声明；二是名字带 `Ref` 能提醒自己它要写 `.value`。

## 和项目代码的联系

- **别直接解构 store**：`const { conversations } = store` 就是实验 4，拿到的是当时的值。要解构用 `storeToRefs`，原理同实验 5 的 `toRefs`
- **computed 里只读不写**：在 computed 里 `sort()` 原数组，等于在 effect 里修改它自己依赖的数据，会再次触发自己，严重时死循环。所以要 `[...arr].sort()`
- **watch 的数据源要写成函数** `() => props.messages.length`：直接传一个数字，watch 拿到的只是当时的值，跟实验 4 一个道理

## 环境踩坑

- **TS 报 ts(1295)，ESM 的 import 不能写在 CommonJS 文件里**：`ts-warmup/package.json` 没有 `"type": "module"`，Node 默认按 CommonJS 处理；tsconfig 又开了 `verbatimModuleSyntax`，不允许偷偷把 import 翻译成 require。加上 `"type": "module"` 解决。Day 8、9 的文件没报错，是因为里面一个 import 都没有
- **换成 arm64 Node 后，旧项目要重装依赖**：ts-warmup 的 node_modules 是 x64 Node 时代装的，里面 esbuild 的原生二进制是 x64 版，删掉 node_modules 和锁文件重装即可

---

# Day 14：把响应式用回项目

> 在 conversation-chat 里实践，commit：`refactor: 用 storeToRefs 和 props 解构重构 ChatView、MessageList`

## storeToRefs：正确解构 store

亲眼看到的 bug：在 ChatView 里写 `const { activeConversation } = store`，切换会话时**消息区跟着变，标题卡住不动**。就是实验 4：解构那一刻拿到的是当时的会话对象。

```ts
const store = useConversationStore();

// 数据（state 和 getter）：用 storeToRefs，原理同实验 5 的 toRefs
const { conversations, activeConversation, searchQuery } = storeToRefs(store);

// 函数（action）：直接解构，函数不会变，不需要响应式
const { createConversation, addMessage } = store;
```

- script 里要写 `.value`（`searchQuery.value = ""`），模板里不用，`<script setup>` 的顶层 ref 在模板里自动拆箱
- 直接写 `store.xxx` 也完全没问题，用到的字段不多时更简单。关键是知道**为什么**不能 `const { x } = store`

## Vue 3.5 的 props 解构：一个例外

```ts
const { messages } = defineProps<{ messages: Message[] }>();
```

按实验 4，这样解构应该会断开，但从 Vue 3.5 开始它依然是响应式的。原因：**这不是运行时的解构**，Vue 编译器会把代码里所有的 `messages` 改写回 `props.messages`，等于替我把完整路径补了回去。只对 `<script setup>` 里的 `defineProps` 有效。

watch 里必须写成 `() => messages`，不能直接传 `messages`。

## ❌ 我踩的坑：watch 数组里直接放值，编译器漏检、运行时静默失败

我把 `watch([() => messages, () => messages.length], ...)` 改成了：

```ts
watch([messages, messages.length], ...)
```

- VS Code 不报错，`vue-tsc -b` 不报错，页面正常运行
- 但**切换会话后不再自动滚动**，watch 实际上失效了，没有任何红色报错

**为什么编译器没拦住**：Vue 编译器只检查 `watch(messages, ...)` 这种"prop 直接作为第一个参数"的写法，放在数组里它认不出来。（Claude 一开始说"编译器会报错"，说得太绝对，被我实测推翻了。）

**为什么类型检查没拦住**：watch 的类型定义允许直接传一个对象当数据源，数组本身就是对象，类型上挑不出毛病。

**编译后实际是什么**：

```ts
watch([props.messages, props.messages.length], ...)
//      ↑ 组件创建时那个会话的消息数组引用 → 思考题的"旧钥匙"
//                      ↑ 当时的条数，一个普通数字 → 实验 4
```

两个 Day 13 的坑，在一行代码里同时出现。不切换会话时可能还能滚（Vue 会深度监听那个旧数组），一切换就彻底不滚。

**其实有提示**：浏览器 Console 里有黄色的 `[Vue warn] Invalid watch source`，只是我没去看。

**教训**：

- **能运行 ≠ 写对了**。类型检查、编译器、运行时警告，每一层只能拦住一部分问题
- 写完代码除了看页面效果，**扫一眼浏览器 Console**，黄色的 `[Vue warn]` 也要读
- watch 的数据源只能是 **ref、reactive 对象、或者一个函数**。解构出来的 props、对象上的某个属性，一律包成 `() => xxx`

## shallowRef 和选择规则

ref 装对象默认是深层的：被访问到的每一层都套 Proxy、每次读取都登记依赖。数据小时感觉不到，这三种场景会变成负担：

1. **大量数据，只读展示、整体刷新**（比如报表页一次拉 1 万行）。Vue 2 里的 `Object.freeze(bigList)` 就是干这个的，Vue 3 用 shallowRef
2. **第三方库实例**（ECharts、地图、编辑器）。被 Proxy 包住可能导致库内部判断出错
3. **高频更新**（Week 9 的 AI 流式输出，一秒几十次追加）

shallowRef 只在 `.value` 整体替换时通知，修改有两种写法：

```ts
rows.value = [...rows.value, newRow]; // 换成新数组
// 或者
rows.value.push(newRow);
triggerRef(rows); // 原地改完，手动通知
```

**我的项目不改 shallowRef**：数据量小，而且 store 里大量原地修改（`conv.messages.push`、`conv.updatedAt = ...`），改了反而容易漏掉通知。**先用最简单的写法，真遇到问题再优化**，没问题就优化叫过早优化。

| 用什么 | 什么时候用 | 要注意 |
|---|---|---|
| **ref** | **默认首选** | script 里要写 `.value` |
| **reactive** | 一组固定字段、只改不换的对象，比如表单 | 只能装对象；不能整体替换；解构会断开 |
| **shallowRef** | 大量数据整体刷新、第三方实例、高频更新 | 原地修改不触发 |
| **markRaw** | 永远不需要响应式的对象 | 放进响应式结构里也不会被代理 |

官方推荐 ref 优先：reactive 的两个限制（不能整体替换、解构断开）正好就是实验 4 和思考题的坑。

## 小测验（我的答案）

1. 搜索关键词字符串 → **ref** ✓ 基本类型只能用 ref
2. 8 个字段的新建客户表单，提交后清空 → **reactive**，可以，但要会清空：

```ts
function createEmptyForm() {
  return { name: "", phone: "", email: "", tags: [] as string[] };
}
const form = reactive(createEmptyForm());

// 清空：不能 form = reactive(...)，只能把新值拷进原对象
Object.assign(form, createEmptyForm());
```

  用 ref 的话清空就是 `form.value = createEmptyForm()`，更直观。两种都对。

  **一定要用函数生成初始值**，不要用一个共享的常量。`Object.assign(form, EMPTY_FORM)` 会让 `form.tags` 和 `EMPTY_FORM.tags` 指向同一个数组，用户加的标签会污染初始值，下次清空清不掉。根源跟思考题一样：对象和数组是引用。

3. ECharts 图表实例 → **shallowRef** ✓ 如果模板根本不需要知道它，直接 `let chart = null` 就行，不是每个变量都要响应式
4. 1 万行只读数据，整体刷新 → **shallowRef** ✓

（我当时没写理由。以后答题把理由写出来，面试时理由比答案更重要。）
