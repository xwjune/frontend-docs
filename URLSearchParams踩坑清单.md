# URLSearchParams 踩坑清单

一句话定性：`URLSearchParams` 实现的是 HTML 表单的 `application/x-www-form-urlencoded` 序列化**方言**，不是 URL 查询串的 RFC 3986 编码——安全字符集窄得多（`,` `/` `:` `~` `!` 这类查询串里合法裸写的字符统统编成 `%XX`）、空格写成 `+`、读写还不对称（读时把 `+` 还原成空格）。拿它"读取 → 改一个参数 → 回写"整个查询串，等于把别人写好的 URL 按表单方言重新排版；拿它从零构造查询串，空格会写成 `+`，而按 RFC 3986 解码的对接方只认 `%20`。

各坑均来自真实事故场景（SSO 单点登录回跳、OAuth 授权跳转、页面地址改写），关键行为均已实测（Node 26 / V8）。

## 一、坑 1：动一个参数，整串重排 —— `ids=a,b` 变 `a%2Cb`

**机制**：WHATWG 规范给 `URLSearchParams` 的序列化安全集只有**字母数字和 `* - . _`**，空格写成 `+`，其余一律 `%XX`。而 RFC 3986 查询串里 `,` `/` `:` `@` `?` `!` `'` `(` `)` `~` 本可裸写。致命的是 `url.searchParams.set()/delete()` 一旦触发，**整个**查询串按方言重排，原有参数无一幸免：

```js
const u = new URL('http://x/p?ids=a,b&tag=u~1');
u.searchParams.set('extra', 'v');
u.href // => http://x/p?ids=a%2Cb&tag=u%7E1&extra=v
       //    只想加一个参数，a,b → a%2Cb、u~1 → u%7E1 全被改写 ★
```

**只删不改同样中招**——`delete` 本身没碰其它参数，但内部不保存原始字符串（存的是解码后的键值对），任何改动类操作（`set`/`delete`/`append`/`sort`）都会让查询串待重序列化，此后取整串时剩余参数**全部**过一遍方言编码器。判定条件不是操作种类，是有没有原串、动没动它：

```js
const u = new URL('http://x/?token=t&ids=1,2&name=a%20b&tag=u~1');
u.searchParams.delete('token');
u.search // => ?ids=1%2C2&name=a+b&tag=u%7E1  只删了 token，其余三个全被改写 ★
// 对照：纯读取（get/has，不做任何改动）不触发，url.search 保持原串
```

**伤害场景（真实事故）**：SSO 登录回跳链路。

1. 登出跳身份平台前，把当前页查询串**原样**缓存（如 sessionStorage），约定"回跳后从缓存找回页面参数"；
2. 身份平台回跳带 token 后，旧实现用 `new URL(href)` + `searchParams` 增删参数再 `history.replace` 拼回地址 —— 整串重排，`ids=a,b` 变 `a%2Cb`；
3. `%2C` 解码后与 `,` 语义等价，**为什么仍然致害**：这类链路里常见的自写解析器直接 `split('&')`/`split('=')` 取裸串、**不做 decode**，下游拿到的是 `'a%2Cb'` —— 任何 `split(',')`、字符串比对、再编码转发从此全错：

```js
// 不 decode 的自写解析器实测
const search = '?ids=a%2Cb';
search.substring(1).split('&')[0].split('=')[1] // => 'a%2Cb'，期望 'a,b' ★
```

**善后结论**：不整串重序列化——缓存的查询串原样透传，只对新增参数 `encodeURIComponent` 后手动拼接：

```js
let newSearch = pageQuery; // 缓存里取出的原始查询串，原样
newSearch += `${newSearch.includes('?') ? '&' : '?'}key=${encodeURIComponent(value)}`;
history.replace(pathname + newSearch);
```

**同款坑的常见残留**：改写当前页地址的代码——`new URL(window.location.href)` + `searchParams.set` + 回写 `url.href`（典型如"切换某个上下文 id 后把新值写回地址栏"），完整触发整串重排。当前页 query 里若有裸写的 `, ~ / :` 等字符，跳转后地址栏被悄悄改写。

**安全出口**：

```js
// 需要"保留原查询串 + 增删个别参数"：把查询串当不透明字符串，手动追加
let newSearch = pageQuery; // 原样
newSearch += `${newSearch.includes('?') ? '&' : '?'}key=${encodeURIComponent(value)}`;

// 直接追加的暗坑：key 已存在时重复（'?tab=old&tab=new'），且读取语义分裂——先者胜的读法
// （searchParams.get 与多数自写解析器）读回旧值、后者胜的读新值。须先剔同 key 旧值再追加

// 删参数同理：按 & 拆成 key=value 对，剔掉目标 key 的对再拼回（其余字符原样不动）
// '?a=1&token=t&b=2'.substring(1).split('&').filter((p) => !p.startsWith('token=')).join('&') // => 'a=1&b=2'
```

若下游**全部**用 `searchParams.get()` 读（会 decode），`%2C` 与 `,` 等价、重排无害——但"全部"这个前提在跨系统回跳 URL 上无法保证，裸串解析器（自写取参函数、后端日志比对、监控埋点）防不胜防。

## 二、坑 2：空格 → `+`，OAuth scope 的互操作地雷

**来源场景**：构造 SSO/OAuth 授权跳转 URL——`URLSearchParams` 会把空格序列化成 `+`，部分身份平台只认 `%20`，修复为**只对查询段替换**：

```ts
const url = new URL(`${issuer}/authorize`);
url.searchParams.set('client_id', clientId);
url.searchParams.set('response_type', 'code');
url.searchParams.set('scope', 'openid profile'); // scope 含空格 → 序列化为 openid+profile
url.searchParams.set('redirect_uri', serviceUrl);
// 只对查询段把 + 替换为 %20，绝不动整条 URL——整串 replaceAll 会误伤 path 里合法的 +。
// 段内替换无损：searchParams 产出的查询段里，字面 + 只可能来自空格（值里的 + 早已被编成 %2B）
url.search = url.search.replaceAll('+', '%20');
window.location.href = url.href; // => ...scope=openid%20profile... ✓
```

**机制**：RFC 6749 规定授权端点参数确实按 form-urlencoded 序列化，`+` 写法规范合法；但不少身份网关/老 CAS 按 RFC 3986 解码（只把 `%20` 当空格，`+` 当字面加号），`openid+profile` 被当成一个非法 scope → 授权失败。

**安全出口**：
- 保留 `URL`/`searchParams` 构造：即上例——只替换 `url.search` 段，绝不 `replaceAll` 整条 URL；
- 更彻底（推荐）：从零手拼 `encodeURIComponent`（封装形态见参考实现 `buildQuery`）——空格恒为 `%20`，不经过方言序列化器，无需任何兜底：

```ts
const query = [
  ['client_id', clientId],
  ['response_type', 'code'],
  ['scope', 'openid profile'],
  ['redirect_uri', serviceUrl],
]
  .map(([key, value]) => `${key}=${encodeURIComponent(String(value))}`)
  .join('&');
window.location.href = `${issuer}/authorize?${query}`; // => ...scope=openid%20profile... ✓
```

编码后的值里 `!'()*~` 仍为裸字符——它们在查询串值里合法，各身份平台均可解。

## 三、坑 3：把已编码的值喂给 set() —— 双重编码

`set()` 只吃"人能读的明文值"，`%` 一律视为数据：

```js
const u = new URL('http://x/');
u.searchParams.set('r', 'http://y/?a=b%20c');
u.href // => http://x/?r=http%3A%2F%2Fy%2F%3Fa%3Db%2520c  %20 → %2520 ★
```

凡是上游给的已经是编码后的字符串（回调 URL、referer、后端拼好的跳转地址），要么先 `decodeURIComponent` 再 set，要么手拼——即坑 2 的写法与参考实现 `buildQuery`。

## 四、坑 4：读写不对称 —— get() 把 `+` 当空格，decodeURIComponent 又不认 `+`

同一个值三种写法 × 三种读法（实测）：

| 值的写法 | `searchParams.get` | `decodeURIComponent` | 裸串 `split('=')` |
|---|---|---|---|
| `b+c`（form 方言的空格） | `'b c'` ✓ | `'b+c'` ★不还原空格 | `'b+c'` |
| `b%20c`（RFC 方言的空格） | `'b c'` ✓ | `'b c'` ✓ | `'b%20c'` ★没解码 |
| `b%2Bc`（编码后的字面加号） | `'b+c'` ✓ | `'b+c'` ✓ | `'b%2Bc'` |

- `searchParams.get()` 是唯一同时理解两种方言的读法，读别人序列化的 URL（SSO 回跳的 `token`/`userInfo`）优先用它；
- 它唯一无法自救的情况：生产方把**字面加号**裸写在 URL 里（RFC 3986 合法）。`+` 被读成空格且不可恢复——标准 base64（含 `+`）、电话号码 `+86` 一类参数，必须要求生产方编码为 `%2B`，否则凭空丢字符；
- 不 decode 的自写裸串解析器两头都不解，只能消费"从头到尾没人动过"的原生 URL——这正是坑 1 里重排致害的根因。

## 附：三种"编码器"对照（实测汇总）

| | 字母数字 | 空格 | `,` `/` `:` `@` `?` | `!` `'` `(` `)` `~` | `*` `-` `.` `_` | `+` | `%` |
|---|---|---|---|---|---|---|---|
| `URLSearchParams`（form-urlencoded） | 裸 | **`+`** | 全部 `%XX` | 除 `*` 外全部 `%XX` | 裸 | `%2B` | `%25` |
| `encodeURIComponent` | 裸 | `%20` | 全部 `%XX` | **全部裸** | 裸 | `%2B` | `%25` |
| RFC 3986 查询串允许裸写 | 裸 | （无裸写概念） | 裸 | 裸 | 裸 | 裸 | 须编码 |

## 落地约定

1. **保留原始查询串**的场景（SSO 回跳拼 URL、改写当前页地址）：查询串当不透明字符串，手动 `encodeURIComponent` 追加 / `split('&')` 过滤，禁止 `searchParams` 改动（`set`/`delete` 皆算）后回写整串（参考实现见下节）；
2. **构造发往外部系统**（OAuth/身份平台）的 URL：优先从零手拼 `encodeURIComponent`（空格成 `%20`，封装形态见参考实现 `buildQuery`）；沿用 `URLSearchParams` 构造时，只对 `url.search` 段替换 `+`，绝不 replaceAll 整条 URL；
3. **读 query**：统一 `searchParams.get()`（兼容 `%20`/`+` 两种空格写法）；不 decode 的自写解析器仅供历史兼容，新代码不新增使用方；
4. `set()` 只喂明文值；已编码的值先 decode 再进。

## 参考实现：patchQuery

约定 1 的落地形态——以"保留原串"的方式增删查询串参数的纯函数，零依赖，可直接拷贝到任何项目。选型分界是有没有需要保留的原串：有（改写当前地址、透传缓存查询串）→ 本函数；没有（从零构造整条查询串）→ 从零裸拼，见下文 buildQuery。

```ts
// 查询串值的编码：encodeURIComponent 保结构安全（空格成 %20 而非表单方言的 +），再还原 , : @ /
// （撤销 %2C %3A %40 %2F）——这四个在查询串里本可裸写，还原无损（裸写与 %XX 对解码器等价）；
// 其余不还原：+ 会被表单方言读成空格，& = ; 是裸串解析（split('&')/split('=')）的分界符
const QUERY_BARE_SAFE: Record<string, string> = { '%2C': ',', '%3A': ':', '%40': '@', '%2F': '/' };
const encodeQueryValue = (value: string | number) =>
  encodeURIComponent(value).replace(/%2C|%3A|%40|%2F/g, (code) => QUERY_BARE_SAFE[code]);

/**
 * 以"保留原串"的方式增删查询串参数：未涉及的参数原样保留，新增/替换的值走结构安全编码
 * （encodeURIComponent 后还原查询串可裸写的 , : @ /，逗号等分隔符保持裸写可读），
 * 值传 null/undefined 等效删除。返回新的 search（含前导 ?，无参数时为空串）。
 *
 * 不要用 URLSearchParams 做这件事——它按表单方言把整个查询串重序列化：已有参数值里的逗号等
 * 合法裸写字符会被编码（a,b → a%2Cb），空格会编成 +。下游凡是按裸串取值、不做 decode 的解析链都会整体受影响。
 *
 * 契约边界：
 * - 入参是 search 片段（前导 ? 可省略），不含 # hash；
 * - key 须由安全字符构成（不含 & = # ? 及空白），不做编码/解码，按裸串与原串匹配；
 * - 未涉及的参数逐字节保留（含重复参数）：其中的裸 + 原样保留（表单方言下游会读作空格），
 *   新值的 + 则一律编码为 %2B；
 * - 原串的空段与末尾 & 会被规范化掉。
 *
 * @example
 * patchQuery('?a=1&b=2', { deleteKeys: ['b'] }) // => '?a=1'
 * patchQuery('?a=1', { set: { c: 'x,y' } }) // => '?a=1&c=x,y'
 * patchQuery('?a=1&b=2', { set: { b: null } }) // => '?a=1'（null/undefined 等效删除）
 * patchQuery('?a=x+y', { set: { b: 'p+q' } }) // => '?a=x+y&b=p%2Bq'（原串裸 + 保留，新值 + 编码）
 * patchQuery('?a=1&&b=2&', { set: { c: 3 } }) // => '?a=1&b=2&c=3'（空段与末尾 & 被规范化）
 */
export function patchQuery(
  search: string,
  ops: { set?: Record<string, string | number | null | undefined>; deleteKeys?: string[] },
) {
  // set 的 key 也进 removed：替换 = 先剔旧值再追加新值（否则重复参数，且解析"先者胜"会读回旧值）；
  // 值为 null/undefined 时剔了不追加，净效果即删除
  const removed = new Set([...(ops.deleteKeys ?? []), ...Object.keys(ops.set ?? {})]);
  const kept = search
    .replace(/^\?/, '')
    .split('&')
    .filter(Boolean)
    .filter((pair) => !removed.has(pair.split('=')[0]));
  Object.entries(ops.set ?? {}).forEach(([key, value]) => {
    if (value != null) {
      kept.push(`${key}=${encodeQueryValue(value)}`);
    }
  });
  return kept.length ? `?${kept.join('&')}` : '';
}
```

典型用法：

```ts
// 改写当前地址：只动这几个参数，其余查询串逐字保留
window.open(origin + pathname + patchQuery(search, { set: { id } }));
history.replace(path + patchQuery(search, { deleteKeys: ['token'] }));
```

设计要点（与正文各坑的对应）：

- 未涉及的参数**逐字透传**，不经任何编码器——坑 1 的原串保留；
- 新值走结构安全编码——`encodeURIComponent` 后还原查询串可裸写的 `, : @ /`：空格成 `%20` 不引入坑 2 的 `+` 方言，`ids=1,2` 这类逗号分隔的列表值保持裸逗号可读（`split(',')` 等裸串消费不失效）；`&` `=` `#` `%` 仍编码，`+` 不还原（表单方言会读成空格）；
- 还原集不是按项目挑的，是三条项目无关规则的交集：RFC 3986 查询语法允许裸写 ∧ 两种方言解码器（`searchParams.get` / `decodeURIComponent`）原样通过 ∧ 不是任何常见解析器的分界符（`+` 死于第二条，`&` `=` `;` 死于第三条）。且还原无损——裸写与 `%XX` 对一切标准解码器结果相同，还原不改变正确读者的行为，只是把不 decode 的裸串读者从读错修为读对；
- `set` 先剔旧值再追加新值——替换语义，绝不产生"先者胜"的重复参数；
- 值传 null/undefined 走"剔旧值但不追加"路径——一组代码同时实现替换与删除。

还原集的推导——决策空间可枚举尽：`encodeURIComponent` 会编码的字符里，RFC 3986 查询语法允许裸写的只有 9 个（`$ & + , ; : @ / ?`），过三道过滤器再按"真实参数值高频出现、有裸写传统"取舍：

```
$ & + , ; : @ / ?      9 个候选
 ├─ 过滤② 方言中立 → 淘汰 +（表单方言把裸 + 读成空格）
 ├─ 过滤③ 非分界符 → 淘汰 & 和 ;（& 是 split('&') 的分界；; 做过历史参数分隔符）
 └─ 幸存 $ , : @ / ?
     ├─ 判断入选 , : @ / —— 真实参数值高频出现、全世界手写查询串都习惯裸写
     └─ 落选 $（几乎不出现在值里，无还原收益）、?（易与查询串起始混淆，可读性差）
```

四个入选字符各自的正向场景：

| 字符 | 真实值场景 | 裸写传统 |
|---|---|---|
| `,` | `ids=1,2` CSV 列表参数 | 事实上的列表分隔符，且下游常按 `split(',')` 裸串消费，还原刚需最强 |
| `:` | `t=09:30`、时长 `HH:mm:ss` | 时间值从不写成 `%3A` |
| `/` | `redirect=https://x/y`、日期 `2024/01/01` | URL 型值、斜杠日期都裸写 |
| `@` | `u=x@y.com` | 邮箱从不写成 `%40` |

反向闭合：`&` `=` `#` `%` 空格 `+` 这些必须编码的字符全部死在过滤器上，不进入判断环节——还原集不可能混入任何危险字符。

## 参考实现：buildQuery —— 从零构造的裸拼形态

约定 2 的落地形态，与 patchQuery 成对互补。**判定条件先给：有没有需要保留的原串**——没有原串（不基于任何已有查询串做增删，直接拼一条新的）→ 本函数；有原串 → patchQuery，绝不在两个实现之间"把原串拆开重拼"。

```ts
// 查询串值的编码：encodeURIComponent 保结构安全（空格成 %20 而非表单方言的 +），再还原 , : @ /
// （撤销 %2C %3A %40 %2F）——这四个在查询串里本可裸写，还原无损（裸写与 %XX 对解码器等价）；
// 其余不还原：+ 会被表单方言读成空格，& = ; 是裸串解析（split('&')/split('=')）的分界符
const QUERY_BARE_SAFE: Record<string, string> = { '%2C': ',', '%3A': ':', '%40': '@', '%2F': '/' };
const encodeQueryValue = (value: string | number) =>
  encodeURIComponent(value).replace(/%2C|%3A|%40|%2F/g, (code) => QUERY_BARE_SAFE[code]);

/**
 * 从零构造查询串（无前导 ?，无参数时为空串）：逐对手拼，值走结构安全编码（encodeURIComponent
 * 后还原可裸写的 , : @ /），空格恒为 %20，逗号等分隔符保持裸写可读。适合发往 SSO/OAuth/身份平台
 * 等按 RFC 3986 解码、只认 %20 空格的外部系统。
 *
 * @example
 * buildQuery([['a', 1], ['b', 'x y']]) // => 'a=1&b=x%20y'
 * buildQuery([['ids', 'x,y'], ['scope', 'openid profile']]) // => 'ids=x,y&scope=openid%20profile'
 */
export function buildQuery(pairs: Array<[string, string | number]>) {
  return pairs.map(([key, value]) => `${key}=${encodeQueryValue(value)}`).join('&');
}
```

使用场景：

- **发往外部系统的跳转 URL（硬需求）**：SSO/OAuth 授权、注销、第三方回调——对方可能按 RFC 3986 解码、空格只认 `%20`（坑 2），手拼天然满足，不需要任何 `+` 兜底；
- **纯内部跳转从零构造（统一约定）**：技术上 `URLSearchParams` 也能用（见下方边界矩阵 ⚠️ 档），统一走本函数只为编码可预测（空格恒 `%20`）与团队单一规则；
- **值是逗号列表（`ids=1,2` 这类）**：默认即安全——值编码器与 patchQuery 同一，逗号保持裸写，`split(',')` 等裸串消费不失效；发外部系统同样成立（裸写与 `%XX` 对任何解码器读回等价）。

设计要点：

- 逐对编码后 `join('&')`，不经过任何方言序列化器——值编码器与 patchQuery 同一，两个参考实现产出同一种方言（空格 `%20`、`, : @ /` 裸写）；
- 参数用数组对而非对象：顺序即输出顺序，且支持重复 key（`a=1&a=2` 这类 API 语义对象表达不了）。

## 结论

一句话：**`URLSearchParams` 是表单（form-urlencoded）方言的序列化器，不是 URL 查询串的编辑器——"读"可以用它，"从零构造"可以用它，唯独"改别人的 URL"不能用它。** 读别人的 URL 用 `searchParams.get()`，改别人的 URL 用裸串拼接 + `encodeURIComponent`，喂 `set()` 只喂明文。

展开为使用边界矩阵，三档强度：

| 场景 | URLSearchParams | 强度 |
|---|---|---|
| **读取参数**（`get` / `getAll` / `entries`） | 推荐——唯一能同时解 `%20` 和 `+` 两种方言的读法。例外：对方裸写字面 `+`（RFC 3986 合法）会被读成空格且不可恢复，此类参数须生产方编码为 `%2B` | ✅ 首选 |
| **改写已有查询串**（有原串：改当前地址、透传缓存串；`set`/`delete`/`append`/`sort` 任一改动后取整串都算） | 禁止——坑 1 整串重排 | ❌ 硬性，无例外 → `patchQuery` |
| **从零构造 → 发外部系统**（SSO/身份平台） | 不建议——坑 2：空格编成 `+`，对方可能只认 `%20` | ❌ 硬性风险 → `buildQuery` 手拼 |
| **从零构造 → 纯内部跳转** | 可以用，不算错——坑 1 不存在，`+` 自家 `get()` 能解回；统一手拼只为编码可预测、团队单一规则 | ⚠️ 软性约定 |

四条约定背后是同一个认知：这套 API 自带编码方言，凡从它嘴里出来的字符串都带方言烙印——要么整个查询串都由它序列化（从零构造），要么完全不经过它（保原串 / 手拼）。**唯一不能做的是两种方言混着来**，四个坑全是这么踩出来的：

| 坑 | 混法 |
|---|---|
| 坑 1 | 方言串（`%2C`）喂给不 decode 的裸串消费者 |
| 坑 2 | 方言串（`+`）流向只认 RFC 方言（`%20`）的外部系统 |
| 坑 3 | 已按 RFC 方言编码的值喂给方言序列化器 `set()` |
| 坑 4 | 用单一方言语义的读法去解两种方言写入的 `+` |
