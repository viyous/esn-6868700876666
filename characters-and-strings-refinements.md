# 字符与字符串：精确化规则记录

记录日期：2026-11-03

本文件整理用户转来的新一轮回答及本轮审阅结论，不是原回答的逐字副本。它更新 [最初提案](characters-and-strings-proposal.md)；当前主记录见 [语言设计文档](language-design.md) 第八节。引用、别名、生命周期作为下一主题，工程实现细节暂不展开。

## 1. 本轮明确的决策

- char 是 Unicode 标量值，不包含代理码点；与用户看到的字素簇区分。
- grapheme 分割、迭代等由库提供，语言核心不内置 grapheme 类型；库可说明所采用的 Unicode 数据版本。
- codeunit 仅描述编码单元：UTF-8 为字节、UTF-16 为 u16。本轮没有将 codeunit 新增为独立内置类型。
- ascii 是独立字符类型，并确定 ascii <: char。字符与整数之间仍需显式转换。
- 分开提供简单字符映射、完整字符映射和字符串级映射：简单方法返回 Option<char>，完整方法返回字符迭代器，字符串级方法返回拥有结果数据的 String。简单映射与 Unicode 标准术语的对应尚待校正。
- 不依赖locale的字符映射与库提供的locale版本分开；完整字符串映射是否包含上下文规则需明确。
- Pos(s) 依赖具体字符串视图，支持同属s的位置之间的 ==、<、<= 比较。
- 父子视图位置不可直接复用，采用隔离方案；通过显式转换获得父视图位置。不能只因字符串内容相等而互换位置。
- str 总是不可变视图，.bytes 是只读字节视图。持有安全str时禁止通过其他安全别名修改其底层数据；修改需要独占访问，之后重新验证并创建视图。
- 修改字符串内容由 StrBuf 或类似可变容器承担，String/StrBuf 的完整接口、内存管理和所有权规则后续设计。
- 字符串字面量指向只读静态数据，具体借用和驻留身份规则待引用系统细化。
- as? 返回 Option，显式match或写 (s as? asciistr)? 传播失败；不直接赋给目标类型。
- 十六进制解析使用 checked_mul、checked_add，并在失败时返回None。它们与已有 *?、+? 运算符的接口关系尚待统一；空输入如何处理仍待决定。

## 2. 大小写API方向

```text
c.to_uppercase_simple() -> Option<char>
c.to_lowercase_simple() -> Option<char>
c.to_uppercase_full() -> Iterator<char>
c.to_uppercase_locale(locale) -> Iterator<char>  // 库功能
s.to_uppercase() -> String
s.to_lowercase() -> String
```

需区分两种可能含义：

1. 按 Unicode 标准的简单大小写映射表处理，未映射值通常保持自身。
2. 先计算完整映射，仅当结果为一个标量时返回Some，否则None。

这两种不是同一套规则。例如 İ 的默认简单小写映射是 i，完整默认小写是 i + U+0307；不能仅因完整映射有两个标量，就说 Unicode 简单映射不存在。对 A 做大写、对无大小写字母做映射时返回原值还是None，也需定义。

字符串级映射还可能依赖上下文，例如 ΟΣ 的默认小写是 ος，不能把每个标量单独小写后拼接当成完整字符串算法。locale与上下文是不同维度。

## 3. 位置API方向及剩余条件

```text
s.start(), s.end() -> Pos(s)
s.char_indices() -> Iterator<(Pos(s), char)>
s.find(...), s.rfind(...) -> Option<Pos(s)>
s.validate_pos(i) -> Option<Pos(s)>
p.prev(s) -> Option<Pos(s)>
p.advance(s, n) -> Option<Pos(s)>
```

- 原回答给 p.next(s) -> Pos(s)，但终点没有下一个标量边界，需补充终点行为（Option、报错或其他明确规则）。
- s[a..b] 除同源边界外还需 a<=b；回答提出不能静态判断时panic，与现有普通访问必须静态证明的基线有差异。需明确是否为字符串切片增加例外，或另设显式checked切片。
- 父子转换必须依据视图的包含/来源关系，不是搜索相同文本；重复子串和内容相等不等于视图关系相同。
- parent[6..11] 只是旧示意；当前str切片端点为Pos(parent)，不能据此默许任意整数切片。常量偏移是否能自动验证为Pos仍待定义。
- Pos(s)与字符串视图复制、重新绑定、返回值和生命周期的关系，留待引用主题设计。

## 4. 不变式与安全权限

```text
s.bytes[i] = value  // 安全代码中编译错误，不是允许写入后才算UB
```

安全代码通过不可变借用及只读字节视图阻止破坏UTF-8；通过明确的危险操作绕过这些规则并破坏保证时，才讨论UB责任。asciistr 同样需维持ASCII不变式。

从可变字节存储构造str的结果是Option<str>，成功后的存活视图须限制对底层数据的修改。借用结束后如何恢复修改权限，由后续引用规则决定。

## 5. 字符切分示例勘误

原 split_at_char_inclusive 在检查 count==n 前递增计数，n=1时使用第一个字符的起点，返回空前缀；非空串n等于总数时也会在循环内提前返回，不能靠循环后的分支补救。

以下是预期逻辑的示意，usize等辅助类型及具体证明规则仍待正式设计：

```text
fn split_at_char(s: str, n: usize) -> Option<(str, str)> {
    if n == 0 {
        return Some((s[s.start()..s.start()], s))
    }

    var count: usize = 0
    for (pos, _) in s.char_indices() {
        if count == n {
            return Some((s[s.start()..pos], s[pos..s.end()]))
        }
        count += 1
    }

    if count == n {
        return Some((s, s[s.end()..s.end()]))
    }
    return None
}
```

有效范围是 0..=标量数量；末尾和空串均可切分，超过数量返回None。

## 6. 下一主题

围绕现有str与Pos需求设计引用、别名、生命周期和可变性，再回到拥有数据的String、StrBuf、格式化与拼接。Unicode版本属于库的语义配置；字素簇是文本分割单位，不承诺始终等于屏幕上一个独立字形。
