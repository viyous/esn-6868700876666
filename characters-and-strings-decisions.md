# 字符与字符串：消除歧义决策记录

记录日期：2026-11-03

本文件整理最新讨论中的明确决策和剩余语义问题，不是原回答的逐字副本。更新 [上一轮记录](characters-and-strings-refinements.md)，当前规范以 [主文档](language-design.md) 第八节为准。

## 1. 单字符大小写转换

```text
c.to_uppercase_simple() -> Option<char>
c.to_lowercase_simple() -> Option<char>
```

这里的 simple **不表示Unicode标准的Simple_Uppercase_Mapping或Simple_Lowercase_Mapping属性**，而表示：对单个标量应用默认、无locale的完整映射，恰好得到一个标量时返回Some，否则None。

- 无变化时返回Some(c)。
- a大写为Some(A)，A大写为Some(A)，数字1大写为Some(1)。
- ß完整大写为SS，所以单字符版本返回None。
- İ完整默认小写为i加组合点，所以单字符版本返回None。
- 这是拒绝把多标量结果悄悄缩成单标量的规则，不保证大小写转换可逆或普遍不丢失信息。

完整字符转换返回Iterator<char>；字符串转换返回String，使用包含上下文的Unicode默认规则，不依赖locale。locale版本由库提供。逐字符映射拼接不是字符串完整映射的等价实现。

例如ΟΣ→ος、ΣΟ→σο说明sigma的上下文差异。正式判断条件依Unicode默认规则，不自行简化为只检查字符串末尾。

## 2. 字符与字符串构造

```text
char::from_code(n: u32) -> Option<char>
ascii::from_byte(b: u8) -> Option<ascii>
```

构造函数显式验证；无效输入返回None。char.to_code()和ascii.to_byte()是显式数值转换。ascii <: char、asciistr <: str沿用已定规则，均不因此成为整数类型。

## 3. 三种切片

```text
s[a..b] -> str
s.slice_checked(a, b) -> Option<str>
s.slice_unchecked(a, b) -> str
```

以上a、b为Pos(s)。普通切片必须静态证明a<=b，否则编译错误；checked显式运行期检查顺序；unchecked由程序员保证，违反约定为UB。不引入普通切片隐式panic。

Pos(s)已经保证同属s的有效边界；其他原始字节偏移形式若提供，应另行设计并承担全部边界和顺序前置条件。

原例中的s[0..start]和s[end..s.len]应写为s[s.start()..start]和s[end..s.end()]，不隐式混用整数与位置。

## 4. 位置移动与来源

```text
p.next(s) -> Option<Pos(s)>
p.prev(s) -> Option<Pos(s)>
p.advance(s, n) -> Option<Pos(s)>
p.retreat(s, n) -> Option<Pos(s)>
p.to_parent(s, parent) -> Option<Pos(parent)>
```

- next/prev移动一个Unicode标量边界。末字符起点的next得到Some(end)，end.next得到None，start.prev得到None。
- advance/retreat精确移动n个标量；n=0返回Some(p)，超出两端返回None，恰好到两端成功。
- 同一字符串位置支持==、<、<=。父子位置不直接兼容，转换需证明或验证视图包含关系。
- 成功移动和父子转换的接口契约应保留位置次序/来源事实，使这些事实能辅助普通切片的静态证明；具体依值类型和借用规则仍后续设计。

## 5. 字节视图与ASCII切片

```text
s.bytes -> 只读 [u8; ?]
a[i] -> ascii
a[i..j] -> asciistr
```

asciistr的整数索引/切片仍需静态证明边界与顺序。最新回答又写a.bytes[i..j]->asciistr；这与.bytes作为字节视图存在类型层次歧义：若.bytes保持原始字节切片，切片结果应仍为字节视图，字符串切片直接用a[i..j]。是否保留ASCII来源细化尚待决定，不默默增加特殊返回类型。

## 6. 其他接口方向和示例整理

- 字符分类：is_alphabetic、is_numeric、is_whitespace、is_ascii；其Unicode属性和版本由库规则明确。
- 字符串提供is_empty、is_ascii、starts_with、ends_with、find、rfind。
- 修改String示例使用var result，不引入与现有规则冲突的let mut语法。
- String的push/extend等操作仅为接口方向，拥有数据的字符串和引用规则尚未完成设计。
- 历史回答中“荷兰语IJ标题式大小写应为Ij而非IJ”的说法有误，荷兰语通常保留IJ，如IJsselmeer；该标题式大小写特例不作为本轮默认upper/lower规则。

下一主题保持为引用、别名、生命周期和可变性；当前只记录这些接口需要保证的语义，不先决定依值类型的实现机制。
