# 大数据 state 与 useModel 深比较方案

> 机制层结论均以运行时代码与官方 tarball diff 钉死；方案来自甘特图等大数据可视化页面的实际踩坑，可跨项目复用。

## TL;DR

大数据（几千~几万元素）放 React state / umi model 时，**唯一存在 O(N×M) 比较开销的位置是 umi useModel 的 selector 深比较层**；React 自身的 useMemo/useCallback/memo 全是浅比较，无需优化也无可优化。两条解法路线：

| 数据形态 / 场景 | 推荐方案 | 买到什么 |
| --- | --- | --- |
| Map 结构 | **ref + stamp（唯一解）** | 修正漏更新（正确性） |
| 数组/对象 + 有命令式消费者（图表库冻结回调、事件回调取最新值） | **ref + stamp** | 正确性 + O(1) 比较 |
| 数组/对象 + 纯响应式消费、不想引入双写纪律 | **valueOf 别名（aliasData）** | O(1) 比较，保持纯 state 写法 |
| 小数据 | 直接 state | 不需要方案 |

## 卡顿根因与开销公式（实践来源：甘特图 task 组件、列表单元格场景）

卡顿由多个正交因子叠加，总开销：

```
比较总成本 = 订阅点数量 × model 更新频率 × 单次深比较成本
```

- **根因一：useModel 引用过多（订阅扇出）**——同一 namespace 被大量**小粒度组件**直接订阅（甘特图的 task、列表的单元格这类动辄成百实例的组件）。model 任何数据更新触发 update，**几十处订阅回调各自执行 selector + fast-deep-equal 比对**，全部叠加在交互热路径上。
  → 对症：**减少 useModel 的使用**。订阅点上移收敛——容器组件订阅一次、数据经 props 下发；小粒度组件（task / 单元格级）不直接 useModel。
- **根因二：数据过大（单次比对成本）**——单次 fast-deep-equal 对大数组/大对象是 O(N×M) 递归比较，且最贵路径恰是「内容没变」的全量确认。
  → 对症：**对数据特殊处理，避免 useModel 递归比对**——见方案一（ref + stamp）/ 方案二（valueOf 别名）。
- **根因三：model 混装（更新频率）**——甘特 model 40+ 状态混在一个 namespace，右键菜单开合这类高频小更新也扇出全部大数据订阅者。
  → 对症一（优先，零结构成本）：**selector + ref+stamp**——大数据订阅者的 selector 返回 `{ref, stamp}`，被无关更新扇出时 2-key + 数字判等即不渲染，比较 O(1)，主要伤害基本消除。
  → 对症二（结构性保险）：**高频小状态（弹窗开关、选中态）与大数据拆进不同 model**——把无关订阅者从扇出名单整个摘掉，不依赖任何组件的 selector 纪律（漏写 selector 直接拿全 model 的组件，任何更新必全 model 深比较 + 必重渲染，混装的杀伤力全留给这类漏网组件）。拆分按变化源聚类：纯 UI 开关拆走，与数据联动的状态（如勾选驱动排序）留下。

三刀各砍一个因子：**订阅收敛**砍数量、**model 拆分**砍频率、**数据方案**砍单次成本。优先级：先砍订阅点（零行为风险、收益立现），数据方案再按形态选择。

## 一、机制层：三层比较的真相

### 1. React 层 —— 浅比较，永远免费

`useMemo`/`useCallback` 的依赖比较是 `areHookInputsEqual`：对依赖数组**逐项 `Object.is`**。几万元素的数组放在依赖里，比较就是一次引用判等，O(依赖项数)、与数据大小无关。`React.memo` 的 props 比较同理（shallowEqual）。

大数据在纯 React 体系里的真实成本是「引用变化引发重算」的**粒度问题**，不是比较问题。任何声称在此层减少比较开销的方案都没有收益空间。

### 2. umi useModel selector 层 —— fast-deep-equal 深比较，开销取决于数据形态

运行时链路（`src/.umi/plugin-model/index.tsx`）：

```ts
const handler = (data: any) => {
  const currentState = selector(data);          // model 任何 state 更新都会执行
  if (!isEqual(currentState, stateRef.current)) // fast-deep-equal 深比较
    setState(currentState);                     // 判不等才 setState → 组件重渲染
};
```

两个关键认知：

- **isEqual 判等直接决定组件重不重渲染**（判等 → 不 setState），不只是保护下游 memo；
- **useModel 一律带 selector，无论数据大小都有收益**：不带 selector 直接拿全 model，model 任何字段变化都判不等 → 必然 setState → **必然重渲染**；带 selector 只选关心的字段，无关字段的更新比较后判等 → 不 setState → 组件不渲染，把「付渲染钱」降成「付比较钱」。selector 挡不住扇出（handler 仍执行），其比较成本按返回值形态分三档：
  - 返回**单字段原始引用**（`(m) => m.contextMenu`）→ isEqual 首行 `===` 短路，**零成本**——混装 model 下小状态消费者的完全体：被扇出也只付一次 `===`；
  - 返回**对象字面量包小字段**（`{ a: m.a, b: m.b }`）→ 每次深比较这个小对象（便宜但非零），值等仍防渲染；
  - 返回**大数据** → 深比较按形态爆炸，selector 在此无解，需本文方案一/二；
- 开销 = **每次该 namespace 任何 state 更新 × 每个订阅组件 × 一次 selector 返回值深比较**。model 里状态越多、交互越频繁（甘特 model 40+ 状态，右键/勾选/拖拽全是高频小更新），深比较被触发的次数越多。

fast-deep-equal **3.1.3**（npm 实际产物）按形态分流，行为差异巨大：

| 形态 | 行为 | 后果 |
| --- | --- | --- |
| 数组（源码 14-20 行） | 长度相同则**逐项递归真比较** | O(N×M) 真开销；最贵路径恰是「内容没变」的全量确认——正是高频小更新时的常态 |
| 普通对象（28-41 行） | 逐 key 递归 | 同上 |
| **Map / Set** | **无分支** → 走 `Object.keys(map)` 为空 → **恒判等** | 比较免费，但**数据变了也判等 → 漏更新（静默不渲染，极难排查）** |
| 自定义 `valueOf` 的对象（25 行） | `a.valueOf() === b.valueOf()`，直接 return | 快捷短路——valueOf 别名方案的基础 |

⚠️ **易踩的考证坑**：`node_modules/fast-deep-equal/es6/index.js` 里**有** Map/Set 分支，但 `src/.umi/plugin-model` 经 `require.resolve` 引用的是根 `index.js`（无 Map 分支）。看源码先确认运行时 import 的是哪份文件，别被 es6 目录误导。验证手段：拉 registry tarball 与本地 diff，锁文件 integrity 与官方一致即未被篡改。

### 3. 常见误判：当初「ref+stamp 是为了减少比较开销」的动机辨析

- React 浅比较层：无开销可减（见 1）；
- Map 形态：恒判等，比较本来就 O(1)（是正确性 bug 不是性能问题）；
- 数组/对象形态：深比较是真开销，stamp 才有真实性能收益。

即：**方案该用，但收益记账要按形态分开**——Map 买正确性，数组买性能，「减少比对」这个笼统说法两种形态都只说对了一半。

## 二、方案一：ref + stamp（甘特图实践）

```ts
// 数据放 ref（篮子），版本号放 state（铃铛）
const dataRef = useRef<RawData>({});
const [dataStamp, setDataStamp] = useState(0);

// 更新：写 ref → 拨 stamp
dataRef.current = mergeData(changed, dataRef.current);
setDataStamp(performance.now());

// 消费（渲染时机）：selector 返回 { ref, stamp }，深比较恒 O(1)
const { dataRef: latestRef } = useModel('xxx', (m) => ({
  dataRef: m.dataRef,   // 引用恒定
  stamp: m.dataStamp,   // 数字必不等，击穿 Map 恒判等
}));
return latestRef.current;
```

### 职责分工（两个组件各买一件事，不要混记）

- **ref**：供「不随 render 刷新的消费者」读最新值——图表库挂载后冻结的交互配置（闭包恒为首次挂载版）、事件回调、setTimeout/Promise 回调。数据放 state 则这些旧闭包永远拿到旧快照，此问题 state 无解（除非镜像 ref 双写）。
- **stamp**：数字比较必不等——对 Map 形态击穿恒判等修漏更新；对数组形态把 O(N×M) 深比较降为 O(1)。

### 收益结账（按形态）

| 收益项 | Map 形态 | 数组/对象形态 |
| --- | --- | --- |
| 正确性（防 Map 恒判等漏更新） | ✅ 核心收益 | 不适用 |
| 性能（深比较 O(N×M) → O(1)） | 本来就 O(1) | ✅ 真实收益 |
| 供冻结消费者读最新 | ✅ | ✅ |
| React 浅比较层 | 无收益也无需 | 无收益也无需 |

### 成本（纪律税）

- **双写纪律**：改 `ref.current` 忘了 `setStamp` → 静默不刷新，极难排查。须配「渲染时机用 hook 快照 / 事件回调用 ref」的双轨消费规则并写进模块文档；
- 粒度粗：stamp 一变全部订阅者重刷（数据一变本来就要全量重算的场景无实害）；
- 理论缺陷：ref 与 stamp 双源写入在并发渲染下不保证成对一致；时间戳精度裁剪后极小概率撞同值。同步模式下无实害。

## 三、方案二：valueOf 别名（不使用 ref 的路线）

数据照常放 state、照常被 selector 返回，但给数据对象重写 `valueOf`，命中 fast-deep-equal 的快捷路径（源码 25 行：`if (a.valueOf !== Object.prototype.valueOf) return a.valueOf() === b.valueOf();`），把深比较短路成一次数字判等：

```ts
export const aliasObject = (obj: object) => {
  // 重写对象的 valueOf，避免 useModel 的递归比较
  const pureObject = Object.create({});
  const current = Date.now();
  Object.defineProperty(pureObject, 'valueOf', {
    value: function valueOf() {
      return current; // 创建时刻的时间戳，两次更新必然不同
    },
    enumerable: false,
  });
  return Object.assign(pureObject, obj);
};

export const aliasArray = (arr: object[]) => {
  const [firstData, ...rest] = arr;
  if (firstData && typeof firstData === 'object') {
    const newData = aliasObject(firstData);
    return [newData, ...rest];
  }
  return arr;
};
```

原理：数组逐项比较时，第一项（被 alias）的 valueOf 返回新时间戳、旧项 valueOf 返回旧值 → 首项即判不等 → 整个数组 O(1) 判不等 → 必然 setState。

### 效果与语义

- 数组/普通对象形态的深比较 → O(1)；
- 语义与 stamp 等价：**无条件判不等**，放弃「内容没变跳过渲染」（数据没变的重拉也会重渲染订阅组件）；
- 保持纯响应式写法：无 ref 双写纪律，事件回调用闭包快照照常安全（数据在 state，每次 render 喂新闭包）。

### 坑（使用前必读）

1. **对 Map 无效**：fast-deep-equal 对 Map 整体恒判等（无 Map 分支），Map 里塞 alias 对象也救不了——Map 形态只能走方案一；
2. **aliasArray 只处理首项**：空数组 / 首项非对象时退化回深比较；改动仅落在尾部时仍靠首项时间戳判不等（正确但要知道机制在首项）；
3. **`Date.now()` 毫秒精度碰撞**：同一毫秒内的两次更新，两个 aliasObject 的时间戳相同 → valueOf 相等 → **直接 return true 判等 → 漏更新**（源码该分支相等即 return，不再继续比属性）。同一 handler 里连续两次更新极易踩中，稳妥可换 `performance.now()`（精度更高，但也非零风险）；
4. **valueOf 语义污染**：所有隐式转换场景（`obj + 1`、大小比较、部分库的 == 判断）会拿到时间戳数字，排查诡异值时先想到这里。

## 四、方案选择决策

```
数据形态是 Map？
├─ 是 → ref + stamp（唯一解：恒判等漏更新）
└─ 否（数组/对象）
   ├─ 有命令式消费者（图表库冻结回调 / 事件回调要最新值）→ ref + stamp
   ├─ 纯响应式消费 + 数据量大（比较已是交互瓶颈）→ valueOf 别名（aliasData）
   └─ 数据量小（比较 < 0.1ms 量级）→ 直接 state，不引入任何方案
```

通用原则：

- **先砍订阅点再谈数据方案**——小粒度组件（task / 单元格级）不直接 useModel，订阅收敛到容器组件后 props 下传；高频小状态与大数据分 model，避免小更新扇出大数据订阅者；
- **useModel 一律带 selector（通用守则，与数据大小无关）**：核心收益是无关更新不重渲染；字段选择上，单字段直接返回原始引用（`===` 短路零成本），多字段包对象字面量；selector 返回值里不要出现大数据（那是深比较的火药桶，见机制层三档成本）；
- 先测后优——selector 返回大数组且交互发粘时再上方案，小数据的深比较是噪声；
- 无论哪个方案，**selector 返回 `{ 稳定引用, 版本值 }` 这种小对象**是通用形态——让 fast-deep-equal 只比 2 个 key + 1 个数字；
- 升级 fast-deep-equal 前先重验 Map 行为：官方 master 有 Map/Set 真比较分支，若随未来版本发布，Map 恒判等会变成 O(N×M) 真比较，两个方案的收益记账都要重算。

## 五、验证方法沉淀（确认运行时行为，别信记忆）

1. 找运行时 import：`grep -n 'fast-deep-equal' src/.umi/plugin-model/index.tsx`——看 `require.resolve` 落到哪个文件；
2. 钉死产物真伪：`curl` registry tarball 解开与本地 `diff`，一致则行为=官方该版本行为；
3. 读判等分支时注意：`es6/index.js` 与 `index.js` 内容可能不同（es6 有 Map 分支），**以运行时 import 的那份为准**；
4. 行为验证：两个内容不同的 Map 经 `isEqual` 判等（`console.log(isEqual(mapA, mapB)) === true`）即命中恒判等。
