# Date 踩坑清单

一句话定性：`new Date` 是**两条完全不同的解析通道**——数字入参走「分量构造」，行为完全由规范定义，唯独 0-99 年会被隐加 1900；字符串入参走「格式解析」，规范只认 ISO 8601 的 `T` 分隔，其余全是引擎方言。两个坑各占一条通道：数字通道把年 4 悄悄变成 1904；字符串通道的 `YYYY-MM-DD HH:mm:ss` 在 Chrome 正常、Safari 直接 `Invalid Date`。

坑一坑二均已实测（Node 26 / V8；Safari 行为为 WebKit 社区公知，Windows 下无法本机复现，结论与 MDN 及 Can I use 一致）。

## 一、坑 1：数字入参 0-99 年隐加 1900——`new Date(4, 1, 29)` 回读成 1904

**机制**：ECMAScript 规定，`new Date(year, month, ...)` 的 year 落在 `0 ≤ year ≤ 99` 时，构造内部按 `1900 + year` 处理（保留自 1900 年代两位年表示的兼容行为）。不抛错、不警告，静默改写。

实测（V8）：

```js
new Date(4, 1, 29)              // => Mon Feb 29 1904  —— 年 4 被改成 1904
new Date(4, 1, 29).getFullYear() // => 1904，不等于 4 ★
new Date(0, 1, 29)              // => Thu Mar 01 1900  —— 年 0 → 1900（平年），
                                //    2 月 29 日再滚成 3 月 1 日，年月日三连错 ★
```

**伤害场景：回读法校验日期合法性。**「构造一个 Date 再回读三字段比对」是校验 `2023-02-30` 这类假日期的常用手法，但年一到 0000-0099 区间就误判——回读的永远是 19xx 年：

```js
// '0004-02-29' 按儒略推格里历是真实日期（4 年是闰年），却被判非法
function isValid(y, m, d) {
  const dt = new Date(y, m - 1, d);
  return dt.getFullYear() === y && dt.getMonth() === m - 1 && dt.getDate() === d;
}
isValid(4, 2, 29)    // => false，误杀 ★
isValid(2024, 2, 29) // => true，正常区间无恙
```

典型的「测试全绿、上线才炸」：业务数据年份几乎都在 1970 之后，单测覆盖不到 0-99 年，一旦遇到（流水号、补零的年份字段、历史/天文数据）就是静默丢数据。

**三个安全出口**（想保留回读法时任选其一）：

```js
// 1. setFullYear 修法落地：载体用现代日期，年月日三分量一起设
function isValid(y, m, d) {
  const dt = new Date(2024, 0, 1); // 载体取任一 1000 年后的正常日期，构造时不踩坑；
  dt.setFullYear(y, m - 1, d);     // 年按字面处理，月日滚动照常
  return dt.getFullYear() === y && dt.getMonth() === m - 1 && dt.getDate() === d;
}
isValid(4, 2, 29)    // => true，坑 1 误杀的日期过检
isValid(0, 2, 29)    // => true，年 0 是闰年，2 月 29 日真实存在
isValid(2023, 2, 30) // => false，假日期滚动检测照常生效

// 2. ISO 字符串通道同样不隐加 1900（注意 T 分隔，见坑 2）
new Date('0004-02-29T00:00:00').getFullYear() // => 4

// 3. 年份已知 ≥ 1000 时，数字构造回读天然安全
```

最稳的还是**不借 Date、手算日历**：正则保结构，大小月查表 + 闰年判式保月日联合规则，全量 0000-9999 年无一例外：

```js
// 校验 YYYY-MM-DD / YYYY/MM/DD 是否为真实日历日期
// 结构：年 4 位，月 01-12，日 01-31，反向引用 \2 锁定前后分隔符一致
const REGEXP = /^(\d{4})([-/])(0[1-9]|1[0-2])\2(0[1-9]|[12]\d|3[01])$/;
// 平年各月天数【下标即月份 - 1，闰年 2 月另加】
const MONTH_DAYS = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];

function isValidDate(value) {
  if (typeof value !== 'string') return false;
  const match = REGEXP.exec(value);
  if (!match) return false; 
  // 正则只保结构，月日联合规则另验
  const year = +match[1], month = +match[3], day = +match[4];
  // 闰年：四年一闰、百年不闰、四百年再闰
  const leap = (year % 4 === 0 && year % 100 !== 0) || year % 400 === 0;
  // 该月天数上限：闰年 2 月从 28 提到 29，其余查表
  const maxDay = month === 2 && leap ? 29 : MONTH_DAYS[month - 1];
  return day <= maxDay; // 日不超该月上限，即日历上真实存在
}

isValidDate('0004-02-29')  // => true，坑 1 误杀的日期在这里过检
isValidDate('2023-02-29')  // => false，平年无 2 月 29 日
isValidDate('2024-04-31')  // => false，4 月只有 30 天
isValidDate('2024-02/29')  // => false，分隔符前后不一致
```

## 二、坑 2：Safari 不认 `new Date('YYYY-MM-DD HH:mm:ss')`——直接 Invalid Date

**机制**：规范只标准化了 ISO 8601 子集，日期与时间的分隔符是字母 `T`。空格分隔 + 连字符（`2024-02-29 10:30:00`）**不在规范内**，属引擎自行扩展的方言——V8（Chrome/Edge/Node）和 SpiderMonkey（Firefox）宽容地收下，JavaScriptCore（Safari）严格拒收，返回 `Invalid Date`。

兼容矩阵：

| 写法 | Chrome / Node | Firefox | Safari |
| --- | --- | --- | --- |
| `'2024-02-29T10:30:00'`（ISO，T 分隔） | ✅ | ✅ | ✅ |
| `'2024-02-29 10:30:00'`（空格分隔）★ | ✅ | ✅ | **Invalid Date** |
| `'2024/02/29 10:30:00'`（斜杠 + 空格） | ✅ | ✅ | ✅ |

★ 行就是坑点：后端最常下发的就是这格式，开发全程 Chrome 验证无异常，Safari 用户一进页面就是 NaN 时间戳、空白日期。

**修法**（按推荐顺序）：

```js
// 1. 统一转 ISO 的 T 分隔——唯一走规范的做法，全引擎一致
s.replace(' ', 'T')            // '2024-02-29T10:30:00'

// 2. 连字符全换斜杠——legacy 格式，事实标准，各引擎都收
s.replace(/-/g, '/')           // '2024/02/29 10:30:00'

// 3. 拆字段走数字构造——行为完全由规范定义
const [y, m, d, h = 0, mi = 0, sec = 0] = s.split(/[-: ]/).map(Number);
new Date(y, m - 1, d, h, mi, sec)   // 注意月份 0 起算；年 < 100 回到坑 1
```

**兜底**：任何解析结果使用前必过 `Number.isNaN(d.getTime())`，别让 `Invalid Date` 流进计算与渲染（`Date` 的算术和字符串化会产出 `NaN` / `'Invalid Date'` 上屏）。

**最终方案**：修法 1 + 兜底，收敛成一个边界归一函数——字符串只从这一个口进 `new Date`，别处不裸调：

```js
// 统一入口：把 'YYYY-MM-DD HH:mm:ss' 归一成 ISO 的 T 分隔再构造
function parseDateTime(s) {
  const d = new Date(s.trim().replace(' ', 'T'));
  // Invalid Date 的 getTime() 为 NaN，归 null 截住，不流进计算与渲染
  return Number.isNaN(d.getTime()) ? null : d;
}

parseDateTime('2024-02-29 10:30:00')  // => 全引擎一致，本地时间 2024-02-29 10:30
parseDateTime('bad')                  // => null
```

选修法 1、弃 2 和 3 的理由：`T` 分隔是规范内格式，全引擎行为一致；空格方言与无偏移 ISO 串都按**本地时间**解析，归一只是换壳不动时区（斜杠方案改纯日期串会从 UTC 挪到本地，见附带第 3 条）；拆字段代码最多，还把月份 0 起算、年 < 100 两个坑重新请回门里。治本在后端直接下发 ISO（`T` 分隔）或时间戳，前端归一函数是防御层。

## 三、附带：同一通道的三个相邻坑

坑 1 坑 2 的「邻居」，同源同管道，顺手记下：

```js
new Date(2024, 1, 29)   // => 2 月 29 日——数字入参月份 0 起算，1 是二月 ★
new Date(2023, 1, 30)   // => 3 月 2 日——超上限不报错，滚入下月 ★
new Date('2024-02-29')  // => 按 UTC 零点解析——纯日期串走 UTC，
                        //    带 T 时间串才走本地时区，东八区两者可差一天 ★
```

第三条对坑 2 的修法选型有直接影响：`'2024-02-29'` 换成 `'2024/02/29'` 后从 UTC 变本地零点，跨时区场景两者不等价，改格式前先确认时区语义。

## 四、速查守则

1. 字符串解析只信两种写法：ISO `T` 分隔（`2024-02-29T10:30:00`），或先拆字段再走数字构造；`YYYY-MM-DD HH:mm:ss` 必须转换后再喂给 `new Date`
2. 数字构造的年份 < 100 会被隐加 1900：要么先正则把年份锁在 1000 以上，要么 `setFullYear`，要么手算日历
3. 回读三字段比对是合法的校验手法，但前提是年份区间已证明安全——两个手法必须配套出现
4. 解析结果一律 `Number.isNaN(d.getTime())` 兜底，`Invalid Date` 不过这道闸不下发
5. 数字构造月份 0 起算；日超上限静默滚入下月，这两条是回读法能生效的原理，也是单独使用时的坑
6. 纯日期串按 UTC、带时间串按本地时区解析，改写格式前先想清楚时区语义
