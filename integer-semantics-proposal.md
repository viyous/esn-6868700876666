# 整数语义提案（讨论稿）

收录日期：2026-11-03

来源：用户提供的与其他 AI 讨论结果。以下保留原提案正文，包括尚需修正的示例与成本判断；它不是已经定稿的语言规范。审阅问题见主文档第五节。

---

# 整数语义规范

## 1. 类型定义

### 1.1 基本类型

8种基本整数类型：`int8` `int16` `int32` `int64` `uint8` `uint16` `uint32` `uint64`

所有类型都是精确位宽，二进制补码（有符号），无符号数按模 2^N 运算（无符号）。

### 1.2 类型配置

整数类型接受以下配置参数：

```text
uint32 {
    range: (min, max),           // 值范围约束，默认为类型的全范围
    overflow: "prove" | "wrap" | "saturate",  // 溢出行为，默认 "prove"
}
```

**配置语义**：

- `range: (0, 100)`：此变量的值保证在 [0, 100] 范围内，编译器可用此信息推导后续运算安全性
- `overflow: "prove"`（默认）：溢出必须在编译期证明不会发生，否则编译错误
- `overflow: "wrap"`：溢出时按模运算回绕，不需要证明
- `overflow: "saturate"`：溢出时饱和到类型边界

**类型等价性**：配置是类型的一部分。`uint32 {overflow: "wrap"}` 和 `uint32` 是不同类型。

**代价说明**：
- `"prove"` 无运行时开销，但需要证明
- `"wrap"` 无运行时开销
- `"saturate"` 每次运算都多2-4条指令（比较和分支或条件移动）

## 2. 算术运算

### 2.1 基本规则

**操作数类型必须完全一致**，否则编译错误：

```text
let a: uint8 = 10
let b: uint16 = 20
let c = a + b        // 错误：类型不匹配
```

**运算结果类型等于操作数类型**，配置继承自操作数：

```text
let a: uint32 {overflow: "wrap"} = ...
let b: uint32 {overflow: "wrap"} = ...
let c = a + b        // c: uint32 {overflow: "wrap"}
```

**如果两操作数配置不同**，编译错误，必须显式统一配置：

```text
let a: uint32 = ...
let b: uint32 {overflow: "wrap"} = ...
let c = a + b        // 错误
let c = a + wrap_cast(b)  // 需要显式转换
```

### 2.2 加减乘除

**加法/减法/乘法**：

```text
let c = a + b
let c = a - b
let c = a * b
```

- `overflow: "prove"`：编译器必须证明结果在类型范围内，否则编译错误
- `overflow: "wrap"`：结果按模运算，`uint32` 相当于模 2^32
- `overflow: "saturate"`：结果超出范围时钳制到边界

**证明方式**：
1. 范围推导：如果 `a: uint32 {range: (0, 1000)}`、`b: uint32 {range: (0, 1000)}`，编译器推出 `a + b` ∈ [0, 2000]，不会溢出
2. 元层断言：`assert(a + b <= UINT32_MAX)`，编译器尝试证明此断言，成功则通过
3. 显式证明（后续设计）

**除法/取余**：

```text
let c = a / b
let c = a % b
```

- 除数必须证明非零，无法证明时编译错误
- 整数除法向零截断（与 C 一致）：`-7 / 2 = -3`，`-7 % 2 = -1`
- 特殊情况：`INT_MIN / -1` 和 `INT_MIN % -1` 归入溢出，需要证明
- 提供替代函数 `div_floor` / `mod_floor`（向下取整）

**代价说明**：
- 除零证明失败时，可用 `div_checked(a, b)` 返回 `Option<T>`，由调用方处理
- 多数除法的除数是常量或循环上界，证明负担不重
- 向下取整的除法有符号时需要额外 2-3 条指令

### 2.3 一元运算

```text
let c = -a
let c = abs(a)
```

- 有符号数取反：`-INT_MIN` 归入溢出，需要证明或使用 `{overflow: "wrap"/"saturate"}`
- `abs` 定义为返回无符号数：`abs(int32) -> uint32`，永不失败

## 3. 位运算

### 3.1 按位逻辑

```text
let c = a & b     // AND
let c = a | b     // OR
let c = a ^ b     // XOR
let c = ~a        // NOT
```

- 有符号/无符号均可
- 结果类型等于操作数类型
- 无溢出问题

### 3.2 移位

```text
let c = a << n    // 左移
let c = a >> n    // 右移
```

**移位量规则**：
- `n` 的类型必须是无符号整数
- 必须证明 `n < 被移位值的位宽`，否则编译错误
- 无法证明时使用 `shl_masked(a, n)`，自动取 `n & (位宽-1)`

**移位语义**：
- 左移 `<<`：
  - 无符号数：逻辑左移，高位丢弃
  - 有符号数：算术左移，遵循操作数的 `overflow` 配置
    - `"prove"`：必须证明移位不丢失有效位
    - `"wrap"`：高位直接丢弃
- 右移 `>>`：
  - 无符号数：逻辑右移，高位补零
  - 有符号数：算术右移，高位补符号位

**循环移位**：

```text
let c = rotl(a, n)   // 循环左移
let c = rotr(a, n)   // 循环右移
```

- `n` 仍需证明 `< 位宽`，或使用自动掩码版本

**代价说明**：
- 移位量证明通常很简单（循环索引、常量）
- `shl_masked` 多一条 AND 指令，现代 CPU 基本无开销
- 循环移位在 x86/ARM 都是单指令

### 3.3 位计数

内建函数，无需证明：

```text
let c = count_ones(a)     // 计数1的个数
let c = count_zeros(a)    // 计数0的个数
let c = leading_zeros(a)  // 前导零个数
let c = trailing_zeros(a) // 尾随零个数
```

返回类型为 `uint8`（足够表示最大位数64）。

## 4. 类型转换

### 4.1 无转换（显式类型断言）

```text
let a: uint32 = 100
let b: uint32 {range: (0, 100)} = narrow(a)
```

`narrow(a)` 从宽类型/宽范围到窄类型/窄范围，必须证明值在目标范围内。

### 4.2 扩宽

```text
widen<uint32>(a: uint8) -> uint32             // 零扩展
widen<int32>(a: int8) -> int32                // 符号扩展
widen<uint64>(a: uint32) -> uint64
```

- 从小位宽到大位宽，有符号扩符号，无符号扩零
- 不需要证明，无信息丢失
- 范围自动扩展：`uint8 {range: (10, 20)}` → `uint32 {range: (10, 20)}`

### 4.3 收窄

```text
narrow<uint8>(a: uint32) -> uint8
```

- 必须证明值在目标类型范围内
- 证明失败时使用显式策略：`truncate(a)` / `saturate(a)` / `narrow_checked(a)`

### 4.4 符号转换

```text
to_signed(a: uint32) -> int32       // 必须证明 a <= INT32_MAX
to_unsigned(a: int32) -> uint32     // 必须证明 a >= 0
```

不满足时使用：
- `reinterpret_as_signed(a: uint32) -> int32`：位模式不变，可能变成负数
- `reinterpret_as_unsigned(a: int32) -> uint32`：位模式不变

`reinterpret_*` 明确表达"我知道符号会变，这是有意为之"。

### 4.5 配置转换

```text
wrap_cast(a: uint32) -> uint32 {overflow: "wrap"}
prove_cast(a: uint32 {overflow: "wrap"}) -> uint32
```

- `wrap_cast`：无开销
- `prove_cast`：需要证明当前值在目标类型范围内（只在边界交接时需要）

## 5. 字面量

字面量是元层的精确整数，赋值时转换为目标类型：

```text
let a: uint8 = 255        // 合法
let b: uint8 = 256        // 编译错误：字面量超出范围
let c = 100               // 错误：无法推导类型
let d: uint32 = 100       // 合法
```

**类型推导**：

```text
let x = a + 10            // 如果 a: uint32，则 10 推导为 uint32
let y = max(a, 100)       // 100 推导为 a 的类型
```

字面量不能推导时报错，不默认为任何类型。

**负字面量**：

```text
let a: int32 = -100       // 合法
let b: uint32 = -1        // 编译错误
```

负号是一元运算符，应用于字面量产生负的精确整数。

## 6. 与 bit 的交互

### 6.1 整数到位视图

```text
let x: uint32 = 0x12345678
let bits: Array<bit, 32> = as_bits(x)
```

- 第0位是LSB（最低有效位），与字节序无关
- 这是值视图，不是内存视图

### 6.2 位到整数

```text
let bits: Array<bit, 32> = ...
let x: uint32 = from_bits(bits)
```

- 长度必须匹配，否则需要显式扩展或截断
- 有符号数需指定扩展方式：`from_bits_signed(bits)` 符号扩展

### 6.3 非对齐长度

```text
let bits: Array<bit, 13> = ...
let x: uint16 = from_bits_extend(bits)     // 高位补零，得到 0..8191
let y: uint16 = from_bits_checked(bits)    // 编译错误，长度不匹配
```

**内存视图转换**：

```text
let x: uint32 = 0x12345678
let bytes_le: Array<uint8, 4> = to_bytes_le(x)   // 小端：[0x78, 0x56, 0x34, 0x12]
let bytes_be: Array<uint8, 4> = to_bytes_be(x)   // 大端：[0x12, 0x34, 0x56, 0x78]
let bytes_ne: Array<uint8, 4> = to_bytes_ne(x)   // 本地序，标记为平台相关
```

- 明确指定字节序，避免编译期求值和运行期不一致
- `_ne` 版本必须显式标记平台依赖，或仅用于不跨平台的场景

## 7. 常见场景的证明负担

### 7.1 循环计数器

```text
for i in 0..100 {
    arr[i] = i * 2       // i ∈ [0, 99]，i * 2 ∈ [0, 198]，不溢出
}
```

编译器从循环范围推导出 `i` 的范围，自动证明。

### 7.2 累加

```text
let sum: uint32 = 0
for x in array {
    sum = sum + x        // 可能溢出，需要证明或改配置
}
```

**选项1**：提供循环不变式

```text
let sum: uint32 {range: (0, 10000)} = 0
for x in array {
    assert(sum + x <= 10000)    // 元层证明或运行时检查
    sum = sum + x
}
```

**选项2**：使用回绕语义

```text
let sum: uint32 {overflow: "wrap"} = 0
for x in array {
    sum = sum + x        // 回绕，用于哈希等场景
}
```

**选项3**：检查式累加

```text
let sum: uint32 = 0
for x in array {
    sum = match add_checked(sum, x) {
        Some(s) => s,
        None => return Error::Overflow
    }
}
```

### 7.3 平均值

```text
let avg = (a + b) / 2
```

不能自动证明（`a + b` 可能溢出）。需要：

```text
let avg = a / 2 + b / 2 + (a % 2 + b % 2) / 2    // 无溢出
```

或者：

```text
let avg = (widen<uint64>(a) + widen<uint64>(b)) / 2
let result = narrow<uint32>(avg)    // 能证明：avg <= UINT32_MAX
```

第二种更清晰，扩宽的开销在这里是值得的。

### 7.4 位操作

```text
let flags: uint32 {overflow: "wrap"} = 0
flags = flags | (1 << bit_pos)
```

- `bit_pos` 需要证明 `< 32`
- `flags` 用 `"wrap"` 因为位操作不关心溢出概念

## 8. C 代码生成策略

### 8.1 类型映射

```text
int8    -> int8_t       (stdint.h)
uint32  -> uint32_t
...
```

保证精确位宽，不依赖 `int` / `long` 的平台差异。

### 8.2 溢出检查的实现

**wrap语义（无符号）**：
```c
uint32_t c = a + b;    // 直接生成，C的无符号运算就是模运算
```

**wrap语义（有符号）**：
```c
// 有符号溢出在C中是UB，需要转换
uint32_t c = (uint32_t)((int32_t)a + (int32_t)b);
// 或使用 -fwrapv 并标记依赖
```

**saturate语义**：
```c
uint32_t c = (a > UINT32_MAX - b) ? UINT32_MAX : a + b;
// 或使用编译器内建函数 __builtin_add_overflow
```

**prove语义**：
```c
// 编译器已证明不溢出，直接生成
uint32_t c = a + b;
// 可选：在debug模式插入断言
assert(a <= UINT32_MAX - b);
```

### 8.3 移位实现

```c
// 右移有符号数，保证算术右移
int32_t c = (int32_t)a >> n;
// C99+保证有符号右移是实现定义的，几乎所有平台是算术右移
// 或用位操作明确实现

// 循环移位
uint32_t c = (a << n) | (a >> (32 - n));   // UB if n=0
// 实际生成带掩码的版本或使用 __builtin_rotleft
```

### 8.4 目标C标准

**建议目标：C11**

- C99：缺少 `<stdatomic.h>` 等，过时
- C11：主流，`_Static_assert`、`_Generic` 等工具齐全
- C17/C23：改进有限，可作为可选目标

有符号整数的补码在C23才强制，但C11之前的实际平台也几乎都是补码。生成的代码需要假定补码，文档注明。

## 9. 总结：默认规则

| 类别 | 默认行为 | 代价 |
|---|---|---|
| 算术溢出 | 必须证明不溢出 | 编译期证明，运行时零开销 |
| 除零 | 必须证明非零 | 编译期证明，运行时零开销 |
| 类型转换 | 不同类型不能混合 | 需显式转换，零开销 |
| 收窄转换 | 必须证明值在范围内 | 编译期证明，运行时零开销 |
| 移位量 | 必须证明 < 位宽 | 编译期证明，运行时零开销 |
| 位视图 | 明确字节序 | 零开销 |

**绕过方式**：每个需要证明的地方都有显式的替代操作（`*_wrap` / `*_saturate` / `*_checked` / `*_masked`），成本透明。

**编译器辅助**：
- 范围自动推导（循环、分支、已有约束）
- 简单不等式求解（线性算术）
- 元层断言和循环不变式
- 清晰的错误信息："无法证明 `x + y` 不溢出，`x` 范围 [0, 1000]，`y` 范围 [0, 2000]，结果可能达到 3000"

## 10. 与元层的边界

对象层整数运算规则已完整定义，元层可以：
- 用编译期求值优化生成代码（常量折叠、范围传播）
- 用符号执行辅助证明
- 生成运行时检查代码（当用户选择 `*_checked` 版本）

元层自己的整数运算（如类型参数计算）遵循精确整数语义，不受对象层限制。

---

**此规范的设计哲学**：

1. **默认安全**：需要证明的地方不多（主要是溢出），且多数场景能自动推导
2. **成本可见**：每个操作的开销一目了然，wrap/saturate/checked 明确写出
3. **不依赖运行时检查**：证明要么编译期完成，要么显式由用户代码处理
4. **可预测的生成代码**：一一对应到C，无隐藏的类型转换或临时变量

**预期使用门槛**：
- 简单算术（计数、索引）：无感，编译器自动处理
- 复杂累加、动态计算：需要理解范围或选择合适的溢出策略
- 不需要写证明脚本，只需要理解"这个值可能超出范围"这个概念
