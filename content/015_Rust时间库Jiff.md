+++
title = "Rust优雅的时间库Jiff"
date = 2024-07-30
[taxonomies]
tags = ["Rust", "lib", "Jiff"]
+++

BurntSushi发现在目前的Rust时间库中, 存在一些"不足", 因而发布了新的时间库[Jiff](https://github.com/BurntSushi/jiff). 在06-24这一天, Jiff发布了0.1版本, 目前(07-29)已经是0.1.2版本, 最近也还在持续迭代中, 期望尽快达到1.0稳定状态. 在这一篇里我会介绍Jiff的优点是什么, 有哪些重要的数据结构, 常用的操作有哪些.

<!-- more -->

## 1 Jiff的优点

### 1.1 统一的时区类型

```txt
// 固定偏移格式
1970-01-01T02:30:00+01:30  3600
\________/ \______/ \___/
    |         |       |
   日期       时间   偏移(时区)

// POSIX格式
EST05:00EDT,M3.2.0/2:00:00,M11.1.0/2:00:00
\_/\___/\_/ \____________/ \_____________/
 |   |   |        |               |
时区 偏移 时区  夏令时开启时间   夏令时结束时间


// TZIF格式
2024-03-10T01:59:59-05:00[America/New_York]
\________/ \______/ \___/ \______________/
    |         |       |          |
   日期       时间    偏移      时区标记符
```

时区其实就是描述了时间戳和本地时间之间该怎么转换, 在前面的文章里列出过, 表达时区的方式至少有三种, 固定偏移, POSIX, TZIF. 

一方面, 这三种格式Jiff都提供了支持, 用`parse()`方法把字符串解析成时间; 另一方面, 在实现上, 用枚举类型屏蔽了差异, 使用者不需要了解内部的细节.

### 1.2 封装的时区数据库

```rust
pub struct TimeZoneDatabase {
    inner: Option<Arc<TimeZoneDatabaseInner>>,
}

struct TimeZoneDatabaseInner {
    zoneinfo: ZoneInfo,         // 系统自带的
    bundled: BundledZoneInfo,   // 预先内置的
}
```

时区数据库, Unix类系统通常会自带, 在`/usr/share/zoneinf`目录下, 而windows系统则不一定. 平台的差异性, 使得开发面临一个抉择, 是使用预置默认的呢, 还是使用系统自带的呢.

这两种方式有各自的优缺点, 使用系统自带的, 时区数据库在系统升级时就可以更新, 会更"新"一点. 而且当时区数据变更时, 不需要重新构建发布新的程序.

如果使用预置默认的, 则可以忽略系统的具体情况, 在所有平台上保持一致; 而且时区可以被预先定义成常量, 在下游代码中可以直接使用, 代码执行也会更有效率.

Jiff对这两种方式都提供了支持, 而且如果你不想因为这点琐事费心, 可以直接使用默认的行为, 即Unix类用系统自带的, windows系统用预置的.

如果你清楚些概念, 知道怎么做对程序更有利, 那也可以通过[features](https://docs.rs/crate/jiff/latest/features)指定策略. 比如在Unix类系统上, 也强制使用预置的时间数据库, 从而使程序在不同平台上更加一致.

### 1.3 一致的API风格

{{ image(src="/image/015_01.png", alt="image 404", position="center") }}

上图是我罗列的时间类型和他们的部分API, 可以看到, 这些类型的API在很大程度上保持了一致(不包含参数), 我们在熟悉了某个类型后, 很快可以找到"感觉", 迅速掌握其他的类型.

不仅API风格的尽量保持了一致, 方法的名称也尽量保持"见文知意". 比如`Date.tomorrow()`, 返回明天的日期; `Zoned.start_of_day()`, 返回当天的开始时间.

当然, 也提供了一些"啰嗦"的API, 名字稍微复杂点, 但方便使用. 比如`Date.days_in_year()`, 返回当月有多少天; `DateTime.first_of_month()`, 返回这当月的第1天对应的日期.

清晰有规律的API, 对库的使用者来说, 可以减轻心智负担, 更专注于自己的代码逻辑. 当然, 心智的负担其实是从使用者那, 转移到了开发者这, 这挺有Rust范的, 也是我认为Jiff优雅的一个原因. 

### 1.4 详尽的注释和示例

> I am not too far away from having more lines of docs in Jiff than I do lines of code. Kudos to the Temporal project for paving a path with excellent docs of their own.

几乎每个方法和类型上都附有注释说明, 阅读源代码的时候可以更容易理解作者的意图; 大部分公共方法都提供了使用示例, 查阅借鉴, 或者复制粘贴都很方便. 文档量几乎和代码相当了, 作者在x(推特)上发布的推文也提到了这一点. 

除了代码相关的文档, BurntSushi还写了Jiff的设计文档[DESIGN.md](https://github.com/BurntSushi/jiff/blob/master/DESIGN.md), 还有和等其他时间库(chrone, time)的比较文档[COMPARE.md](https://github.com/BurntSushi/jiff/blob/master/COMPARE.md).

详细的文档能带来几个好处, 一是使用者能更快掌握库的使用, 二是开发者能有更多理解上的共识, 三是给后来的库(或者其他语言的库)提供借鉴. Jiff本身也是站在了"巨人的肩膀"上, 从[temporal](https://tc39.es/proposal-temporal/docs/), [java.time](https://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html)都有借鉴.

## 2. 主要的数据类型

### 2.1 时间戳Timestamp

```rust
pub struct Timestamp {
    second: UnixSeconds,
    nanosecond: FractionalNanosecond,
}
```
时间戳记录了从`Epoch`以来经过的秒数, `Timestamp`属于"绝对"时间, 可以对应到现实时间中的某个具体时刻. `Timestamp`也可以是"负数", 代表`Epoch`之前的时刻.

`now()`方法可以获取当前的时间, 底层是调用的是操作系统提供的获取当前时间的方法, 在Linux系统上是`clock_gettime`方法. 和`Instant`不同的是, `Timestamp`不保证单调, 在时钟回调的情况下会有所区别.

### 2.2 时区TimeZone

```rust
pub struct TimeZone {
    kind: Option<Arc<TimeZoneKind>>,
}

enum TimeZoneKind {
    Fixed(TimeZoneFixed),
    Posix(TimeZonePosix),
    TZIF(TimeZoneTZIF),
}
```

时区描述了时间戳和本地时间之间的转换规则, Jiff支持固定偏移, POSIX和TZIF三种格式的时区, 统一用枚举类型`TimeZone`表示.

因为TZIF格式包含的是不确定的规则, 所以是用`Box`存在堆上的, 所以`TimZone`没有实现`Copy`, 所以在所有权转移的场景下需要注意.

### 2.3 带时区的时间Zoned

```rust
pub struct Zoned {
    inner: ZonedInner,
}

struct ZonedInner {
    timestamp: Timestamp,
    datetime: DateTime,
    offset: Offset,
    time_zone: TimeZone,
}
```

`Zoned`的主要部分是是时间戳和时区, 一个定义了"绝对"时间, 一个定义了时间转换的规则. `Zoned`即可以表示"绝对"时间, 对应到具体的时刻; 也可以转换成日常时间, 方便我们使用.

由于包含了时区信息, 所以`Zoned`可以也会自动处理夏令时. `Zoned`支持大小的比较, 只比较时间戳的大小, 日期上的数字和时区不影响结果.

### 2.4 日常时间Time, Date, DateTime

```rust
pub struct Time {
    hour: Hour,
    minute: Minute,
    second: Second,
    subsec_nanosecond: SubsecNanosecond,
}
```

`Time`表示钟表时间, 时分秒, 当然这边还包含了更高的精度纳秒. `Time`的取值范围是`00:00:00.000_000_000`到`23:59:59.999_999_999`.

`Time`的1小时总是有60分钟(在夏令时规则下, 1小时并不总是60分钟), 而1分钟总是有60秒, 也就不会有`07:59:60`这样的闰秒时间.

```rust
pub struct Date {
    year: Year,
    month: Month,
    day: Day,
}
```

`Date`表示日历时间, 年月日, `year`的取值范围是`-9999 ~ 9999`, `month`的取值范围是`1 ~ 12`, `day`的取值范围是`1 ~ 31`.

因为`Date`要保证总是有效, 所以构造的时候会做检查, 比如月份是6的话, 日就不能是31; 如果月份是2的话, 只有闰年, 日才能是29.     

```rust
pub struct DateTime {
    date: Date,
    time: Time,
}
```

`DateTime`就是日期加上时间, 年月日+时分秒, 所以`DateTime`的取值范围也遵循`Date`+`Time`的范围.

### 2.5 时间跨度Span

```rust
pub struct Span {
    sign: Sign,
    years: t::SpanYears,
    months: t::SpanMonths,
    days: t::SpanDays,
    hours: t::SpanHours,
    minutes: t::SpanMinutes,
    seconds: t::SpanSeconds,
    ...
}

let a = Span::new()
  .days(1)
  .hours(12)
  .minutes(65)
  .seconds(7200);
// a = P1dT12h65m7200s 1天12小时65分7200秒
```

`Span`表示两个时间之间的跨度, 不同的时间单位, 都单独记录了值(不进位换算). `until`, `since`方法的返回值, 还有`*_add`, `*_sub`方法的入参等, 日期之间的数学运算用的都是`Span`类型.

标准库里也有`Duration`表示两个时刻之间的跨度, `Span`与之不同的地方在于, 一`Span`有正负, 时间跨度有方向, 可以表示"之后, 也可以表示"之前"; 二保留了不同时间单位的值, 没有全部换算到秒(纳秒), 所以可以处理"不规则"的时间, 比如只有23小时的天.

## 3. 时间类型之间的转换

得益于多种时区格式被统一了, 所以时间类型之间的转换也很轻松, 只需要记住, 时区是时间戳和日常时间的"转接头"就行了.

时间戳在时区化后, 才能转换成日常时间; 而日常时间在时区化后, 才能转换成时间戳; 带时区的时间, 既能转换成时间戳, 也能转换成日常时间, 当然也带有时区信息.

### 3.1 Timestamp

```rust
Timestamp.intz(&str) -> Result<Zoned>
Timestamp.to_zoned(TimeZone) -> Zoned
```

`Timestamp`+`TimeZone`=`Zoned`, 时间戳+时区=带时区的时间.


### 3.2 Time, Date, DateTime

```rust
Time.on(Y, M, D) -> DateTime
Time.to_datetime(Date) -> DateTime
```

`Time`+`Date`=`DateTime`, 时间+日期=日常时间.


```rust
Date.at(H, M, S, NS) -> DateTime
Date.intz(&str) -> Result<Zoned>
Date.to_zoned(TimeZone) -> Zoned
```

`Date`+`Time`=`DateTime`, 时间+日期=日常时间.

`Date`+`TimeZone`=`Zoned`, 日期+时区=带时区的时间

```rust
DateTime.date() -> Date
DateTime.time() -> Time
DateTime.intz(&str) -> Result<Zoned>
DateTime.to_zoned(TimeZone) -> Zoned
```

`DateTime`+`TimeZone`=`Zoned`, 日常时间+时区=带时区的时间

### 3.3 Zoned
 
```rust
Zoned.date() -> Date
Zoned.time() -> Time
Zoned.datetime() -> DateTime
Zoned.intz(&str) -> Result<Zoned>
Zoned.timestamp() -> TimeStamp
Zoned.time_zone() -> &TimeZone
```

带时区的时间`Zoned`是集大成者, 包含了`Date`, `Time`, `DateTiem`, `Timestamp`, `TimeZone`.

##  4. 辅助操作类型

{{ image(src="/image/015_02.png", alt="image 404", position="center") }}

### 4.1 时间跨度Difference

```rust
let a = date(2024, 1, 1).at(0, 0, 0, 0);
let b = date(2024, 1, 1).at(1, 22, 3, 0);
let c = a.until(
  DateTimeDifference::new(b)
    .smallest(Unit::Minute)       // 最小单位是分, 秒会被舍掉
    .mode(RoundMode::HalfExpand)  // 取舍的策略是HalfExpand
    .increment(6)                 // 最终结果是6的倍数, 22更接近24
).unwrap();
// c = PT1h24m
```

`*Difference`类型主要用来计算时间跨度, 比如在`since`和`until`方法里. `*Difference`为计算时间跨度提供了配置, 有三个方面.

一是指定最大和最小的单位, 比如1小时, 既可以用单位"小时"表示, 也能用单位"分"或"秒"表示, 我们可以限制到底用什么单位.

二是怎么处理"小数", 比如我们使用的最小单位是分, 那秒级开始就是"小数". 我们可以设置取舍的规则, 是"四舍五入", 还是直接丢弃, 还是其他什么策略.

三是设置结果的幅度, 比如设置成n的话, 就会把结果与n * m和n * (m+1)两个数比较, 然后取更"接        近"的那个做为最终结果.

### 4.2 Round

```rust
let a = date(2024, 1, 1).at(1, 2, 35, 0);
let b = a.round(
  DateTimeRound::new()
    .smallest(Unit::Minute)   // 最小单位是分
    .mode(RoundMode::Trunc)   // 直接丢弃策略
).unwrap();
// b = 2024-01-01T01:02:00
```

`*Round`类型用来对时间做取舍操作, 我们可以指定最小的保留单位, 然后设置取舍的规则, 同样也可以设置结果的幅度是什么.

### 4.3 With

```rust
let a = date(2024, 1, 1).at(0, 0, 0, 0);
let b = a.with()
  .hour(1)
  .minute(2)
  .build().unwrap();
```

`*With`类型用来修改时间的某些字段, 这里需要注意的是, `with(self)`方法的入参是值类型.

### 4.4 Series

```rust
let a = date(2024, 1, 1).at(0, 0, 0, 0);
let mut b = a.series(Span::new().days(1));  // 跨度为1天
// b.next = Some(2024-01-01T00:00:00)
// b.next = Some(2024-01-02T00:00:00)
```

`*Series`类型用来生成一系列的的时间, 我们可以指定时间的跨度, `series(self, ...)`方法的入参也是值类型.  

## 5. 小结

这篇文章简述了Jiff时间库的一些优点, 以及库里主要的数据结构和操作方法. 因为篇幅的原因, 具体的使用示例, 还有其他一些值得探讨的问题, 会放到后续的文章中, 感谢!


## 6. 更新日志

这边只罗列了相对来说比较重要的, 比如和Rust生态集成, 严重BUG修复, 新增平台支持等, 完整的更新日志在

### 0.1.1 (2024-07-26)

+ 支持标准库的`Duration`和`Span`互相转换(try_from).

### 0.1.3 (2024-07-30)

+ 支持`wasm{32, 64}-unknown-unknown`, 需要开启`js`特性


</br>

*-> 如果文章有不足之处或者有改进的建议，可以在[这边](https://github.com/dlzht/dlzht.github.io/discussions/12)告诉我，也可以发送给我的[邮箱](mailto:dlzht@protonmail.com)*