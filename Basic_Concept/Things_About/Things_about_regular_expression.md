
abseil:re2 ？

# Abseil 性能技巧 #21 总结：提升 RE2 正则表达式效率
本文发布于2020年（2025年更新），聚焦 C++ 环境下 RE2 正则表达式库的效率优化，核心从“优化正则使用代码”和“优化正则表达式本身”两大维度展开，同时补充多正则匹配的高效方案，适用于 Google 级大规模场景（简单优化可节省数千核资源）。

## 一、优化正则使用代码
核心思路：减少 RE2 对象的构造开销（构造时需解析正则为语法树、编译为自动机，CPU 和内存成本高），优先用轻量字符串操作替代正则。

### 1. 用 Abseil 字符串函数替代字面量正则
正则并非万能，简单匹配场景用 Abseil 专用函数更高效、可读性更强：
- 精确匹配：用 `absl::string_view::operator==()` 替代正则全匹配；
- 子串/前缀/后缀匹配：分别用 `absl::StrContains()`、`absl::StartsWith()`、`absl::EndsWith()`（头文件 `absl/strings/match.h`）；
- 示例：将 `RE2::FullMatch(row, absl::StrCat(row_key, ":.*"))` 优化为 `absl::StartsWith(row, absl::StrCat(row_key, ":"))`。

### 2. 减少 RE2 对象频繁创建（最小化 churn）
RE2 构造开销大，应尽量延长其生命周期：
- 预编译或缓存 RE2 对象，避免循环/高频场景中重复构造；
- 静态/全局正则禁用直接 RE2 实例：改用 `LazyRE2`（延迟构造底层 RE2 对象且不销毁，线程安全）。
- 示例：将临时正则 `R"(.*\.zone(\d+))"` 改为 `static constexpr LazyRE2 kZoneRe = {R"(\.zone(\d+)$)"}`。

## 二、优化正则表达式本身
核心思路：简化正则逻辑、减少不必要计算、避免自动机（DFA）状态爆炸。

### 1. 用 PartialMatch 替代 FullMatch，移除首尾 .*
`FullMatch` 搭配首尾 `.*` 是反模式，`PartialMatch` 支持非锚定搜索，配合 `^`/`$` 锚定可大幅提升效率：
- 移除前导 `.*`：结尾锚定 `$`，例 `.*-(?:bar|qux)-foo` → `-(?:bar|qux)-foo$`（避免反向遍历冗余字符）；
- 移除尾随 `.*`：开头锚定 `^`，例 `foo-(?:bar|qux)-.*` → `^foo-(?:bar|qux)-`（RE2 会用 `memcmp` 优化前缀匹配）；
- 移除首尾 `.*`：直接删除，例 `.*-(?:bar|qux)-.*` → `-(?:bar|qux)-`（避免禁用 `memchr` 优化）；
- 示例：初始正则 `R"(.*\.zone(\d+))"` 优化为 `R"(\.zone(\d+)$)"`，配合 `RE2::PartialMatch`。

### 2. 理解 .* 的真实开销，避免滥用
- 默认含义：`.*` 匹配“零个或多个非换行多字节字符”（RE2 默认 UTF-8 编码），自动机需逐字节遍历，远慢于 `memchr`（SIMD 优化）；
- 核心问题：`.*` 会迫使 RE2 遍历全部冗余字符，无法匹配后立即终止。

### 3. 减少捕获组与子匹配开销
- 非提取场景用非捕获组：用 `(?:...)` 替代 `(...)`，避免不必要的子匹配存储；
- 子匹配用 `absl::string_view`：提取到 `std::string` 或数值类型会产生字符串拷贝，改用 `absl::string_view` + `absl::SimpleAtoi()` 等函数避免拷贝；
- 示例：将 `RE2::PartialMatch(zone_name, *kZoneRe, &zone_id)` 优化为 `RE2::PartialMatch(zone_name, *kZoneRe, &zone_id_str) && absl::SimpleAtoi(zone_id_str, &zone_id)`。

### 4. 减少正则歧义
正则执行成本与歧义性强相关，需满足：① 交替项分支可立即确定；② 重复项结束条件明确：
- 避免模糊匹配：例 `(.+)/` 或 `(.+?)/`（`/` 可能被 `.` 消耗）→ 改为 `([^/]+)/`（`[^/]` 与 `/` 互斥，无歧义）；
- 交替项优化：RE2 会提取交替项的公共前缀（例 `abacus|abalone` → `aba(?:cus|lone)`），建议按字典序排序交替项以最大化优化。

### 5. 避免计数重复（x{n} 类语法）
`x{n}`/`x{n,}`/`x{n,m}` 会在自动机中重复 `x` 至少 `n`/`m` 次，可能导致 DFA 状态爆炸：
- 示例：锚定正则 `^[a-q][^u-z]{13}x` 需 22KiB DFA 内存，非锚定版本 `[a-q][^u-z]{13}x` 需 6MiB；
- 替代方案：优先用 `x+` 等非计数重复，必要时在匹配后单独校验长度。

## 三、多正则匹配高效方案
需避免多次遍历输入字符串时，选择以下两种方案：
- `RE2::Set`：将多个正则合并为“多匹配自动机”，适合短、简单的正则；
- `FilteredRE2`：提取正则中的字面量“原子”，先通过 Aho-Corasick 或 `RE2::Set` 遍历输入找到原子，再筛选可能匹配的正则，适合长、复杂的正则。

## Ref

- [Performance Tip of the Week #21: Improving the efficiency of your regular expressions](https://abseil.io/fast/21)