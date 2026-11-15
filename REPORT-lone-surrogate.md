# 孤代理（lone surrogate）字符串编码分叉排查报告

## 现象

交换件中混入损坏的代理对字符串（孤代理，即不配对的 UTF-16 高/低代理码元）后：

- 读写往返静默变成替换符（U+FFFD），且不同路径替换出的个数不同；
- 小字符串路径向缓冲区写出非法 UTF-8 字节流；
- 校验层（`StringType.isValid`）对损坏值放行，损坏数据静默入库。

## 三条路径与两种阈值的分叉根因

字符串字面量编码集中在 `lib/utils.js` 的 `Tap.prototype.writeString` 及其辅助函数，
按字符串特征分入三条路径，由两个阈值控制分叉：

**阈值一：21 个 UTF-16 码元**（`writeString` 小/大字符串分叉，选 21 是因为
21×3 字节的 UTF-8 长度上界恰可用单字节 varint 存储）。

**阈值二：4096 字节**（`encodeSlice` 共享编码缓冲区 `encodeBuf` 的大小，
`TextEncoder.encodeInto` 装不下时回退 `TextEncoder.encode`）。

### 路径一：小字符串手写编码（≤ 21 码元，`writeString` 内联循环）

循环逐码元判断：`< 0x80` 写 1 字节，`< 0x800` 写 2 字节，合法代理对写 4 字节，
**其余（含孤代理）落入 `else` 分支，把代理码元本身按 3 字节 UTF-8 形式写出**
（如 U+D800 → `ED A0 80`）。这是 CESU-8 式编码，**不是合法 UTF-8**
（RFC 3629 明确禁止对 U+D800–U+DFFF 编码）。读回时 `readString` 经
`Buffer.utf8Slice`/`TextDecoder` 解码，非法序列的**每个字节**被替换为 U+FFFD，
于是一个孤代理往返后静默变成**三个** U+FFFD。

### 路径二：大字符串 TextEncoder 编码（> 21 码元，`encodeSlice`）

`TextEncoder.encodeInto`（≤ 4096 字节）或 `TextEncoder.encode`（> 4096 字节）
遵循 WHATWG 编码规范，在**编码端**把每个孤代理替换为**一个** U+FFFD
（`EF BF BD`）。写出的是合法 UTF-8，但数据已被静默改写。

### 路径三：缓冲区估算（tap 失效时，`utf8Length`）

`Type.toBuffer` 是两遍式的：第一遍写入 1024 字节共享 `TAP`，溢出后 tap 失效，
此时 `writeString` 只调 `utf8Length`（Node 下即 `Buffer.byteLength`，浏览器下为
`encodeInto` 循环）**计数不写字节**，第二遍按计数分配精确缓冲区重写。
`Buffer.byteLength` 对孤代理按 U+FFFD 口径计 3 字节——第三种语义：
它默认了"替换"行为却从不真正产生字节，与路径一实际写出的 CESU-8 字节
语义不符（仅长度碰巧一致），与路径二的替换口径则是隐式耦合、无任何单一出处。

### 校验层

`StringType._check` 只判断 `typeof val == 'string'`，孤代理值一路放行：
`isValid` 返回 true，`toBuffer` 静默写出（路径一甚至是非法字节流），
损坏数据就此入库。

## 损坏机理小结

同一个孤代理字符串，因长度落在不同阈值两侧，会得到三种结果：
非法字节流（路径一）、编码端单替换符（路径二）、只计数不产生字节且口径
隐式（路径三）。读端统一走容错解码再叠一层替换，损坏被进一步掩盖，
全程无任何报错信号。

## 互操作影响

Avro 规范规定 `string` 必须是合法 UTF-8。路径一产出的 `ED A0 80` 类字节流：

- 会被严格解码的实现直接拒绝（如 Python `bytes.decode('utf-8')` 抛
  `UnicodeDecodeError`、Go `string` 转换出现 `utf8.RuneError`、Java
  `new String(bytes, UTF_8)` 静默替换），即交换件对其他语言消费者
  要么不可读、要么读到的值与 JS 侧读回的值都不一致；
- 同一逻辑值经不同路径序列化得到不同字节，破坏二进制比较
  （`matchBytes`/`matchString` 用于排序与相等判断）与指纹/哈希的一致性。

## 修复方案与取舍

**统一口径：拒绝（抛错），不做替换。**

- `Tap.writeString` 两条写路径在编码前统一检测孤代理并抛出
  `Error('invalid string: lone surrogate')`；估算路径（失效 tap）走同一
  检测，两遍式 `toBuffer` 行为一致。该函数是所有字符串字面量编码
  （含 map 键）的唯一收口点。
- `StringType._check` 同步拒绝含孤代理的字符串，`isValid` 给出真实信号，
  与编码口径一致，损坏数据在入库前即可被拦截。
- 读端保持容错解码不变：历史遗留的非法字节仍能读出（为 U+FFFD），
  不因读侧崩溃影响存量数据消费。

**为什么选抛错而非替换：**

1. 本次事故的核心是"静默"——替换口径（对齐 TextEncoder/WHATWG）只会让
   损坏从"非法字节"变成"静默改写"，数据依然失真且调用方无感知，
   投诉根因并未消除。
2. 孤代理不是合法 Unicode 标量值，无法无损表示；Avro 的 `string` 语义
   也不允许它。在写入边界 fail-fast 能把数据质量问题暴露给产生方，
   而不是转嫁给下游消费者。
3. 替换口径下校验层只能继续放行（替换后"可编码"），`isValid` 失去
  拦截损坏数据的能力；抛错口径下校验层与编码层天然一致。

**保真承诺：** 合法字符（含合法代理对/astral 字符，如 U+1D11E）的往返
完全保真，不回退；仅含孤代理的字符串从"静默损坏"变为"明确抛错"。

## 改动清单

- `lib/utils.js`
  - 新增并导出 `hasLoneSurrogate(str)`：先以 `/[\uD800-\uDFFF]/` 快速
    排除无代理码元的常见情形，再精确逐码元校验配对。
  - `Tap.writeString` 大字符串分支：编码前统一检测并抛错（先于
    `isValid` 分支，估算路径同样生效）。
  - `Tap.writeString` 小字符串分支：`else` 分支内识别孤代理
    （`(c1 & 0xf800) === 0xd800`）并抛错，不再写出 CESU-8 字节。
- `lib/types.js`：`StringType._check` 拒绝含孤代理的字符串。
- `test/test_utils.js`：新增"合法代理对跨路径往返保真"与"孤代理跨
  路径/跨阈值（21 码元、4096 字节、失效 tap）一致抛错"测试。
- `test/test_types.js`：`StringType` 的 invalid 用例补充孤代理值；新增
  类型层跨路径（含共享 TAP 溢出估算路径、map 键）拒绝测试与跨阈值
  合法字符串往返保真测试。

## 验证

- 新增 4 个测试全部通过；全量测试 505 passing。
- 唯一失败项 `specs assembleProtocol "import idl from nested paths"` 在
  未改动的基线快照上同样失败（Windows 路径相关的既有问题），与本次
  改动无关。
- eslint 对改动文件无新增告警（现存 7 处告警均为基线既有）。
