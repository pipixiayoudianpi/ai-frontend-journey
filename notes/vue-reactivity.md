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
| 4 | `const { age } = state`，再改 `state.age` | 解构的 effect 不打印，实验 3 的 effect 打印 ✓ 我猜对了 | 解构那一刻读出的是当时的值（一个普通数字），之后跟 state 毫无联系 |
| 5 | `const { age: ageRef } = toRefs(state)` | 改 `state.age` 或改 `ageRef.value`，两边的 effect 都打印 | toRefs 生成的 ref 不存值，只是转接：读写 `.value` 实际读写的是 `state.age` |
| 6 | `shallowRef({ count: 0 })` | `big.value.count++` 不打印，`big.value = {...}` 打印 ✓ 我猜对了 | shallowRef 只监听 `.value` 有没有被整个替换，里面的对象不是 Proxy。数据其实变了，只是没人收到通知 |

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

## 下一步（Day 14）

- `storeToRefs` 正确解构 store
- Vue 3.5 的 props 解构为什么不会丢响应式
- shallowRef 在真实项目里什么时候该用
