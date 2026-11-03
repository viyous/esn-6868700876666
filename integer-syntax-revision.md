# 整数语法修订稿

收录日期：2026-11-03

来源：用户提供，修订上一版整数提案的语法。以下保留修订稿正文；与已有规则的冲突及未确定细节见主文档第七节。原语义提案作为历史讨论材料保留，语法方向以本次修订为准。

---

# 整数语义规范（语法修订版）

## 1. 类型推导原则

**默认推导，显式标注只在必要时使用**：

```text
let sum = a + b              // 从操作数推导
let result = compute()       // 从返回值推导
let bytes = x.to_bytes_le()  // 从方法签名推导

// 显式标注的必要场景：
let x: u32 = read_input()              // 明确意图
let y: u32{range: (0, 100)} = calc()   // 添加约束
let z: u32 = ambiguous_call()          // 消除歧义（如重载）
```

## 2. 类型名简写

```text
i8  i16  i32  i64      // 代替 int8, int16, ...
u8  u16  u32  u64      // 代替 uint8, uint16, ...
```

简短，一眼识别。配置写法不变：`u32{overflow: "wrap"}`

## 3. 类型转换语法

### 3.1 转换运算符

```text
// as - 安全扩宽（无信息丢失）
let x: u32 = ...
let y = x as u64          // 零扩展
let z = x as i64          // 零扩展（保持数值）

let a: i8 = -10
let b = a as i32          // 符号扩展
let c = a as u32          // 编译错误：有符号到无符号需要证明非负

// as! - 收窄或改变符号（需要证明）
let x: u32 = ...
let y = x as! u8          // 必须证明 x <= 255

let a: i32 = ...
let b = a as! u32         // 必须证明 a >= 0

// as~ - 重新解释位模式
let x: u32 = 0xFFFFFFFF
let y = x as~ i32         // -1（位模式不变）

// as@策略 - 带策略的转换
let x: u32 = 300
let y = x as@truncate u8  // 44（截断低8位）
let z = x as@saturate u8  // 255（饱和）
```

**运算符优先级**：低于算术运算，高于比较运算

```text
let x = a + b as u64      // 相当于 a + (b as u64)，编译错误（类型不匹配）
let y = (a + b) as u64    // 先算后转
```

### 3.2 方法语法（明确参数）

用于需要额外参数或明确意图的转换：

```text
// 字节序转换
let bytes = x.to_bytes_le()        // 推导出 [u8; 4]
let bytes = x.to_bytes_be()
let bytes = x.to_bytes_ne()        // 本地序

// 位视图
let bits = x.to_bits()             // 推导出 [bit; 32]
let back = u32::from_bits(bits)    // 类型函数

// 配置转换（当配置较复杂时）
let wrapped = x.to_type(u32{overflow: "wrap"})
```

### 3.3 类型作为构造函数

```text
// 从位数组构造
let bits: [bit; 32] = ...
let x = u32(bits)                  // 等价于 u32::from_bits(bits)

// 从字节构造（需指定字节序）
let bytes: [u8; 4] = ...
let x = u32::from_le(bytes)
let y = u32::from_be(bytes)
```

### 3.4 属性访问（位视图）

```text
let x: u32 = 0x12345678
let bit5 = x.bits[5]               // 读取第5位
x.bits[5] = 1                      // 修改第5位
let all_bits = x.bits              // 获取完整位数组视图
```

`.bits` 返回一个 `BitView<u32>`，表现为 `[bit; 32]`，但底层仍引用原变量。

## 4. 算术运算的策略标注

### 4.1 默认行为（需要证明）

```text
let c = a + b     // 必须证明不溢出
let d = a * b     // 必须证明不溢出
```

### 4.2 后缀策略

```text
// wrapping 算术（回绕）
let c = a +% b
let d = a -% b
let e = a *% b

// saturating 算术（饱和）
let c = a +| b
let d = a -| b
let e = a *| b

// checked 算术（返回 Option）
let c = a +? b     // 返回 Option<T>
match c {
    Some(val) => ...,
    None => // 处理溢出
}
```

**语法规则**：
- `%` 代表 wrap（想象成"模运算"）
- `|` 代表 saturate（想象成"卡在边界"）
- `?` 代表 checked（Rust 风格）

### 4.3 类型配置影响运算符

```text
let a: u32{overflow: "wrap"} = ...
let b: u32{overflow: "wrap"} = ...
let c = a + b     // 自动使用回绕语义，等价于 a +% b
```

配置是类型的一部分，运算符行为遵循操作数类型。

## 5. 移位运算

### 5.1 默认（需要证明范围）

```text
let c = a << n    // 必须证明 n < 32
let d = a >> n    // 必须证明 n < 32
```

### 5.2 自动掩码版本

```text
let c = a <<% n   // 自动 n & 31
let d = a >>% n   // 自动 n & 31
```

### 5.3 循环移位

```text
let c = a <<< n   // 循环左移（rotl）
let d = a >>> n   // 循环右移（rotr）
```

三个字符的运算符清晰表达"这是特殊的移位"。

## 6. 位运算

保持简洁：

```text
let c = a & b     // AND
let c = a | b     // OR
let c = a ^ b     // XOR
let c = ~a        // NOT
```

## 7. 完整示例对比

### 7.1 字节序转换

**之前**：
```text
let bytes_le: Array<uint8, 4> = to_bytes_le(x)
let bytes_be: Array<uint8, 4> = to_bytes_be(x)
```

**现在**：
```text
let bytes_le = x.to_bytes_le()
let bytes_be = x.to_bytes_be()
```

### 7.2 类型转换

**之前**：
```text
let wider = widen<uint64>(x)
let narrow = narrow<uint8>(x)
let reinterp = reinterpret_as_signed(x)
let wrapped = wrap_cast(x)
```

**现在**：
```text
let wider = x as u64
let narrow = x as! u8
let reinterp = x as~ i32
let wrapped = x as u32{overflow: "wrap"}
// 或者用后缀运算符：
let sum = wrapped +% y
```

### 7.3 位操作

**之前**：
```text
let bits = as_bits(x)
let x = from_bits(bits)
let flag = get_bit(x, 5)
set_bit(x, 5, 1)
```

**现在**：
```text
let bits = x.to_bits()       // 或 x.bits
let x = u32(bits)
let flag = x.bits[5]
x.bits[5] = 1
```

### 7.4 溢出处理

**之前**：
```text
let c = add_wrap(a, b)
let d = add_saturate(a, b)
let e = add_checked(a, b)
```

**现在**：
```text
let c = a +% b
let d = a +| b
let e = a +? b
```

## 8. 命名空间组织

### 8.1 类型相关的函数

类型本身作为命名空间：

```text
u32::from_le(bytes)
u32::from_be(bytes)
u32::from_bits(bits)
i32::MAX
i32::MIN
```

### 8.2 内建函数

```text
// 位计数（作为方法或函数）
let c = x.count_ones()
let c = count_ones(x)      // 也可接受

// 数学函数
let m = max(a, b)
let m = min(a, b)
let a = abs(x)             // 返回无符号数
```

### 8.3 检查函数（返回 bool）

```text
// 值范围检查
if a.would_overflow_add(b) {
    // 处理
}

// 或者用 checked 版本的 Option
```

## 9. 配置简写

常用配置提供类型别名：

```text
type u32w = u32{overflow: "wrap"}      // wrapping u32
type u32s = u32{overflow: "saturate"}  // saturating u32

let hash: u32w = ...
hash = hash *% 31 +% value    // 或者直接用类型配置影响 +
```

## 10. 语法总结表

| 操作 | 语法 | 说明 |
|---|---|---|
| 扩宽 | `x as T` | 无信息丢失，自动扩展 |
| 收窄 | `x as! T` | 需要证明 |
| 重新解释 | `x as~ T` | 位模式不变 |
| 策略转换 | `x as@strategy T` | 明确策略 |
| 回绕运算 | `a +% b` | 模运算 |
| 饱和运算 | `a +\| b` | 钳制到边界 |
| 检查运算 | `a +? b` | 返回 Option |
| 掩码移位 | `a <<% n` | 自动掩码 |
| 循环移位 | `a <<< n` | rotl |
| 位访问 | `x.bits[i]` | 位视图 |
| 字节转换 | `x.to_bytes_le()` | 方法调用 |

---

这个版本：
- **简洁**：运算符短，类型名短，常见操作不超过5个字符
- **有组织**：转换是运算符，方法属于类型，没有全局函数污染
- **一致**：`%` 都表示"允许回绕/掩码"，`?` 表示"可能失败"，`!` 表示"需要证明"
- **可推导**：绝大部分时候不需要写类型
