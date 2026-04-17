# asun

[![Pub Version](https://img.shields.io/pub/v/asun.svg)](https://pub.dev/packages/asun)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

高性能 [ASUN](https://github.com/asunLab/asun)（Array-Schema Unified Notation）Dart 编解码库 —— 一种面向 LLM 交互和大规模数据传输的高效序列化格式。

[English](https://github.com/asunLab/asun-dart/blob/main/README.md)

## 什么是 ASUN？

ASUN 将 **Schema** 与 **数据** 分离，消除了 JSON 中每个对象都重复出现 Key 的冗余。Schema 只声明一次，数据行仅保留纯值：

```text
JSON (100 tokens):
{"users":[{"id":1,"name":"Alice","active":true},{"id":2,"name":"Bob","active":false}]}

ASUN (~35 tokens, 节省 65%):
[{id@int, name@str, active@bool}]:(1,Alice,true),(2,Bob,false)
```

| 方面                   | JSON         | ASUN                 |
| ---------------------- | ------------ | -------------------- |
| Token 效率             | 100%         | 30–70% ✓             |
| Key 重复               | 每个对象都有 | 声明一次 ✓           |
| 人类可读               | 是           | 是 ✓                 |
| 嵌套结构               | ✓            | ✓                    |
| 字段绑定与基本类型提示 | 无           | 支持 ✓               |
| 序列化速度             | 1x           | **~1.7–2.6x 更快** ✓ |
| 数据体积               | 100%         | **40–55%** ✓         |

## 快速开始

在 `pubspec.yaml` 中添加：

```yaml
dependencies:
  asun: ^1.0.0
```

### 序列化与反序列化结构体

```dart
import 'package:asun/asun.dart';

class User implements AsunSchema {
  final int id;
  final String name;
  final bool active;

  User({required this.id, required this.name, required this.active});

  @override List<String> get fieldNames => ['id', 'name', 'active'];
  @override List<String?> get fieldTypes => ['int', 'str', 'bool'];
  @override List<dynamic> get fieldValues => [id, name, active];

  factory User.fromFields(Map<String, dynamic> m) => User(
    id: m['id'] as int, name: m['name'] as String, active: m['active'] as bool,
  );
}

void main() {
  final user = User(id: 1, name: 'Alice', active: true);

  // 序列化
  final s = encode(user);
  assert(s == '{id,name,active}:(1,Alice,true)');

  // 反序列化
  final user2 = decodeWith(s, User.fromFields);
  assert(user2.id == 1 && user2.name == 'Alice');
}
```

### 带基本类型提示的序列化

使用 `encodeTyped` 输出带基本类型提示的 schema 绑定文本 —— 适用于文档生成、LLM 提示词和跨语言数据交换：

```dart
final s = encodeTyped(user);
// 输出: {id@int,name@str,active@bool}:(1,Alice,true)

// 反序列化同时支持带基本类型提示和不带基本类型提示的 Schema
final user2 = decodeWith(s, User.fromFields);
```

### 序列化与反序列化 List（Schema 驱动）

对于 `List<AsunSchema>`，ASUN 只写入一次 Schema，每个元素以紧凑元组形式输出 —— 这是相比 JSON 的核心优势：

```dart
final users = [
  User(id: 1, name: 'Alice', active: true),
  User(id: 2, name: 'Bob', active: false),
];

// 不带基本类型提示的 Schema
final s = encode(users);
// 输出: [{id,name,active}]:(1,Alice,true),(2,Bob,false)

// 带基本类型提示的 Schema
final s2 = encodeTyped(users);
// 输出: [{id,name,active}]:
    (1,Alice,true),
    (2,Bob,false)

// 反序列化 —— 两种格式均可
final users2 = decodeListWith(s, User.fromFields);
```

## 支持的类型

| 类型         | ASUN 表示            | 示例                     |
| ------------ | -------------------- | ------------------------ |
| int          | 纯数字               | `42`, `-100`             |
| float        | 带小数点             | `3.14`, `-0.5`           |
| bool         | 字面量               | `true`, `false`          |
| str          | 无引号或有引号       | `Alice`, `"Carol Smith"` |
| null         | 留空                 | _(空白)_ 表示 null       |
| List         | `[v1,v2,v3]`         | `[rust,go,python]`       |
| 键值条目列表 | `[(key,value), ...]` | `[(age,30),(score,95)]`  |
| 嵌套结构体   | `(field1,field2)`    | `(Engineering,500000)`   |

当前 ASUN 格式刻意不支持原生 `Map<K,V>` 字段。
如果你需要键值集合，请显式建模成 entry-list 数组：

```text
{attrs@[{key@str,value@int}]}:([(age,30),(score,95)])
```

### 嵌套结构体

```dart
class Dept implements AsunSchema {
  final String title;
  // ...fieldNames/fieldTypes/fieldValues
}

class Employee implements AsunSchema {
  final String name;
  final Dept dept;
  // ...fieldNames/fieldTypes/fieldValues
}

// Schema 自动反映嵌套结构：
// {name@str,dept@{title@str}}:(Alice,(Engineering))
```

### 可选字段

```text
// 有值:    {id,label}:(1,hello)
// null: {id,label}:(1,)
```

### 数组与键值条目

```text
// 数组字段:
{name,tags@[]}:(Alice,[rust,go,python])

// 键值条目字段:
{name,attrs@[{key@str,value@int}]}:(Alice,[(age,30),(score,95)])
```

### `@` 字段绑定与可选基本类型提示

在 ASUN schema 中，`@` 是字段与其后续 schema/type 描述之间的**字段绑定符**。对终端基本类型来说，`@type` 是可选提示：

```text
// 不带基本类型提示（encode 的默认输出）
{id,name,salary,active}:(1,Alice,5000.50,true)

// 带基本类型提示（encodeTyped 的输出）
{id@int,name@str,salary@float,active@bool}:(1,Alice,5000.50,true)
```

对终端字段，`@int`、`@str`、`@float`、`@bool` 是可选提示；对复杂字段，`@{...}` 和 `@[...]` 是必需的结构绑定，不能省略。

**适用场景：**

- LLM 提示词 — 帮助模型理解并生成正确的数据
- API 文档 — 无需外部文档即可自描述 Schema
- 跨语言数据交换 — 消除类型歧义（`42` 是 `int` 还是 `float`？）
- 调试 — 一眼看出数据类型

### 注释

```text
/* 用户列表 */
[{id@int, name@str, active@bool}]:(1,Alice,true),(2,Bob,false)
```

### 多行格式

```asun
[{id@int, name@str, active@bool}]:
  (1, Alice, true),
  (2, Bob, false),
  (3, "Carol Smith", true)
```

## API 参考

### 文本格式

| 函数                             | 说明                                                  |
| -------------------------------- | ----------------------------------------------------- |
| `encode(value)`                  | 序列化 → 不带基本类型提示的 Schema `{id,name}:`       |
| `encodeTyped(value)`             | 序列化 → 带基本类型提示的 Schema `{id@int,name@str}:` |
| `decode(input)`                  | 反序列化为动态字段袋 / List                           |
| `decodeWith(input, factory)`     | 反序列化为类型化对象                                  |
| `decodeListWith(input, factory)` | 反序列化为类型化列表                                  |

### 美化格式

| 函数                       | 说明                              |
| -------------------------- | --------------------------------- |
| `encodePretty(value)`      | 美化格式 ASUN（不带基本类型提示） |
| `encodePrettyTyped(value)` | 美化格式 ASUN（带基本类型提示）   |

### 二进制格式

| 函数                                                 | 说明                 |
| ---------------------------------------------------- | -------------------- |
| `encodeBinary(value)`                                | 编码为紧凑二进制字节 |
| `decodeBinaryWith(data, fields, types, factory)`     | 解码为类型化对象     |
| `decodeBinaryListWith(data, fields, types, factory)` | 解码为类型化列表     |

## Bench 输出

通过下面命令运行自带 benchmark：

```bash
dart run example/bench.dart
```

现在 Dart 版 benchmark 已经和 Go 版统一成 JSON / ASUN / BIN 的输出风格：

```text
  Flat struct × 500 (8 fields, vec)
    Serialize:   JSON 16.22ms/60784B | ASUN 10.11ms(1.6x)/28327B(46.6%) | BIN 4.92ms(3.3x)/37230B(61.2%)
    Deserialize: JSON    22.09ms | ASUN     5.70ms(3.9x) | BIN     2.11ms(10.5x)
```

这里的 `(46.6%)` 表示 ASUN 体积是 JSON 的 `46.6%`，不是“节省了 46.6%”。
具体耗时会随 CPU、Dart VM 版本以及数据结构深度而变化。

## 为什么 ASUN 表现更好？

1. **零键哈希** — Schema 只解析一次，字段按位置匹配，不需要重复做 key 查找。
2. **Schema 缓存** — 解析后的 schema 头会全局缓存，热点场景下不会重复做同样的 header 工作。
3. **模式驱动解析** — 嵌套对象和数组通过 `@{...}` / `@[...]` 结构支架被稳定解析。
4. **最小化分配** — 文本解码直接在源字符串上工作，只在结果值真正需要时才分配。
5. **体积更小** — 多条记录共享同一个 schema，重复字段名不会像 JSON 那样一遍遍出现。

## 示例

```bash
# 基础用法
dart run example/basic.dart

# 全面测试（全类型、嵌套结构、边界用例）
dart run example/complex.dart

# 性能基准（ASUN vs JSON，吞吐量，体积比较）
dart run example/bench.dart
```

## ASUN 格式规范

完整的 [ASUN 规范](https://github.com/asunLab/asun/blob/main/docs/ASUN_SPEC_CN.md) 包含语法规则、BNF 文法、转义规则、类型系统及 LLM 集成最佳实践。

### 语法速查表

| 元素     | Schema 语法                 | 数据语法            |
| -------- | --------------------------- | ------------------- |
| 对象     | `{field1@type,field2@type}` | `(val1,val2)`       |
| 简单数组 | `field@[type]`              | `[v1,v2,v3]`        |
| 对象数组 | `field@[{f1@type,f2@type}]` | `[(v1,v2),(v3,v4)]` |
| 键值条目 | `field@[{key@str,value@T}]` | `[(k1,v1),(k2,v2)]` |
| 嵌套对象 | `field@{f1@type,f2@type}`   | `(v1,(v3,v4))`      |
| 空值     | —                           | _(空白)_            |
| 空字符串 | —                           | `""`                |
| 注释     | —                           | `/* ... */`         |

## 许可证

MIT

## Contributors

- [Athan](https://github.com/athxx)
