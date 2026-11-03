# 整数语义规范（完整版·范围统一视角）

收录日期：2026-11-03

本文保留用户提供的最新稿正文。它更新前两版的范围模型；内部冲突、公式修正及与既有规则的衔接见 [主设计文档](language-design.md) 第五节。保留原文不表示其中每个示例和 C 实现均已验证。

## 0. 核心原则

**所有整数类型本质上是范围类型**。一个整数类型由三要素组成：

1. **基础类型**（存储格式）：`u8` `u16` `u32` `u64` `i8` `i16` `i32` `i64`
2. **值范围**：`[下界, 上界]`，表示该值可能的取值区间
3. **溢出行为**：`prove` / `wrap` / `saturate`

基础类型名是完整范围的语法糖：

```text
u32  ≡  u32{range: [0, 4294967295], overflow: "prove"}
i32  ≡  i32{range: [-2147483648, 2147483647], overflow: "prove"}
```

常量是范围退化为单点的类型：

```text
42   →  整数{range: [42, 42]}
```

运算是范围的变换，类型检查是范围的验证。

---

## 1. 类型系统

### 1.1 类型的完整形式

```text
完整类型 ::= 基础类型 { range: [下界, 上界], overflow: 行为 }

基础类型 ::= u8 | u16 | u32 | u64 | i8 | i16 | i32 | i64

行为 ::= "prove"      // 必须证明不溢出（默认）
       | "wrap"       // 模运算
       | "saturate"   // 饱和到边界
```

**常用简写**：

```text
// 只写基础类型，使用默认配置
u32  ≡  u32{range: [0, UINT32_MAX], overflow: "prove"}

// 自定义范围
u32{range: [0, 1000]}  ≡  u32{range: [0, 1000], overflow: "prove"}

// 自定义溢出行为
u32{overflow: "wrap"}  ≡  u32{range: [0, UINT32_MAX], overflow: "wrap"}
```

**常量别名**：

```text
UINT8_MAX  = 255
UINT16_MAX = 65535
UINT32_MAX = 4294967295
UINT64_MAX = 18446744073709551615

INT8_MIN   = -128,    INT8_MAX   = 127
INT16_MIN  = -32768,  INT16_MAX  = 32767
INT32_MIN  = -2147483648,  INT32_MAX  = 2147483647
INT64_MIN  = -9223372036854775808,  INT64_MAX  = 9223372036854775807
```

### 1.2 类型等价

两个类型相等当且仅当三要素完全相同：

```text
u32{range: [0, 1000], overflow: "prove"}
  ≡  u32{range: [0, 1000], overflow: "prove"}

u32{range: [0, 1000]}  ≠  u32{range: [0, 2000]}    // 范围不同
u32{overflow: "wrap"}  ≠  u32{overflow: "prove"}    // 行为不同
u32  ≠  i32                                         // 基础类型不同
```

### 1.3 子类型关系

窄范围是宽范围的子类型（前提：基础类型和溢出行为相同）：

```text
T{range: [a1, a2], overflow: O}  <:  T{range: [b1, b2], overflow: O}
⟺  b1 ≤ a1  且  a2 ≤ b2  （范围包含）

示例：
u32{range: [10, 100]}  <:  u32{range: [0, 200]}   ✓
u32{range: [10, 100]}  <:  u32                    ✓
i32{range: [0, 100]}   <:  i32{range: [-10, 200]} ✓
```

**赋值规则**：子类型可以赋给父类型

```text
let narrow: u32{range: [0, 100]} = ...
let wide: u32{range: [0, 1000]} = narrow   // OK
let wide2: u32 = narrow                    // OK

let wide: u32 = ...
let narrow: u32{range: [0, 100]} = wide    // 错误：需要证明
```

---

## 2. 字面量与常量

### 2.1 字面量的类型

字面量是多态的整数常量，类型从上下文推导：

```text
100  →  forall T. T{range: [100, 100]}
     可以实例化为任何基础类型，只要能容纳

let a: u8 = 100        // 实例化为 u8{range: [100, 100]}
let b: u32 = 100       // 实例化为 u32{range: [100, 100]}
let c: i16 = 100       // 实例化为 i16{range: [100, 100]}

let d: u8 = 300        // 错误：300 超出 u8 范围 [0, 255]
let e: i8 = 200        // 错误：200 超出 i8 范围 [-128, 127]

let f = 100            // 错误：无法推导基础类型
```

### 2.2 负字面量

负号 `-` 作为一元运算符应用于字面量：

```text
-100  →  forall T（有符号）. T{range: [-100, -100]}

let a: i32 = -100      // OK
let b: u32 = -100      // 错误：负数不能用于无符号类型
```

### 2.3 编译期常量

```text
const MAX_SIZE = 1000
// MAX_SIZE: 编译期整数{range: [1000, 1000]}

// 用于类型
let arr: [u8; MAX_SIZE] = ...
let count: u32{range: [0, MAX_SIZE]} = ...
```

---

## 3. 算术运算

### 3.1 运算的类型规则

给定：
```text
a: T{range: [a₁, a₂], overflow: O}
b: T{range: [b₁, b₂], overflow: O}
```

要求：
- 基础类型 `T` 必须相同
- 溢出行为 `O` 必须相同

结果类型的基础类型和溢出行为保持不变，范围根据运算重新计算。

### 3.2 加法

**范围计算**：

```text
a + b  →  T{range: [a₁ + b₁, a₂ + b₂], overflow: O}
```

**溢出检查**（当 `O = "prove"`）：

需要证明 `[a₁ + b₁, a₂ + b₂] ⊆ [T_MIN, T_MAX]`

```text
let a: u32{range: [0, 1000]} = ...
let b: u32{range: [0, 2000]} = ...
let c = a + b
// c: u32{range: [0, 3000]}
// 检查：[0, 3000] ⊆ [0, UINT32_MAX] ✓

let d: u32{range: [0, UINT32_MAX - 100]} = ...
let e: u32{range: [0, 200]} = ...
let f = d + e
// f: u32{range: [0, UINT32_MAX + 100]}
// 检查：UINT32_MAX + 100 > UINT32_MAX ✗
// 编译错误："无法证明不溢出，d 最大 4294967195，e 最大 200，和可能达到 4294967395"
```

**示例**：

```text
// 常量折叠
let a: u32 = 10        // a: u32，当前值范围 [10, 10]
let b: u32 = 20        // b: u32，当前值范围 [20, 20]
let c = a + b          // c: u32，当前值范围 [30, 30]
// 编译器直接生成：uint32_t c = 30;

// 循环计数
for i in 0..100 {
    // i: u32{range: [0, 99]}（从循环范围推导）
    let j = i + 1      // j: u32{range: [1, 100]}，不会溢出
}
```

### 3.3 减法

**范围计算**：

```text
a - b  →  T{range: [a₁ - b₂, a₂ - b₁], overflow: O}
```

注意：下界用 `a₁ - b₂`（最小值减最大值），上界用 `a₂ - b₁`

```text
let a: u32{range: [100, 200]} = ...
let b: u32{range: [10, 50]} = ...
let c = a - b
// c: u32{range: [50, 190]}
// 下界：100 - 50 = 50
// 上界：200 - 10 = 190
```

**无符号数减法的下溢**：

```text
let a: u32{range: [10, 20]} = ...
let b: u32{range: [15, 30]} = ...
let c = a - b
// c: u32{range: [-20, 5]}
// 检查：-20 < 0 ✗
// 编译错误："无符号减法可能下溢"
```

### 3.4 乘法

**范围计算**：

需要考虑所有四种组合，取最小和最大：

```text
a * b  →  T{range: [min(a₁b₁, a₁b₂, a₂b₁, a₂b₂),
                    max(a₁b₁, a₁b₂, a₂b₁, a₂b₂)], overflow: O}
```

**无符号数**：

```text
let a: u32{range: [10, 20]} = ...
let b: u32{range: [3, 5]} = ...
let c = a * b
// c: u32{range: [30, 100]}
// min(10*3, 10*5, 20*3, 20*5) = 30
// max(...) = 100
```

**有符号数**（需考虑负数）：

```text
let a: i32{range: [-10, 10]} = ...
let b: i32{range: [5, 15]} = ...
let c = a * b
// 计算所有组合：
//   (-10) * 5  = -50
//   (-10) * 15 = -150
//   10 * 5     = 50
//   10 * 15    = 150
// c: i32{range: [-150, 150]}
```

### 3.5 除法

**范围计算**（复杂，需考虑符号和除零）：

基本原则：
- 除数必须证明非零：`b₁ > 0` 或 `b₂ < 0`
- 整数除法向零截断

```text
let a: u32{range: [100, 200]} = ...
let b: u32{range: [2, 10]} = ...
let c = a / b
// 除数非零：[2, 10] 不包含 0 ✓
// c: u32{range: [10, 100]}
// 下界：100 / 10 = 10（分子最小，分母最大）
// 上界：200 / 2 = 100（分子最大，分母最小）
```

**除零检查**：

```text
let a: u32{range: [100, 200]} = ...
let b: u32{range: [0, 10]} = ...
let c = a / b
// 错误："除数可能为零，范围 [0, 10] 包含 0"
```

**有符号数特殊情况**：

```text
// INT_MIN / -1 会溢出
let a: i32 = INT32_MIN
let b: i32 = -1
let c = a / b
// 错误："有符号除法溢出，-2147483648 / -1 = 2147483648 超出 i32 范围"
```

### 3.6 取余

```text
a % b  →  T{range: [0, max(|b₁|, |b₂|) - 1], overflow: O}（无符号）
       →  T{range: [-(|b₁|, |b₂|最大值 - 1), |b₁|, |b₂|最大值 - 1], overflow: O}（有符号）
```

C 的取余向零截断，余数符号同被除数。

```text
let a: u32{range: [100, 200]} = ...
let b: u32{range: [7, 13]} = ...
let c = a % b
// c: u32{range: [0, 12]}（余数 < 除数）

let a: i32{range: [-100, 100]} = ...
let b: i32{range: [7, 13]} = ...
let c = a % b
// c: i32{range: [-12, 12]}
```

同样需要除零检查。

### 3.7 一元运算

**取负**：

```text
-a  →  T{range: [-a₂, -a₁], overflow: O}
```

注意上下界交换。

```text
let a: i32{range: [10, 20]} = ...
let b = -a
// b: i32{range: [-20, -10]}

let c: i32 = INT32_MIN
let d = -c
// d: i32{range: [-INT32_MIN, -INT32_MIN]}
//    = i32{range: [2147483648, 2147483648]}
// 错误："取负溢出，-(-2147483648) = 2147483648 超出 i32"
```

**绝对值**：

定义为返回无符号数，永不失败：

```text
abs: i8  → u8
     i16 → u16
     i32 → u32
     i64 → u64
```

范围计算：

```text
abs(a)  →  U{range: [0, max(|a₁|, |a₂|)]}
```

```text
let a: i32{range: [-100, 50]} = ...
let b = abs(a)
// b: u32{range: [0, 100]}
// max(|-100|, |50|) = 100
```

---

## 4. 位运算

### 4.1 按位逻辑运算

```text
a & b, a | b, a ^ b, ~a
```

**范围计算**（保守估计）：

```text
a & b  →  T{range: [0, min(a₂, b₂)]}（无符号）
a | b  →  T{range: [max(a₁, b₁), 2^⌈log₂(max(a₂,b₂))⌉ - 1]}（无符号）
a ^ b  →  T{range: [0, T_MAX]}（保守）
~a     →  T{range: [0, T_MAX]}（无符号）
```

有符号数的位运算范围计算复杂，保守估计为全范围，除非编译器能做更精细的分析。

```text
let a: u32{range: [0, 15]} = ...     // 0b0000..1111
let b: u32{range: [0, 255]} = ...    // 0b0000..11111111
let c = a & b
// c: u32{range: [0, 15]}（AND 不会超过较小的那个）
```

### 4.2 移位运算

#### 4.2.1 左移

**语法**：`a << n`

**要求**：
- `n` 必须是无符号类型
- 必须证明 `n < 基础类型位宽`

**范围计算**（`overflow: "prove"`）：

```text
a << n  →  T{range: [a₁ * 2^n₁, a₂ * 2^n₂], overflow: O}
```

需要证明结果不溢出。

```text
let a: u32{range: [1, 100]} = ...
let n: u8{range: [0, 3]} = ...
let c = a << n
// c: u32{range: [1, 800]}
// 上界：100 * 2³ = 800
// 检查：800 ≤ UINT32_MAX ✓
```

**`overflow: "wrap"`**：

高位丢弃，范围重置为全范围（保守估计）：

```text
let a: u32{overflow: "wrap"} = ...
let c = a << 5
// c: u32{range: [0, UINT32_MAX], overflow: "wrap"}
```

#### 4.2.2 右移

**语法**：`a >> n`

**无符号数（逻辑右移）**：

```text
a >> n  →  T{range: [a₁ / 2^n₂, a₂ / 2^n₁]}
```

```text
let a: u32{range: [100, 800]} = ...
let n: u8{range: [2, 3]} = ...
let c = a >> n
// c: u32{range: [12, 200]}
// 下界：100 / 2³ = 12（向下取整）
// 上界：800 / 2² = 200
```

**有符号数（算术右移）**：

符号位保持，需要考虑负数：

```text
let a: i32{range: [-100, 100]} = ...
let n: u8{range: [2, 2]} = ...
let c = a >> n
// c: i32{range: [-25, 25]}
```

#### 4.2.3 掩码移位

**语法**：`a <<% n`, `a >>% n`

自动对移位量掩码，不需要证明范围：

```text
a <<% n  ≡  a << (n & (位宽 - 1))
```

范围计算同普通移位，但 `n` 的范围自动限制为 `[0, 位宽-1]`。

#### 4.2.4 循环移位

**语法**：`a <<< n`, `a >>> n`

```text
let a: u32{range: [1, 255]} = ...
let n: u8{range: [0, 4]} = ...
let c = a <<< n
// c: u32{range: [0, UINT32_MAX]}（保守估计，任何位模式都可能）
```

循环移位不改变 1 的个数，但范围难以精确计算，保守返回全范围。

### 4.3 位计数

```text
count_ones(a)     → u8{range: [0, 基础类型位宽]}
count_zeros(a)    → u8{range: [0, 基础类型位宽]}
leading_zeros(a)  → u8{range: [0, 基础类型位宽]}
trailing_zeros(a) → u8{range: [0, 基础类型位宽]}
```

如果 `a` 的范围已知，可以更精确：

```text
let a: u32{range: [1, 255]} = ...
let c = leading_zeros(a)
// c: u8{range: [24, 31]}
// 1 有 31 个前导零，255 有 24 个前导零
```

---

## 5. 溢出策略

### 5.1 三种溢出行为

| 行为 | 语义 | 范围影响 |
|---|---|---|
| `prove` | 必须证明不溢出 | 按数学结果计算 |
| `wrap` | 模运算 | 重置为全范围 |
| `saturate` | 饱和到边界 | 钳制到类型边界 |

### 5.2 通过类型配置

```text
let a: u32{overflow: "wrap"} = UINT32_MAX
let b: u32{overflow: "wrap"} = 10
let c = a + b
// c: u32{range: [0, UINT32_MAX], overflow: "wrap"}
// 实际值：9（回绕）
```

### 5.3 通过运算符后缀

**回绕运算**：`+%` `-%` `*%` `<<%`

```text
let a: u32 = UINT32_MAX - 100
let b: u32 = 200
let c = a +% b
// 等价于临时将操作数转换为 wrap 模式
// c: u32{range: [0, UINT32_MAX], overflow: "wrap"}
```

**饱和运算**：`+|` `-|` `*|`

```text
let a: u32 = UINT32_MAX - 100
let b: u32 = 200
let c = a +| b
// c: u32{range: [0, UINT32_MAX], overflow: "saturate"}
// 实际值：UINT32_MAX（饱和）
```

但结果类型的 `overflow` 属性仍是 `"prove"`（除非变量本身声明为 `saturate`）。这些后缀只影响单次运算。

**检查运算**：`+?` `-?` `*?` `/?`

返回 `Option<T>`：

```text
let a: u32 = UINT32_MAX - 100
let b: u32 = 200
let c = a +? b
// c: Option<u32>
match c {
    Some(sum) => ...,
    None => // 处理溢出
}
```

---

## 6. 类型转换

### 6.1 扩宽（无信息丢失）

**语法**：`x as T`

允许的扩宽转换：
- 小位宽 → 大位宽，符号一致
- 无符号 → 有符号，前提是值非负

```text
// 位宽扩展
let a: u8{range: [10, 100]} = ...
let b = a as u32
// b: u32{range: [10, 100]}（范围保持）

let c: i8{range: [-50, 50]} = ...
let d = c as i32
// d: i32{range: [-50, 50]}（符号扩展）

// 无符号→有符号（值非负时安全）
let e: u32{range: [0, 100]} = ...
let f = e as i64
// f: i64{range: [0, 100]}

let g: u32 = ...  // 全范围
let h = g as i64
// 错误："u32 的最大值 4294967295 超出 i32 范围"
// 需要 as! 或确保范围在 [0, INT64_MAX]
```

### 6.2 收窄（需要证明）

**语法**：`x as! T`

需要证明值在目标范围内：

```text
let a: u32{range: [0, 100]} = ...
let b = a as! u8
// 检查：[0, 100] ⊆ [0, 255] ✓
// b: u8{range: [0, 100]}

let c: u32{range: [0, 1000]} = ...
let d = c as! u8
// 错误："无法证明 u32{range: [0, 1000]} 的值能放入 u8"
```

### 6.3 符号转换（需要证明）

```text
// 有符号→无符号
let a: i32{range: [10, 100]} = ...
let b = a as! u32
// 检查：[10, 100] 不包含负数 ✓
// b: u32{range: [10, 100]}

let c: i32{range: [-10, 100]} = ...
let d = c as! u32
// 错误："有符号到无符号转换，范围包含负数 -10"
```

### 6.4 重新解释位模式

**语法**：`x as~ T`

位模式不变，不做范围检查：

```text
let a: u32 = 0xFFFFFFFF
let b = a as~ i32
// b: i32{range: [INT32_MIN, INT32_MAX]}
// 实际值：-1

let c: i8 = -1
let d = c as~ u8
// d: u8{range: [0, 255]}
// 实际值：255
```

结果范围重置为目标类型的全范围。

### 6.5 策略转换

**语法**：`x as@strategy T`

```text
let a: u32 = 300
let b = a as@truncate u8
// b: u8{range: [0, 255]}
// 实际值：44（300 & 0xFF）

let c = a as@saturate u8
// c: u8{range: [0, 255]}
// 实际值：255

let d: i32 = -100
let e = d as@saturate u32
// e: u32{range: [0, UINT32_MAX]}
// 实际值：0（饱和到下界）
```

---

## 7. 控制流中的范围细化

### 7.1 条件分支

```text
let x: u32 = get_input()   // x: u32{range: [0, UINT32_MAX]}

if x < 100 {
    // 此分支内，x: u32{range: [0, 99]}
    let y = x + 50         // y: u32{range: [50, 149]}，证明不溢出
} else if x < 1000 {
    // 此分支内，x: u32{range: [100, 999]}
    let z = x - 50         // z: u32{range: [50, 949]}
} else {
    // 此分支内，x: u32{range: [1000, UINT32_MAX]}
}

// 分支汇合后，x 恢复为 u32{range: [0, UINT32_MAX]}
```

### 7.2 断言

```text
let x: u32 = get_input()
assert(x <= 1000)
// 断言后，x: u32{range: [0, 1000]}

let y = x + 500
// y: u32{range: [500, 1500]}，自动证明不溢出
```

断言失败会导致运行时错误（由元层生成的检查代码）。

### 7.3 循环

```text
for i in 0..100 {
    // i: u32{range: [0, 99]}（从迭代器推导）
    let doubled = i * 2    // u32{range: [0, 198]}
}

for i in start..end {
    // 如果 start 和 end 范围未知
    // i: u32{range: [0, UINT32_MAX]}（保守估计）
}
```

循环不变式（后续设计）可以辅助范围推导。

---

## 8. 与位的交互

### 8.1 整数到位视图

**属性访问**：

```text
let x: u32 = 0x12345678
let bit5 = x.bits[5]       // bit5: bit，读取第5位（LSB = 第0位）
x.bits[5] = 1              // 修改第5位

let all_bits = x.bits      // BitView<u32>，表现为 [bit; 32]
```

`x.bits` 返回值视图，与字节序无关。

**方法转换**：

```text
let x: u32 = ...
let bits = x.to_bits()     // [bit; 32]，复制视图
```

### 8.2 位到整数

**构造函数语法**：

```text
let bits: [bit; 32] = ...
let x = u32(bits)          // 从位构造

// 不对齐长度需要显式策略
let short: [bit; 13] = ...
let y = u16::from_bits_extend(short)   // 零扩展，y: u16{range: [0, 8191]}
```

### 8.3 字节序转换

**明确字节序**：

```text
let x: u32 = 0x12345678
let le = x.to_bytes_le()   // [u8; 4] = [0x78, 0x56, 0x34, 0x12]
let be = x.to_bytes_be()   // [u8; 4] = [0x12, 0x34, 0x56, 0x78]
let ne = x.to_bytes_ne()   // 本地序，标记为平台相关

// 反向
let x = u32::from_le(bytes)
let y = u32::from_be(bytes)
```

**编译期一致性**：

编译期求值和运行期必须使用相同的字节序解释。`_ne` 版本在编译期求值时报错，或要求元层明确指定目标平台。

---

## 9. 代码生成

### 9.1 类型擦除

生成 C 代码时，范围信息被擦除，只保留基础类型：

```text
// 源码
let a: u32{range: [0, 1000]} = ...
let b: u32{range: [0, 2000]} = ...
let c = a + b

// 生成的 C（已证明不溢出）
uint32_t a = ...;
uint32_t b = ...;
uint32_t c = a + b;   // 直接生成，无运行时检查
```

### 9.2 溢出策略的实现

**wrap（无符号）**：

```text
// 源码
let c = a +% b    // a, b: u32

// 生成的 C
uint32_t c = a + b;    // C 的无符号运算就是模运算
```

**wrap（有符号）**：

```text
// 源码
let c = a +% b    // a, b: i32

// 生成的 C（安全方式）
uint32_t c = (uint32_t)((int32_t)a + (int32_t)b);
// 或依赖 -fwrapv
```

**saturate**：

```text
// 源码
let c = a +| b    // a, b: u32

// 生成的 C
uint32_t c;
if (a > UINT32_MAX - b) {
    c = UINT32_MAX;
} else {
    c = a + b;
}
// 或使用编译器内建 __builtin_add_overflow
```

**checked**：

```text
// 源码
let c = a +? b    // 返回 Option<u32>

// 生成的 C（假设 Option 展开为结构体）
struct Option_u32 {
    bool is_some;
    uint32_t value;
};

struct Option_u32 c;
uint32_t temp;
if (__builtin_add_overflow(a, b, &temp)) {
    c.is_some = false;
} else {
    c.is_some = true;
    c.value = temp;
}
```

### 9.3 移位实现

```text
// 算术右移（有符号）
// 源码：a >> n，a: i32, n: u8
// 生成的 C
int32_t c = a >> n;   // C99+ 几乎所有平台是算术右移

// 循环移位
// 源码：a <<< n
// 生成的 C（可移植写法）
uint32_t c = (a << (n & 31)) | (a >> ((-n) & 31));
// 或使用 __builtin_rotateleft32
```

### 9.4 目标 C 标准

**推荐：C11**

- 必需特性：`stdint.h`, `_Static_assert`
- 假定：二进制补码（实际上所有现代平台都是）
- 文档注明：生成的代码依赖补码和算术右移

**可选：C23**

- 强制补码，有符号转换有明确定义
- 但支持度尚不广泛

---

## 10. 完整示例

### 10.1 基本运算

```text
let a: u32{range: [0, 1000]} = read_sensor()
let b: u32{range: [0, 500]} = read_config()

// 自动证明安全
let sum = a + b                    // sum: u32{range: [0, 1500]}
let diff = a - b                   // diff: u32{range: [-500, 1000]}
                                   // 错误："无符号减法可能下溢"

// 条件细化后安全
if a >= b {
    // 此分支内 a: [b, 1000], b: [0, min(a, 500)]
    let diff = a - b               // 安全
}

// 使用 wrap
let hash: u32{overflow: "wrap"} = 0
hash = hash *% 31 +% value         // 哈希计算，允许回绕
```

### 10.2 类型转换

```text
let wide: u32{range: [0, 200]} = ...
let narrow = wide as! u8           // 证明：[0, 200] ⊆ [0, 255] ✓

let input: u32 = get_user_input()  // 全范围
let clamped = input as@saturate u8 // 饱和到 [0, 255]

let signed: i32 = -100
let unsigned = signed as! u32      // 错误：包含负数
let reinterpreted = signed as~ u32 // OK：位模式 0xFFFFFF9C
```

### 10.3 位操作

```text
let flags: u32{overflow: "wrap"} = 0
let bit_pos: u8{range: [0, 31]} = get_bit_index()

// 设置位
flags.bits[bit_pos] = 1

// 等价于
flags = flags | (1 << bit_pos)     // 自动证明 bit_pos < 32
```

### 10.4 网络协议解析

```text
struct IPv4Header {
    version: [bit; 4] {default_value: 4},
    ihl: [bit; 4] {default_value: 5},
    dscp: [bit; 6],
    ecn: [bit; 2],
    total_length: u16,
    // ...
}

fn parse_header(data: [u8; 20]) -> IPv4Header {
    let version = u8::from_bits(data.bits[0..4])
    assert(version == 4)

    let ihl = u8::from_bits(data.bits[4..8])
    // ihl: u8{range: [0, 15]}（从 4 位推导）

    let header_len = ihl * 4
    // header_len: u8{range: [0, 60]}，不会溢出

    // ...
}
```

### 10.5 循环累加的三种方式

```text
// 方式1：提供足够的范围信息
fn sum_small(arr: [u32{range: [0, 100]}; 100]) -> u32 {
    let sum: u32 = 0
    for x in arr {
        sum = sum + x
        // 每次最多加 100，总共 100 次，最多 10000，不溢出
    }
    return sum
}

// 方式2：使用 wrap 语义
fn checksum(data: [u8; ?]) -> u32 {
    let sum: u32{overflow: "wrap"} = 0
    for byte in data {
        sum = sum +% byte as u32
    }
    return sum
}

// 方式3：检查溢出
fn sum_checked(arr: [u32; ?]) -> Option<u32> {
    let sum: u32 = 0
    for x in arr {
        match sum +? x {
            Some(s) => sum = s,
            None => return None
        }
    }
    return Some(sum)
}
```

---

## 11. 语法总览

| 类别 | 语法 | 含义 |
|---|---|---|
| **类型简写** | `u32` | `u32{range: [0, UINT32_MAX], overflow: "prove"}` |
| | `i32` | `i32{range: [INT32_MIN, INT32_MAX], overflow: "prove"}` |
| **类型配置** | `u32{range: [0, 1000]}` | 值范围约束 |
| | `u32{overflow: "wrap"}` | 回绕语义 |
| **算术运算** | `a + b` | 需要证明不溢出 |
| | `a +% b` | 回绕 |
| | `a +\| b` | 饱和 |
| | `a +? b` | 返回 `Option<T>` |
| **移位运算** | `a << n` | 需要证明 `n < 位宽` |
| | `a <<% n` | 自动掩码 |
| | `a <<< n` | 循环左移 |
| | `a >>> n` | 循环右移 |
| **类型转换** | `x as T` | 安全扩宽 |
| | `x as! T` | 收窄（需要证明） |
| | `x as~ T` | 重新解释位模式 |
| | `x as@truncate T` | 截断 |
| | `x as@saturate T` | 饱和 |
| **位访问** | `x.bits[i]` | 访问第 i 位 |
| | `x.bits` | 位视图 |
| | `x.to_bits()` | 转换为位数组 |
| **字节序** | `x.to_bytes_le()` | 小端字节数组 |
| | `x.to_bytes_be()` | 大端字节数组 |
| | `u32::from_le(b)` | 从小端构造 |

---

## 12. 设计哲学总结

1. **统一的范围模型**：所有整数类型本质上是范围类型，运算是范围变换
2. **零成本抽象**：范围信息只在编译期存在，生成的 C 代码零开销
3. **默认安全**：溢出需要证明，无法证明时显式选择策略
4. **可预测性**：每个操作的成本透明，没有隐式转换和临时变量
5. **编译器辅助**：自动范围推导、控制流细化、简单约束求解
6. **显式意图**：`as!` / `+%` / `as~` 等语法清晰表达"我知道这里有风险"

**预期使用门槛**：

- 简单算术（计数器、固定范围）：无感，自动通过
- 动态数据（用户输入、文件长度）：需要理解范围或选择策略
- 不需要写证明脚本，但需要理解"这个值可能超出预期范围
