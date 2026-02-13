# ISON-JVM 代码结构分析报告

## 1. 项目概述

**ISON-JVM** 是 ISON（Intelligent Structured Object Notation）数据格式的 JVM 实现。ISON 是一种极简的、Token 高效的数据格式，专为 LLM（大语言模型）和 Agentic AI 工作流优化设计。相比 JSON，ISON 更紧凑，同时保持了结构化特性，便于 AI 处理。

- **当前版本**: 1.0.0.0
- **开源协议**: MIT License
- **发布渠道**: Maven Central (`com.github.isyscore:ison-jvm:1.0.0.0`)
- **上游项目**: [ISON-format/ison](https://github.com/ISON-format/ison)

---

## 2. 技术栈

| 项目 | 说明 |
|------|------|
| **编程语言** | Kotlin 2.1.20 + Java 17+ |
| **构建工具** | Gradle（Kotlin DSL） |
| **依赖库** | `com.github.isyscore:common-jvm:3.0.0.7` |
| **测试框架** | JUnit 4.13.2 |
| **发布工具** | JReleaser 1.18.0 |
| **产物格式** | JAR, Javadoc, Sources |

---

## 3. 目录结构

```
ison-jvm/
├── .gitignore
├── LICENSE                           # MIT 协议
├── README.md                         # 项目文档
├── build.gradle.kts                  # Gradle 构建配置（含发布/签名）
├── settings.gradle.kts               # Gradle 项目设置
└── src/
    ├── main/kotlin/                  # 核心源码（20 个文件）
    │   ├── ISON.kt                   # 主入口对象
    │   ├── Parser.kt                 # ISON 文本解析器
    │   ├── Document.kt               # 文档模型
    │   ├── Block.kt                  # 块模型（table/object/meta）
    │   ├── Row.kt                    # 行类型别名
    │   ├── Value.kt                  # 值封装
    │   ├── ValueType.kt              # 值类型枚举
    │   ├── Reference.kt              # 引用类型
    │   ├── FieldInfo.kt              # 字段元信息
    │   ├── Dump.kt                   # 序列化输出
    │   ├── Dict.kt                   # 字典转换选项
    │   ├── Schema.kt                 # Schema 验证体系
    │   ├── ValidationError.kt        # 验证错误
    │   ├── ISONANTIC.kt              # 验证功能占位
    │   ├── I.kt                      # Schema 工厂命名空间
    │   ├── Extension.kt              # String 扩展函数
    │   └── ClassExtension.kt         # 对象/列表 ↔ ISON 扩展函数
    └── test/
        ├── kotlin/                   # Kotlin 单元测试
        │   ├── TestISON.kt           # 核心解析/序列化测试
        │   ├── TestISONANTIC.kt      # Schema 验证测试
        │   └── TestObjConvert.kt     # 对象转换测试
        └── java/com/rarnu/ison/test/java/  # Java 单元测试
            ├── TestISON.java         # Java 端核心测试
            └── TestISONANTIC.java    # Java 端 Schema 测试
```

---

## 4. 包结构

所有源代码均位于单一包 `com.rarnu.ison` 下，无子包分层。

---

## 5. 核心类与接口详解

### 5.1 入口层

#### `ISON` (object / 单例)
主入口对象，提供所有顶级 API，所有方法均标注 `@JvmStatic` 以支持 Java 调用。

| 方法 | 说明 |
|------|------|
| `parse(text: String): Document` | 解析 ISON 文本为 Document |
| `load(path: String): Document` | 从文件加载并解析 ISON |
| `toJson(isonText: String): String` | ISON 文本 → JSON 字符串 |
| `fromJson(jsonText: String): Document` | JSON 字符串 → ISON Document |
| `parseISONL(text: String): Document` | 解析 ISONL 流式格式 |
| `loadISONL(path: String): Document` | 从文件加载 ISONL |
| `ISONToISONL(isonText: String): String` | ISON 格式 → ISONL 格式 |
| `ISONLToISON(isonlText: String): String` | ISONL 格式 → ISON 格式 |
| `fromDict(data: Map): Document` | Map → ISON Document |
| `fromDictWithOptions(data, opts): Document` | 带选项的 Map → Document |
| `smartOrderFields(fields): MutableList<String>` | 字段智能排序（id → name → data → ref） |
| `interfaceToValue(v: Any?): Value` | 任意类型 → ISON Value |

### 5.2 数据模型层

#### `Document` (data class)
表示一个完整的 ISON 文档，包含多个 Block。

| 属性 | 类型 | 说明 |
|------|------|------|
| `blocks` | `MutableMap<String, Block>` | 按名称索引的块集合 |
| `order` | `MutableList<String>` | 块的插入顺序 |

| 方法 | 说明 |
|------|------|
| `addBlock(block)` | 添加块 |
| `get(name): Block?` | 按名称获取块 |
| `toDict(): Map` | 转换为 Map |
| `toJson(): String` | 转换为 JSON |

#### `Block` (data class)
表示单个块（table 表格 / object 对象 / meta 元数据）。

| 属性 | 类型 | 说明 |
|------|------|------|
| `kind` | `String` | 块类型: `"table"`, `"object"`, `"meta"` |
| `name` | `String` | 块名称 |
| `fields` | `MutableList<FieldInfo>` | 字段定义列表 |
| `rows` | `MutableList<Row>` | 数据行列表 |
| `summaryRow` | `Row?` | `---` 后的汇总行 |

| 方法 | 说明 |
|------|------|
| `addField(name, typeHint)` | 添加字段定义 |
| `addRow(row)` | 添加数据行 |
| `getFieldNames(): List<String>` | 获取字段名列表 |
| `toDict(): Map` | 转换为 Map |

#### `Row` (typealias)
```kotlin
typealias Row = MutableMap<String, Value>
```
表示一行数据，键为字段名，值为 `Value` 对象。

#### `Value` (data class)
ISON 值的封装类，支持多种类型。

| 属性 | 类型 | 说明 |
|------|------|------|
| `type` | `ValueType` | 值的类型 |
| `boolVal` | `Boolean` | 布尔值 |
| `intVal` | `Long` | 整数值 |
| `floatVal` | `Double` | 浮点值 |
| `stringVal` | `String` | 字符串值 |
| `refVal` | `Reference` | 引用值 |

| 工厂方法 | 说明 |
|----------|------|
| `Value.NULL()` | 创建空值 |
| `Value.BOOL(v)` | 创建布尔值 |
| `Value.INT(v)` | 创建整数值 |
| `Value.FLOAT(v)` | 创建浮点值 |
| `Value.STRING(v)` | 创建字符串值 |
| `Value.REF(v)` | 创建引用值 |

| 访问方法 | 说明 |
|----------|------|
| `isNull(): Boolean` | 是否为空 |
| `asBool(): Boolean?` | 获取布尔值 |
| `asInt(): Long?` | 获取整数值 |
| `asFloat(): Double?` | 获取浮点值（Int 自动转换） |
| `asString(): String?` | 获取字符串值 |
| `asRef(): Reference?` | 获取引用值 |
| `intf(): Any?` | 获取原始值 |
| `toIson(): String` | 转为 ISON 文本表示 |
| `json(): String` | 转为 JSON 表示 |

#### `ValueType` (enum)
```kotlin
enum class ValueType {
    TypeNull, TypeBool, TypeInt, TypeFloat, TypeString, TypeReference
}
```

#### `Reference` (data class)
表示 ISON 引用关系（如 `:1`, `:user:42`, `:OWNS:5`）。

| 属性 | 说明 |
|------|------|
| `id` | 引用目标 ID |
| `namespace` | 命名空间（如 `user`） |
| `relationship` | 关系名（大写，如 `OWNS`） |

| 方法 | 说明 |
|------|------|
| `toIson(): String` | 转为 ISON 引用格式 |
| `isRelationship(): Boolean` | 是否为关系引用 |
| `getNsOrRel(): String` | 获取命名空间或关系名 |
| `json(): String` | 转为 JSON 对象 |

#### `FieldInfo` (data class)
表示字段的元信息。

| 属性 | 说明 |
|------|------|
| `name` | 字段名 |
| `typeHint` | 类型提示（`"int"`, `"float"`, `"bool"`, `"string"`, `"ref"`, `"computed"`, 或空） |

### 5.3 解析层

#### `Parser` (class)
核心解析器，将 ISON 文本解析为 `Document` 结构。

| 属性 | 说明 |
|------|------|
| `text` | 原始文本 |
| `lines` | 按行分割的文本 |
| `pos` | 当前解析位置 |

| 方法 | 说明 |
|------|------|
| `parse(): Document` | 解析完整文档 |
| `parseBlock(kind, name): Block` | 解析单个块 |

| 伴生对象方法（静态） | 说明 |
|----------------------|------|
| `isValidKind(kind): Boolean` | 检查是否为有效块类型 |
| `parseFieldDef(field): Pair` | 解析字段定义（名称:类型） |
| `tokenizeLine(line): List<String>` | 分词（支持引号/转义） |
| `parseReference(token): Reference` | 解析引用 token |
| `parseValue(token, typeHint): Value` | 解析值（支持类型推断） |

**解析流程**:
1. 按行遍历文本
2. 跳过空行和注释（`#`）
3. 识别块头（如 `table.users`）
4. 解析字段定义行
5. 解析数据行（支持 `---` 分隔汇总行）
6. 值解析支持类型提示和自动推断

### 5.4 序列化层

#### `Dump` (object / 单例)
负责将 `Document` 序列化回 ISON/ISONL 格式。

| 方法 | 说明 |
|------|------|
| `dumps(doc): String` | Document → ISON 字符串 |
| `dumpsWithOptions(doc, opts): String` | 带选项序列化 |
| `dump(doc, path/file)` | 写入 ISON 文件 |
| `dumpsISONL(doc): String` | Document → ISONL 字符串 |
| `dumpISONL(doc, path/file)` | 写入 ISONL 文件 |

#### `DumpsOptions` (data class)
序列化选项。

| 属性 | 说明 |
|------|------|
| `alignColumns` | 是否对齐列（视觉对齐） |
| `delimiter` | 列分隔符（默认空格） |

### 5.5 转换层

#### `FromDictOptions` (data class)  — 文件: `Dict.kt`
Map → Document 的转换选项。

| 属性 | 说明 |
|------|------|
| `autoRefs` | 自动检测外键并转换为 Reference |
| `smartOrder` | 智能重排字段顺序 |

#### 扩展函数 — 文件: `Extension.kt`
String 工具扩展。

| 函数 | 说明 |
|------|------|
| `String.containsAny(vararg v)` | 是否包含任一子串 |
| `String.splitN(delimiter, n)` | 限制分割次数 |

#### 扩展函数 — 文件: `ClassExtension.kt`
Kotlin 对象/列表与 ISON 互转的扩展函数。

| 函数 | 说明 |
|------|------|
| `List<T>.toIson(table): String` | 列表 → ISON 表格 |
| `T.toIson(obj): String` | 对象 → ISON object 块 |
| `String.toIsonTable<T>(table): List<T>` | ISON 文本 → 对象列表 |
| `String.toIsonObj<T>(obj): T?` | ISON 文本 → 单个对象 |

### 5.6 Schema 验证层

#### `Schema` (interface)
验证接口定义。

| 方法 | 说明 |
|------|------|
| `validate(v: Any?): Exception?` | 验证值，返回错误或 null |
| `isOptional(): Boolean` | 是否可选 |
| `getDefault(): Pair<Any?, Boolean>` | 获取默认值 |
| `getDescription(): String` | 获取描述 |

#### `BaseSchema` (abstract class)
Schema 的基类，提供通用功能（optional、default、description、refinements）。

#### 具体 Schema 实现

| Schema 类 | 说明 | 特有约束 |
|-----------|------|----------|
| `StringSchema` | 字符串验证 | min/max/length/email/url/regex/refine |
| `NumberSchema` | 数值验证 | min/max/positive/negative/INT()/FLOAT() |
| `BooleanSchema` | 布尔值验证 | — |
| `NullSchema` | 空值验证 | — |
| `RefSchema` | 引用验证 | namespace/relationship |
| `ObjectSchema` | 对象验证 | 字段 Schema 映射，支持 extend/pick/omit |
| `ArraySchema` | 数组验证 | min/max + 元素 Schema |
| `TableSchema` | 表格验证 | 表名 + 行 Schema（复用 ObjectSchema） |
| `DocumentSchema` | 文档验证 | 多块 Schema 映射 |

#### `I` (object / 单例)  — Schema 工厂
类似 Zod 的 `z` 命名空间，提供简洁的 Schema 创建 API。

```kotlin
I.STRING()    I.NUMBER()    I.INT()      I.FLOAT()
I.BOOLEAN()   I.BOOL()      I.NULL()     I.REF()
I.REFERENCE() I.OBJECT(fields)  I.ARRAY(schema)  I.TABLE(name, fields)
```

#### `ValidationError` / `ValidationErrors`

| 类 | 说明 |
|----|------|
| `ValidationError` | 单个验证错误（field + message + value） |
| `ValidationErrors` | 验证错误集合，继承 Exception |

#### `ISONANTIC` (object)
验证功能的占位对象（当前仅含 `VERSION` 常量）。

### 5.7 `SafeParseResult` (data class)
文档 Schema 验证的安全返回值。

| 属性 | 说明 |
|------|------|
| `success` | 是否验证通过 |
| `data` | 验证通过时的数据 |
| `error` | 验证失败时的错误 |

---

## 6. 数据流与核心流程

### 6.1 解析流程 (ISON Text → Document)

```
ISON Text
   │
   ▼
ISON.parse(text)
   │
   ▼
Parser(text, lines, pos)
   │
   ├─ 按行遍历
   ├─ 跳过空行/注释
   ├─ 识别块头 (table.xxx / object.xxx)
   │
   ▼
Parser.parseBlock(kind, name)
   │
   ├─ 解析字段定义行 → FieldInfo[]
   ├─ tokenizeLine() → 分词（引号/转义）
   ├─ parseValue() → 类型推断
   ├─ 解析数据行 → Row[]
   └─ 解析汇总行 (---) → summaryRow
   │
   ▼
Document { blocks, order }
```

### 6.2 序列化流程 (Document → ISON Text)

```
Document
   │
   ▼
Dump.dumps(doc) / Dump.dumpsISONL(doc)
   │
   ├─ 遍历 doc.order 保持顺序
   ├─ 输出块头
   ├─ 输出字段定义
   ├─ Value.toIson() 序列化每个值
   └─ 输出数据行 / 汇总行
   │
   ▼
ISON Text / ISONL Text
```

### 6.3 格式互转

```
ISON Text ←→ Document ←→ JSON String
                ↕
           Map / Dict
                ↕
         Kotlin Objects (via ClassExtension)
```

---

## 7. ISON 格式说明

### 7.1 ISON 格式示例

```
# 注释
table.users
id:int name:string active:bool
1 Alice true
2 Bob false

object.config
key value
debug true
timeout 30

table.orders
id user_id product
1 :1 Widget
2 :user:42 Gadget
---
total 300
```

### 7.2 ISONL 格式示例（流式）

```
table.users|id:int name:string|1 Alice
table.users|id:int name:string|2 Bob
```

### 7.3 支持的值类型

| 类型 | 示例 | 说明 |
|------|------|------|
| 整数 | `42` | 自动推断或 `:int` 提示 |
| 浮点数 | `3.14` | 自动推断或 `:float` 提示 |
| 布尔值 | `true` / `false` | 支持 `TRUE`/`FALSE`/`1`/`0` |
| 字符串 | `Alice` / `"Hello World"` | 含空格时需引号 |
| 空值 | `~` / `null` / `NULL` | 三种写法 |
| 引用 | `:1` / `:user:42` / `:OWNS:5` | 简单/命名空间/关系引用 |

---

## 8. 测试覆盖

### Kotlin 测试（3 个文件）

| 文件 | 测试类 | 测试数量 | 覆盖内容 |
|------|--------|---------|----------|
| `TestISON.kt` | `TestISON` | 30 | 解析、序列化、往返、格式转换、Dict 转换 |
| `TestISONANTIC.kt` | `TestISONANTIC` | 35 | 所有 Schema 验证、I 命名空间 |
| `TestObjConvert.kt` | `TestObjConvert` | 4 | 对象/列表 ↔ ISON 转换 |

### Java 测试（2 个文件）

| 文件 | 测试类 | 测试数量 | 覆盖内容 |
|------|--------|---------|----------|
| `TestISON.java` | `TestISON` | 30 | 与 Kotlin 测试对等，验证 Java 互操作 |
| `TestISONANTIC.java` | `TestISONANTIC` | 33 | 与 Kotlin 测试对等，验证 Java 互操作 |

---

## 9. 构建与发布

### 构建

```bash
gradle build
```

### 测试

```bash
gradle test
```

### 发布到 Maven Central

```bash
gradle clean publishMavenKotlinPublicationToPreDeployRepository
gradle publish jreleaserDeploy
```

### 发布配置
- 签名: GPG（FILE 模式）
- 仓库: Sonatype Central (`https://central.sonatype.com/api/v1/publisher`)
- 产物: JAR + Javadoc JAR + Sources JAR（含 MD5/SHA1 校验）

---

## 10. 依赖关系

### 运行时依赖
- `com.github.isyscore:common-jvm:3.0.0.7` — 提供 `toJson()`, `toObj()` 等 JSON 序列化/反序列化工具

### 测试依赖
- `junit:junit:4.13.2` — JUnit 4 测试框架

---

## 11. 类关系图

```
                    ┌──────────────┐
                    │     ISON     │  (入口单例)
                    │  parse()     │
                    │  fromJson()  │
                    │  fromDict()  │
                    └──────┬───────┘
                           │ uses
                    ┌──────▼───────┐
                    │    Parser    │  (解析器)
                    │  tokenize()  │
                    │  parseValue()│
                    └──────┬───────┘
                           │ produces
                    ┌──────▼───────┐
                    │   Document   │  (文档)
                    │  blocks{}    │
                    │  order[]     │
                    └──────┬───────┘
                           │ contains
                    ┌──────▼───────┐
                    │    Block     │  (块)
                    │  fields[]    │───── FieldInfo {name, typeHint}
                    │  rows[]      │───── Row = Map<String, Value>
                    │  summaryRow  │
                    └──────┬───────┘
                           │ values are
              ┌────────────▼────────────┐
              │         Value           │
              │  type: ValueType        │
              │  boolVal / intVal /     │
              │  floatVal / stringVal / │
              │  refVal: Reference      │
              └─────────────────────────┘

        ┌──────────────────┐
        │    Dump (序列化)  │  Document → ISON/ISONL Text
        └──────────────────┘

        ┌──────────────────┐
        │  Schema (验证)    │  Interface
        │  ├─ BaseSchema   │  Abstract
        │  ├─ StringSchema │
        │  ├─ NumberSchema │
        │  ├─ BooleanSchema│
        │  ├─ NullSchema   │
        │  ├─ RefSchema    │
        │  ├─ ObjectSchema │
        │  ├─ ArraySchema  │
        │  ├─ TableSchema  │
        │  └─ DocumentSchema│
        └──────────────────┘
              │ factory
        ┌─────▼────────┐
        │   I (单例)    │  Schema 创建工厂
        └──────────────┘
```

---

## 12. 总结

ISON-JVM 是一个功能完整的 ISON 数据格式库，具备以下核心能力:

1. **解析** — 支持 ISON / ISONL 两种格式的解析
2. **序列化** — 支持 Document 到 ISON / ISONL / JSON 的输出
3. **格式互转** — ISON ↔ ISONL ↔ JSON ↔ Map ↔ Kotlin 对象
4. **类型系统** — 6 种值类型（null/bool/int/float/string/reference）
5. **Schema 验证** — 完整的验证体系，类似 Zod 的 API 设计
6. **智能排序** — 字段自动重排（id → name → data → ref）
7. **自动引用检测** — 自动识别外键关系
8. **双语言支持** — Kotlin 原生 + Java 完整互操作
