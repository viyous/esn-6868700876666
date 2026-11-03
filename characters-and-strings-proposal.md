# 字符与字符串：重新设计

收录日期：2026-11-03

来源：用户提供的新一轮讨论结果。本文保留原稿，含暂定方向和未校正示例；当前设计整理及语义待定项见 [主设计文档](language-design.md) 第八节。末尾三项倾向不视为最终定论。

## 1. 字符不是整数的子集

### 1.1 字符是独立的基础类型

```text
char        Unicode 标量值
ascii       ASCII 字符
```

**不是**：

```text
char = u32{range: [...]}   // ✗ 用户不关心码点的数值范围
ascii = u8{range: [0,127]} // ✗ 用户不关心 ASCII 是 0-127
```

**而是**：

```text
char        独立的基础类型，表示"一个 Unicode 字符"
ascii       独立的基础类型，表示"一个 ASCII 字符"
```

### 1.2 字符与整数的转换是显式的

```text
let c: char = 'A'
let code: u32 = c.to_code()        // 显式：字符 → 码点值
let c2: char = char::from_code(65) // 显式：码点值 → 字符（需验证）

let a: ascii = 'A'
let byte: u8 = a.to_byte()         // 显式：ASCII → 字节值
let a2: ascii = ascii::from_byte(65) // 显式：字节值 → ASCII（需验证）
```

**为什么需要显式**：

大多数对字符的操作不关心底层数值：

```text
c.is_alphabetic()
c.is_digit()
c.to_uppercase()  // 返回 char，不是 u32
```

把码点值暴露成"范围受限的整数"，会让用户误以为可以对字符做算术：

```text
// 如果 char 是整数子集，这些看起来是合法的
let c: char = 'A'
let next = c + 1        // 这是什么意思？B？还是非法字符？
let sum = c1 + c2       // 完全没有意义
```

显式转换后，意图清晰：

```text
let c: char = 'A'
let code = c.to_code()
let next_code = code + 1
let next = char::from_code(next_code)?  // 需要验证
```

### 1.3 字符字面量

```text
'a'         char 字面量
'中'        char 字面量
'😀'        char 字面量

// ASCII 需要明确标记或从上下文推导
let a: ascii = 'a'      // 'a' 在 ASCII 范围内，可以实例化为 ascii
let b: ascii = '中'     // 错误：不是 ASCII

// 或者用前缀
ascii'a'
```

## 2. 字符串就是字节序列的视图

### 2.1 基础抽象

```text
str         UTF-8 字符串视图
```

**本质**：`str` 是 `[u8; ?]` 加上"合法 UTF-8"的不变式。

但这个不变式**不需要在类型语法中体现**：

```text
// 不是这样
type str = [u8; ?]{valid_utf8: true}

// 而是
str 是一个独立类型，语言保证它总是合法 UTF-8
```

用户不需要知道"什么叫合法 UTF-8"，只需要知道：

- `str` 是字符串
- 可以迭代字符
- 可以切片（在字符边界）
- 可以和字节序列互转（需要验证）

### 2.2 字节访问是显式的

```text
s: str

s[i]              // 错误：不能直接索引（会破坏抽象）
s.bytes[i]        // OK：明确访问底层字节，返回 u8
s.bytes           // [u8; ?]，字节视图
```

**为什么 `s[i]` 不行**：

两个原因都说不通：

1. 如果 `[i]` 返回字节：为什么不直接写 `.bytes[i]`？
2. 如果 `[i]` 返回字符：那是 O(n) 操作，成本不可见

所以干脆禁止，强制用户明确意图。

### 2.3 字符迭代

```text
for c: char in s.chars() { ... }
```

成本可见：`chars()` 表明这是在做编码解析。

### 2.4 切片需要位置类型

```text
s[a..b]   // a, b 必须是 Pos(s) 类型
```

`Pos(s)` 是依值类型，表示"字符串 s 中的字符边界位置"。

**来源**：

```text
s.start(), s.end()                    // 两端
s.char_indices()                       // 迭代器产生 (Pos(s), char)
s.find(...), s.rfind(...)             // 返回 Option<Pos(s)>
pos.next(s), pos.prev(s)              // 移动到相邻字符
s.validate_pos(i: usize)              // 运行期验证，返回 Option<Pos(s)>
```

**不安全后门**：

```text
s.slice_unchecked(start: usize, end: usize) -> str
// 假设索引在字符边界，跳过验证
// 违反假设是 UB
```

## 3. ASCII 字符串的特殊性

### 3.1 ASCII 字符串可以直接索引

```text
asciistr    ASCII 字符串视图
```

**与 `str` 的区别**：

```text
s: str
s[i]          // 错误

a: asciistr
a[i]          // OK，返回 ascii（前提：i < a.len）
```

**为什么可以**：

- ASCII 是单字节编码
- 字节边界 = 字符边界
- 索引是 O(1)

**类型关系**：

```text
asciistr <: str
```

可以在需要 `str` 的地方使用 `asciistr`：

```text
fn process(s: str) { ... }

let a: asciistr = ascii"hello"
process(a)   // OK
```

但反过来不行，需要验证：

```text
let s: str = "hello"
let a: asciistr = s as? asciistr  // 运行期验证
```

### 3.2 字面量

```text
"hello"           str
ascii"hello"      asciistr（编译期验证只含 ASCII）
```

## 4. 类型系统的层次

```text
基础类型（不是整数）：
  char          Unicode 字符
  ascii         ASCII 字符

字符串视图：
  str           UTF-8 字符串
  asciistr      ASCII 字符串（str 的子类型）

辅助类型：
  Pos(s)        字符串 s 的字符边界位置（依值类型）

原始数据：
  [u8; N]       字节数组
  [u8; ?]       字节切片
```

## 5. 转换矩阵

| 从 | 到 | 语法 | 成本 |
|---|---|---|---|
| `char` | `u32` | `.to_code()` | 零 |
| `u32` | `char` | `char::from_code(n)?` | 验证码点范围 |
| `ascii` | `u8` | `.to_byte()` | 零 |
| `u8` | `ascii` | `ascii::from_byte(b)?` | 验证 < 128 |
| `ascii` | `char` | `as char` | 零（子类型） |
| `char` | `ascii` | `as? ascii` | 验证 < 128 |
| `str` | `[u8;?]` | `.bytes` | 零（视图） |
| `[u8;?]` | `str` | `as? str` | UTF-8 验证 O(n) |
| `asciistr` | `str` | `as str` | 零（子类型） |
| `str` | `asciistr` | `as? asciistr` | 验证 ASCII O(n) |

## 6. 操作的性能特征

| 操作 | 类型 | 成本 |
|---|---|---|
| `s.len` | `str → usize` | O(1)，字节长度 |
| `s.bytes[i]` | `str → u8` | O(1)，访问字节 |
| `s.chars()` | `str → Iterator<char>` | O(n) 总共，迭代所有字符 |
| `s.chars().count()` | `str → usize` | O(n)，字符数量 |
| `s.chars().nth(i)` | `str → char` | O(n)，第 i 个字符 |
| `s[a..b]` | `str, Pos(s), Pos(s) → str` | O(1)，已知边界 |
| `a.len` | `asciistr → usize` | O(1)，字节 = 字符 |
| `a[i]` | `asciistr → ascii` | O(1)，直接索引 |
| `a[i..j]` | `asciistr → asciistr` | O(1)，字节切片 |

## 7. 完整示例

### 7.1 通用字符串处理

```text
fn first_word(s: str) -> str {
    let space_pos = s.find(' ');
    match space_pos {
        Some(pos) => s[s.start()..pos],
        None => s
    }
}
```

### 7.2 ASCII 快速路径

```text
fn parse_hex(s: str) -> Option<u32> {
    // 十六进制只用 ASCII
    let ascii_s = s as? asciistr?;

    var result: u32 = 0;
    for i in 0..ascii_s.len {
        let c = ascii_s[i];  // 直接索引
        let digit = match c {
            '0'..='9' => (c.to_byte() - '0'.to_byte()) as u32,
            'a'..='f' => (c.to_byte() - 'a'.to_byte() + 10) as u32,
            'A'..='F' => (c.to_byte() - 'A'.to_byte() + 10) as u32,
            _ => return None
        };
        result = result * 16 + digit;
    }
    Some(result)
}
```

### 7.3 Unicode 处理

```text
fn has_emoji(s: str) -> bool {
    for c in s.chars() {
        if c.is_emoji() {
            return true;
        }
    }
    false
}
```

### 7.4 字符边界安全

```text
fn split_at_char(s: str, n: usize) -> Option<(str, str)> {
    var count = 0;
    for (pos, _) in s.char_indices() {
        if count == n {
            return Some((s[s.start()..pos], s[pos..s.end()]));
        }
        count += 1;
    }
    None
}
```

## 8. 设计原则总结

| 原则 | 体现 |
|---|---|
| **字符不是数字** | `char` 和 `ascii` 是独立类型 |
| **转换显式** | `.to_code()`, `char::from_code()` |
| **成本可见** | `.chars()` 表示解码，`.bytes[i]` 表示原始访问 |
| **类型保证安全** | `Pos(s)` 保证字符边界 |
| **特例优化** | `asciistr` 支持直接索引 |
| **子类型关系** | `asciistr <: str`，`ascii <: char`（可选） |

---

## 现在需要决定的

1. **`ascii` 是否是 `char` 的子类型**？
   - 如果是：`ascii` 可以直接用在需要 `char` 的地方
   - 如果不是：需要显式转换 `a as char`

   我倾向：**是子类型**，因为 ASCII 字符就是 Unicode 字符的子集。

2. **`Pos(s)` 的具体语法**？
   - `Pos(s)`：类型参数化语法
   - `s.Pos`：作为类型的成员
   - `&pos s`：特殊语法

   我倾向：**`Pos(s)`**，和其他依值类型一致。

3. **`str` 的字面量是否总是在只读段**？
   - 是：所有 `"..."` 都是视图，指向只读数据
   - 否：某些场景可能需要栈上的小字符串

   我倾向：**是**，拥有数据的字符串等内存管理确定后再加。
