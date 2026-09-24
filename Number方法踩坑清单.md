# Number() 踩坑清单

一句话定性：`Number()` 是**类型转换**，不是**校验**。它的接受面远比「数字」宽——布尔、数组、Date、空白串、十六进制字面量都能转出值，转换失败还有返回 `NaN`、返回 `Infinity`、直接抛错三种下场。凡是拿它当校验器用的地方，都是在裸奔。

以下所有例子均实测（Node 26 / 现代浏览器一致）。

## 一、非字符串类型的隐式转换

| 输入 | 结果 | 说明 |
| --- | --- | --- |
| `Number(true)` | `1` | 布尔可转 |
| `Number([])` | `0` | 空数组转空串再转 0 ★ |
| `Number([20])` | `20` | 单元素数组先 `String()` 再转 ★类数组当数字 |
| `Number([''])` | `0` | 同上链条 |
| `Number(new Date())` | 时间戳 | Date 转毫秒数 |
| `Number(null)` | `0` | ★null 变 0，和 undefined 行为不一致 |
| `Number(undefined)` | `NaN` | |
| `Number({})` | `NaN` | 普通对象转不了 |
| `Number(10n)` | `10` | BigInt 可转，但见第四节 |
| `Number(Symbol())` | **抛 TypeError** | ★校验函数自己炸，连 false 都返回不了 |

★ 号是最容易中招的：`[20]` 进校验器会被洗白成合法数字 `20`，调用方的 bug 被静默吞掉；`null` 变 `0` 则把「没传」伪装成「传了零」。

## 二、字符串解析规则比想象中宽

| 输入 | 结果 | 说明 |
| --- | --- | --- |
| `Number('')` | `0` | ★最大坑，空串等于 0 |
| `Number('  ')` / `'\t'` / `'\n'` | `0` | 纯空白串先 trim 再转 |
| `Number('0x10')` | `16` | 十六进制字面量 |
| `Number('0b11')` | `3` | 二进制字面量 |
| `Number('0o17')` | `15` | 八进制字面量 |
| `Number('020')` | `20` | 前导零照转 |
| `Number('1e3')` | `1000` | 科学计数法 |
| `Number('+5')` | `5` | 前导正号 |
| `Number('Infinity')` | `Infinity` | 字面量直接过 |
| `Number('12x')` / `Number('1,000')` | `NaN` | 混入其他字符才是 NaN |

注意：`parseInt` 规则又不一样（`parseInt('12x')` 得 `12`，且不认 `0x` 之外的进制前缀写法差异），别混用两套心智模型。

## 三、溢出不报错

`Number('1e999') => Infinity`——超双精度上限静默变 `Infinity`，不抛错。若后续没有 `Number.isFinite` 拦截，`Infinity` 会一路流进计算与渲染管线（上屏 `'Infinity'`）。同理 `Number('1e-999') => 0`，下溢静默归零。

## 四、精度两大坑

**1. 浮点运算误差（二进制近似值）**

```js
0.1 + 0.2    // => 0.30000000000000004
0.1 * 100    // => 10.000000000000002
11.2 / 100   // => 0.11199999999999999
```

乘除放大后误差暴露。金额场景必须走字符串逐位运算或整数（分）为单位。

**2. 安全整数边界 2^53（Number.MAX_SAFE_INTEGER）**

```js
Number('9007199254740993')  // => 9007199254740992，装载那一刻就丢了末位
Number(2n ** 70n)           // => 1.1805916207174113e+21，BigInt 超长同样静默丢精度
```

超过 2^53 的整数字面量/字符串，`Number()` 装不下就舍入，**没有任何警告**。大数用 BigInt 或字符串管线。

**3. 附带：toFixed 的两个坑**

```js
(1.005).toFixed(2)  // => '1.00'（期望 '1.01'）——它对二进制近似值（实为 1.00499…）舍入
(1e21).toFixed(2)   // => '1e+21'——值 >= 1e21 时无视小数位参数，直接返回科学计数法串
```

## 五、校验函数的正确写法

反例（jQuery 老 `$.isNumeric` 同款思路）：

```js
function isNumber(value) {
  if (value === undefined || value === null || value === '') return false;
  return !Number.isNaN(Number(value));
}
```

它放行 `true`、`[]`、`[20]`、`new Date()`、`' '`、`'0x10'`、`'020'`、`Infinity`、`'1e999'`，遇到 `Symbol()` 还直接抛错——黑名单挡不住 `Number()` 宽转换，列举法必漏。

正解是**三层白名单**（类型 → 字面量形态 → 有限性）：

```js
function isNumber(value) {
  if (typeof value === 'number') {
    return Number.isFinite(value); // 挡 NaN / Infinity
  }
  if (typeof value === 'string'
    && /^[+-]?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?$/.test(value) // 仅十进制与科学计数法
    && Number.isFinite(Number(value))) { // 挡 '1e999' => Infinity
    return true;
  }
  return false;
}
```

## 六、判 NaN 别用全局 isNaN

```js
isNaN('x')         // => true，全局 isNaN 先做 Number() 转换再判
Number.isNaN('x')  // => false，不转换，非 number 一律 false
Number.isFinite(x) // 一步到位挡 NaN 和 ±Infinity，优先用它
```

## 七、速查守则

1. 校验「是不是数字」用白名单（typeof + 正则 + isFinite），永远不用「转一下看看炸不炸」的探测法
2. `Number()` 的结果进入业务前先过 `Number.isFinite`
3. 判 NaN 只用 `Number.isNaN`，配合 `Number.isFinite` 更省事
4. 金额与超 2^53 的大数：字符串逐位运算或 BigInt，不让 float 参与
5. 记住 `Number('') === 0`、`Number(null) === 0`、`Number([]) === 0` 这三个「 falsy 变零」链
6. Symbol 是唯一让 `Number()` 抛错的类型，校验器若先转换后判类型，一个 Symbol 就能炸穿
