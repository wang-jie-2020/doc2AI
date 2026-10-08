
---
## 😀 默认助手

 | 类型 | 是否含日期 | 是否含时间 | 时区/偏移量 | 是否表示绝对时间点 | 典型用途 |
|---|---:|---:|---|---:|---|
| `java.util.Date` | 是 | 是 | 无时区信息，内部是毫秒时间戳 | 是 | 遗留 API、兼容旧代码 |
| `java.sql.Date` | 是 | 否 | 无 | 否 | SQL `DATE`，仅年月日 |
| `java.sql.Time` | 否 | 是 | 无 | 否 | SQL `TIME`，仅时分秒 |
| `java.sql.Timestamp` | 是 | 是 | 无 | 近似是 | SQL `TIMESTAMP`，支持纳秒 |
| `LocalDate` | 是 | 否 | 无 | 否 | 生日、账期、日期 |
| `LocalTime` | 否 | 是 | 无 | 否 | 每日营业时间、闹钟时间 |
| `LocalDateTime` | 是 | 是 | 无 | 否 | 不涉及时区的本地日期时间 |
| `Instant` | 否 | 是 | UTC | 是 | 时间戳、日志、事件发生时间 |
| `OffsetDateTime` | 是 | 是 | 固定偏移量，如 `+08:00` | 是 | 接口传输、保存带偏移量的时间 |
| `ZonedDateTime` | 是 | 是 | 时区，如 `Asia/Shanghai` | 是 | 跨时区计算、夏令时处理 |
| `OffsetTime` | 否 | 是 | 固定偏移量 | 否 | 带偏移量的时间，不常用 |

### 核心区别

| 需求 | 推荐类型 |
|---|---|
| 只保存日期 | `LocalDate` |
| 只保存时间 | `LocalTime` |
| 保存本地日期时间，不考虑时区 | `LocalDateTime` |
| 保存全球唯一时间点 | `Instant` |
| 保存日期时间和固定偏移量 | `OffsetDateTime` |
| 根据时区规则进行计算 | `ZonedDateTime` |
| 兼容旧 API | `java.util.Date` |
| 对应数据库 `DATE` | `java.sql.Date` |
| 对应数据库 `TIME` | `java.sql.Time` |
| 对应数据库 `TIMESTAMP` | `java.sql.Timestamp` |

补充：

- `Instant` 表示 UTC 时间线上的一个点，例如 `2025-01-01T00:00:00Z`。
- `OffsetDateTime` 只记录固定偏移量，不记录时区规则；`+08:00` 不等同于 `Asia/Shanghai`。
- `ZonedDateTime` 同时记录时间和时区规则，能够处理夏令时。
- 新代码优先使用 `java.time`；`java.sql.*` 主要用于数据库兼容，现代 JDBC 通常可直接映射 `LocalDate`、`LocalDateTime`、`Instant` 等类型。


---
## 😀 默认助手



## MyBatis 交互表现对比

| 类型 | MyBatis 3.5.0+ 内置 TypeHandler | 常用 JDBC 映射 | 时区影响 | 数据库字段建议 |
|---|:---:|---|---|---|
| `java.util.Date` | ✅ (`DateTypeHandler`) | `setTimestamp` / `setDate` | 取值为 JVM 默认时区 | TIMESTAMP / DATE |
| `LocalDateTime` | ✅ (`LocalDateTimeTypeHandler`) | `setObject(TIMESTAMP)` | ❌ 无时区信息，存与取取决于驱动/JVM 默认时区 | TIMESTAMP / DATETIME |
| `Instant` | ✅ (`InstantTypeHandler`) | `setObject(TIMESTAMP)` | ✅ UTC 绝对时刻，与时区无关 | TIMESTAMP / DATETIME |
| `ZonedDateTime` | ✅ (`ZonedDateTimeTypeHandler`) | `setObject(TIMESTAMP)` | ⚠️ 存入时转为 UTC 时间戳，**时区信息会丢失** | TIMESTAMP / DATETIME |
| `OffsetDateTime` | ✅ (`OffsetDateTimeTypeHandler`) | `setObject(TIMESTAMP_WITH_TIMEZONE)` | ⚠️ PostgreSQL/Oracle 可保留偏移量，MySQL 只会存为时间戳 | TIMESTAMP / TIMESTAMP WITH TIME ZONE |

### 关键提示

1. **MyBatis 3.5.0+** 已内置 `LocalDateTimeTypeHandler`、`InstantTypeHandler`、`OffsetDateTimeTypeHandler`、`ZonedDateTimeTypeHandler`，可直接使用；低版本需自定义 `TypeHandler` 或引入 `mybatis-typehandlers-jsr310`。
2. **时区陷阱**：`LocalDateTime` 不含时区，写入数据库时会按 **驱动/JVM 时区** 转换；`Instant` 是 UTC 绝对时刻，不受时区干扰，推荐用于分布式系统。
3. **数据库字段差异**：
   - MySQL `DATETIME`：存储字面值，**不受时区影响**
   - MySQL `TIMESTAMP`：存时按会话时区转换，取时再转换回来，**受时区影响**
   - PostgreSQL `TIMESTAMP WITH TIME ZONE`：可保留 `OffsetDateTime` 的偏移量
4. **时区丢失**：`ZonedDateTime` / `OffsetDateTime` 写入 MySQL `TIMESTAMP`/`DATETIME` 时，**时区/偏移量会丢失**，仅保留时间点；需要时区信息时返回 `LocalDateTime`。
5. **推荐做法**：
   - 数据库 + 应用统一使用 `Instant`（UTC），展示层再转 `ZonedDateTime` 或 `LocalDateTime`
   - MySQL 优先 `DATETIME(6)`，避免 `TIMESTAMP` 的 2038 限制和时区干扰
   - 在 JDBC URL 显式指定时区，例如 `serverTimezone=Asia/Shanghai&forceConnectionTimeZoneToServer=true`（MySQL Connector/J 8+）


---
## 😀 默认助手



## 根本原因

`Instant` 在 ISO-8601 标准中**只能用 `Z` 结尾表示**（UTC 绝对时刻）。  
Jackson 默认的 `InstantDeserializer` 严格遵循此规则，使用 `DateTimeFormatter.ISO_INSTANT`，遇到 `+08:00` 直接拒绝。

---

## 解决方案对比

| 方案 | 实现位置 | 推荐度 | 适用场景 |
|---|---|:---:|---|
| 前端统一输出 `Z` UTC | 前端 `toISOString()` | ⭐⭐⭐⭐⭐ | 跨系统、跨端协作 |
| 后端改用 `OffsetDateTime` 接收 | Java 方法签名 | ⭐⭐⭐⭐ | 对外接口、字段不定 |
| 自定义 Jackson `Instant` 反序列化器 | 后端配置 | ⭐⭐⭐ | 老接口改造、兜底 |
| 注册宽松反序列化器（全局生效） | Spring Boot 配置 | ⭐⭐⭐⭐ | 一次性解决所有接口 |

---

## 推荐方案：组合使用

### 1. 前端标准做法（最简单）

```js
new Date().toISOString()   // "2026-09-23T07:58:02.000Z"
// 不使用：toLocaleString()、toJSON().replace('Z','+08:00') 等
```

### 2. 后端全局兜底配置

```java
@Configuration
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer instantCustomizer() {
        return builder -> builder.deserializerByType(
            Instant.class,
            new InstantDeserializer<>(
                InstantDeserializer.INSTANT,
                null,
                DateTimeFormatter.ISO_OFFSET_DATE_TIME,  // 关键：接受 +08:00
                null, null, false
            )
        );
    }
}
```

这样 `Instant` 字段可同时接受两种格式：

```
"2026-09-23T15:58:02+08:00"  → Instant = 2026-09-23T07:58:02Z
"2026-09-23T07:58:02Z"        → Instant = 2026-09-23T07:58:02Z
```

---

## 关键提示

1. **原则**：API 层只输出 `Z` UTC 字符串，**绝不**输出 `+08:00` 等带本地时区的偏移量，避免多端解析不一致。
2. **最稳妥**：前端 `new Date().toISOString()` + 后端全局兜底配置，两端都兼容。
3. **验证后端能力**：写完 `curl -H "Content-Type: application/json" -d '{"time":"2026-09-23T15:58:02+08:00"}'` 测一下能否正常绑定字段。
4. **避免在控制器内转换**：不要在每个接口用 `OffsetDateTime.parse(req.getTime()).toInstant()` 重复造轮子，统一在 Jackson 层解决。
5. **DTO 字段类型选型**：单一公司内部用 `Instant` 即可；对外公共服务/三方接口可考虑 `OffsetDateTime`，更宽容。


---
## 😀 默认助手



## 核心定义

`OffsetDateTime` = `LocalDateTime` + `ZoneOffset`  
表示**带 UTC 偏移量的日期时间**，但不含时区规则（如夏令时）。  

```
2026-09-23T15:58:02.123+08:00
├─────── LocalDateTime ───────┤└─ ZoneOffset ─┘
```

---

## 字段组成

| 字段 | 示例 | 说明 |
|---|---|---|
| `LocalDate` | `2026-09-23` | 年月日 |
| `LocalTime` | `15:58:02.123` | 时分秒 + 纳秒精度 |
| `ZoneOffset` | `+08:00` | UTC 偏移量（`-18:00` ~ `+18:00`） |

**不可变对象**（类似 `String`），所有修改方法返回新实例。

---

## 与同类对比

| 类 | 时区 | 偏移量 | 适用场景 |
|---|:---:|:---:|---|
| `LocalDateTime` | ❌ | ❌ | 仅表示"几点几分"，与时区无关 |
| `Instant` | ❌ | ❌ | UTC 时间戳，跨系统首选 |
| **`OffsetDateTime`** | ❌ | ✅ | **跨时区数据传输**（API、客户端时间） |
| `ZonedDateTime` | ✅ | ✅ | 含夏令时规则的本地时间 |
| `OffsetTime` / `LocalDate` | — | — | 仅时间/日期场景 |

**关键区别**：
- `OffsetDateTime` 偏移量是**固定值**（如 `+08:00`），无历史规则
- `ZonedDateTime` 带**时区规则**（如 `Asia/Shanghai`），能处理夏令时切换

---

## 常用 API

```java
// 构造
OffsetDateTime.now()                              // 当前时间 + JVM 默认偏移量
OffsetDateTime.of(2026, 9, 23, 15, 58, 2, 0, ZoneOffset.ofHours(8))
OffsetDateTime.parse("2026-09-23T15:58:02+08:00")
OffsetDateTime.ofInstant(Instant.now(), ZoneOffset.UTC)

// 转换
.toInstant()                                     // → Instant (UTC)
.toLocalDateTime()                               // → LocalDateTime（剥离偏移）
.toZonedDateTime()                               // → ZonedDateTime（需配合 ZoneId）
.atZoneSameInstant(ZoneId.of("Asia/Shanghai"))  // 转另一时区的 ZonedDateTime

// 修改偏移量（不同时区换算）
.withOffsetSameInstant(ZoneOffset.UTC)           // 保留绝对时刻，仅改偏移量
.withOffsetSameLocal(ZoneOffset.ofHours(-5))     // 保留本地时间，仅改偏移量

// 字段访问
.getYear() / .getMonth() / .getDayOfMonth()
.getHour() / .getMinute() / .getSecond() / .getNano()
.getOffset()                                     // ZoneOffset
```

---

## 序列化（Jackson）

默认输出 ISO-8601 格式：

```
"2026-09-23T15:58:02+08:00"
```

前端解析：
```js
const d = new Date("2026-09-23T15:58:02+08:00")
d.toISOString()  // "2026-09-23T07:58:02.000Z"（自动转 UTC）
```

---

## 数据库映射

| 数据库 | 推荐字段类型 | 保留偏移量 |
|---|---|:---:|
| **PostgreSQL** | `TIMESTAMP WITH TIME ZONE` | ✅ |
| **Oracle** | `TIMESTAMP WITH TIME ZONE` | ✅ |
| **MySQL** | 只能用 `TIMESTAMP` / `DATETIME` | ❌（偏移丢失） |

MyBatis 3.5.0+ 已内置 `OffsetDateTimeTypeHandler`，可直接使用。

---

## 适用场景

✅ **跨时区 API 通信**：保留客户端原始偏移量，便于追溯用户提交时刻  
✅ **多端协作**：明确知道"这个时间是东八区 15:58"  
✅ **日志、审计**：精确记录事件与时区的关系  
✅ **前端表单**：用户预约、订单创建等带本地时区信息的场景  

---

## 陷阱与注意点

1. **相同偏移量 ≠ 同一时区**：`+08:00` 可表示北京、新加坡、珀斯（冬令时）等，不能与 `Asia/Shanghai` 划等号。
2. **不能处理夏令时**：北京无夏令时用 `OffsetDateTime` 即可；美国/欧洲必须用 `ZonedDateTime` + `ZoneId`，否则 `2025-03-09T02:30+01:00` 这种切换时刻无法表达。
4. **存储到 MySQL 时偏移量丢失**：只存绝对时间戳；如需回显用户原始偏移，需单独字段存储。
5. **避免用作数据库字段类型**：除非使用 PostgreSQL/Oracle，否则仍推荐 `Instant`。
6. **比较两个 OffsetDateTime**：`.equals()` 比较**绝对时刻**（会转 UTC），`isEqual()` 仅比较本地时间，需求不同选错会出错。


---
## 😀 默认助手



## 核心差异

```
OffsetDateTime  = LocalDateTime + ZoneOffset   （固定偏移，无规则）
ZonedDateTime    = LocalDateTime + ZoneId      （带地理/历史规则）
```

- `+08:00` 是一个**数字偏移量**
- `Asia/Shanghai` 是**地理时区规则集**（含历史变更、DST 等）

---

## 结构对比

| 维度 | `OffsetDateTime` | `ZonedDateTime` |
|---|---|---|
| 携带信息 | 固定偏移量 | ZoneId（含 DST 规则、历史偏移） |
| 实例 | `2026-09-23T15:58:02+08:00` | `2026-09-23T15:58:02 Asia/Shanghai` |
| JSON 兼容 | ✅ 标准 ISO-8601 | ❌ 需自定义格式 |
| DST 处理 | ❌ 不支持 | ✅ 自动处理 |
| 历史偏移 | ❌ 无法表达 | ✅ 可根据日期查表 |
| 反序列化要求 | 偏移量字符串 | 需 ZoneId |

---

## 关键差异：DST 夏令时

以纽约为例，每年 3 月第二个周日 02:00 → 03:00（跳过一小时），11 月反向。

```java
// 想表达 "纽约时间 2025-03-09 02:30"
// 这是一个不存在的时刻！Spring forward 时跳过 02:00-03:00

ZoneId ny = ZoneId.of("America/New_York");

ZonedDateTime zdt = ZonedDateTime.of(
    LocalDateTime.of(2025, 3, 9, 2, 30), ny);
// 抛出 ZoneRulesException：LocalDateTime 不存在于该时区

// OffsetDateTime 完全不报错——它根本不关心规则
OffsetDateTime odt = OffsetDateTime.of(
    2025, 3, 9, 2, 30, 0, 0, ZoneOffset.ofHours(-5));
// 正常创建，但语义是"伪造"的时间
```

**结论**：在 DST 切换区域，OffsetDateTime 会生成无效/模糊的时间。

---

## 关键差异：歧义时刻

DST 回拨时本地时间会重复出现两次：
- `2025-11-02T01:30:00 America/New_York` 既可能是 EDT 也可能是 EST

```java
ZonedDateTime zdt = ZonedDateTime.of(
    LocalDateTime.of(2025, 11, 2, 1, 30), ny);

// ZonedDateTime 提供处理策略：
zdt.withEarlierOffsetAtOverlap();  // EDT (-04:00)
zdt.withLaterOffsetAtOverlap();    // EST (-05:00)

// OffsetDateTime 无法表达这种歧义
```

---

## 关键差异：历史偏移

同一个 ZoneId 在不同历史时期偏移量可能不同。

```
俄罗斯 Europe/Moscow：
  2014-10-26 前：+04:00（冬令时）
  2014-10-26 起：+03:00（普京宣布永久夏令时）

ZonedDateTime  能根据日期自动推断：2010-01-01 → +03:00（旧规则当时是冬令时）
OffsetDateTime 只能手动指定，无法体现规则变化
```

中国 1949–1986 期间多次调整时区，ZoneId 也能追溯历史。

---

## 互相转换

```java
// OffsetDateTime → ZonedDateTime（需补 ZoneId）
OffsetDateTime odt = OffsetDateTime.now();
ZonedDateTime zdt = odt.atZoneSameInstant(ZoneId.of("Asia/Shanghai"));

// ZonedDateTime → OffsetDateTime（丢弃 ZoneId）
ZonedDateTime zdt = ZonedDateTime.now();
OffsetDateTime odt = zdt.toOffsetDateTime();
// 警告：丢失时区规则，仅剩瞬时偏移
```

---

## 选择建议

| 场景 | 推荐 |
|---|---|
| 跨系统 API 传输 | `Instant` |
| API 需保留客户端原始时区 | `OffsetDateTime` |
| 中国境内业务（无 DST） | `OffsetDateTime` + `+08:00` 足够 |
| 美国 / 欧洲 / 全球业务 | `ZonedDateTime` + `ZoneId` |
| 日志、审计、计算间隔 | `Instant` |
| 用户本地日历、会议安排 | `ZonedDateTime` |

---

## 陷阱

1. **`+08:00` ≠ `Asia/Shanghai`**：前者只是数字，后者含规则。用 `ZoneOffset.ofHours(8)` 创建的 OffsetDateTime 不能反向推断出城市。
2. **ZonedDateTime 不能完全 ISO 标准化**：JSON 里需特殊字段（如 `"zone":"Asia/Shanghai"`）配合 Jackson 自定义序列化。
3. **不要混用**：`OffsetDateTime` 和 `Instant` 转换容易写反方向，特别是 `withOffsetSameInstant` 与 `withOffsetSameLocal`：
   - `SameInstant` 改偏移量但**保留绝对时刻**（15:58+08:00 → 07:58+00:00）
   - `SameLocal` 改偏移量但**保留本地时间**（15:58+08:00 → 15:58+00:00）
4. **持久化注意**：数据库存 `OffsetDateTime` 时，MySQL 丢偏移，PostgreSQL 用 `timestamptz` 才行；存 `ZonedDateTime` 几乎所有数据库都丢 ZoneId，需单独字段保存。
