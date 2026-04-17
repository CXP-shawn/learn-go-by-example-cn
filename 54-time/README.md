# 第 54 关：时间（师兄带你学 Go）

🎯 这一关你会学到
• Go 的 time 包：time.Now / time.Date 构造时间
• 从 Time 里抽取年月日时分秒毫秒、取星期几
• 时间大小比较：Before / After / Equal
• Duration（时间段）怎么表示，和 Sub / Add 怎么搭档

🤔 先想一个问题

你上班打卡，系统要算"这个月一共加班几小时"、"下一次考核在 14 天后的几点"、"海外会议对应北京时间几点"——这些都离不开**时间运算**。Go 的 time 包把时间和时间段分成两种类型（time.Time 和 time.Duration）分别处理，设计干净。这一关咱们把时间基本操作全过一遍。

## 📖 看代码

```go
// Go 为时间（time）和时间段（duration）提供了大量的支持；这儿有是一些例子。

package main

import (
	"fmt"
	"time"
)

func main() {
	p := fmt.Println

	// 从获取当前时间时间开始。
	now := time.Now()
	p(now)

	// 通过提供年月日等信息，你可以构建一个 `time`。
	// 时间总是与 `Location` 有关，也就是时区。
	then := time.Date(
		2009, 11, 17, 20, 34, 58, 651387237, time.UTC)
	p(then)

	// 你可以提取出时间的各个组成部分。
	p(then.Year())
	p(then.Month())
	p(then.Day())
	p(then.Hour())
	p(then.Minute())
	p(then.Second())
	p(then.Nanosecond())
	p(then.Location())

	// 支持通过 `Weekday` 输出星期一到星期日。
	p(then.Weekday())

	// 这些方法用来比较两个时间，分别测试一下是否为之前、之后或者是同一时刻，精确到秒。
	p(then.Before(now))
	p(then.After(now))
	p(then.Equal(now))

	// 方法 `Sub` 返回一个 `Duration` 来表示两个时间点的间隔时间。
	diff := now.Sub(then)
	p(diff)

	// 我们可以用各种单位来表示时间段的长度。
	p(diff.Hours())
	p(diff.Minutes())
	p(diff.Seconds())
	p(diff.Nanoseconds())

	// 你可以使用 `Add` 将时间后移一个时间段，或者使用一个 `-` 来将时间前移一个时间段。
	p(then.Add(diff))
	p(then.Add(-diff))
}
```

🔍 师兄给你逐行拆

**两种核心类型：time.Time 和 time.Duration**

讲代码前师兄先把两个类型说清楚：
- **time.Time**：表示**某个具体时刻**（例如 "2024-03-15 10:30:00"）。它带时区信息，精度到纳秒
- **time.Duration**：表示**一段时长**（例如 "3 小时 25 分钟"）。底层是 int64 类型的纳秒数

两者组合能覆盖绝大多数时间需求——Time 相减得 Duration、Time + Duration 得新 Time。

**p := fmt.Println**

```go
p := fmt.Println
```

老朋友了——给 fmt.Println 起个短名，后面调用省事。本关要 print 一大堆东西，p() 一顿操作猛如虎。

**now := time.Now()**

```go
now := time.Now()
p(now)
```

`time.Now()` 返回当前时间（本地时区）。打印的 Time 默认格式像 `2026-04-17 15:04:05.123456789 +0800 CST`——包含日期、时间、时区偏移、时区名。

**time.Date：构造一个具体时间**

```go
then := time.Date(
    2009, 11, 17, 20, 34, 58, 651387237, time.UTC)
```

`time.Date(year, month, day, hour, min, sec, nsec, loc)` 手动构造一个 Time。参数意思分别是：年、月、日、时、分、秒、纳秒、时区。

- **month 是 time.Month 类型**，不是普通 int。你可以直接传 11 自动转换，也可以写 `time.November`。
- **loc 是 *time.Location**，表示时区。time.UTC 是世界协调时，time.Local 是操作系统本地时区，也可以 `time.LoadLocation("Asia/Shanghai")` 加载指定时区。

这里 then 就是 UTC 时间 2009-11-17 20:34:58.651387237。

**提取时间组成部分**

```go
p(then.Year())         // 2009
p(then.Month())        // November
p(then.Day())          // 17
p(then.Hour())         // 20
p(then.Minute())       // 34
p(then.Second())       // 58
p(then.Nanosecond())   // 651387237
p(then.Location())     // UTC
```

每一个方法返回对应的字段。注意 `Month()` 返回的是 time.Month 枚举（实际 Println 会打英文月份名）；Day() 是 int；Location() 返回 *time.Location。

**Weekday：星期几**

```go
p(then.Weekday())  // Tuesday
```

`Weekday()` 返回 time.Weekday 枚举，打印英文。实战里可用来根据周内日算班次、做定时任务过滤。

**Before / After / Equal：时间比较**

```go
p(then.Before(now))  // true（then 在 now 之前）
p(then.After(now))   // false
p(then.Equal(now))   // false
```

两个 Time 比大小就用这三个方法。**一定要用 Equal 而不是 `==`**，因为两个 Time 可能"同一时刻但时区不同"——== 会说不相等，Equal 会说相等。师兄吃过这个亏。

**Sub：计算时间差**

```go
diff := now.Sub(then)
p(diff)
```

`a.Sub(b)` 返回 Duration，表示 a - b 是多少。diff 这里是"从 then 到 now 过去了多久"。Duration 默认打印格式是 `16y3m15d...` 那种人类可读形式。

**Duration 单位换算**

```go
p(diff.Hours())         // float64
p(diff.Minutes())       // float64
p(diff.Seconds())       // float64
p(diff.Nanoseconds())   // int64
```

Duration 上挂了各种单位的换算方法：
- Hours()、Minutes()、Seconds() 返回 **float64**（带小数）
- Nanoseconds()、Milliseconds()、Microseconds() 返回 **int64**（整数）

想加减时间段有对应的常量：`time.Hour`、`time.Minute`、`time.Second`、`time.Millisecond`、`time.Microsecond`、`time.Nanosecond`。比如 `3 * time.Hour` 就是 3 小时。

**Add：时间加减**

```go
p(then.Add(diff))   // then 往后加 diff —— 结果约等于 now
p(then.Add(-diff))  // then 往前倒 diff —— 结果是 then 之前 diff 长度的时刻
```

`t.Add(d)` 在 t 的基础上加一个 Duration 返回新 Time。负的 Duration 就是往前倒——**Go 里 Duration 可以是负数**，这点和直觉相符。

没有单独的 Subtract 方法处理 Time + Duration，就用 `Add(-d)`。但 Time - Time 的场景用前面的 Sub 方法。

**小彩蛋：time.Parse / time.Format**

本关没演示，但实战必备。time.Format(layout) 把 Time 转字符串，time.Parse(layout, str) 反过来。**layout 不是 yyyy-MM-dd，而是 Go 独有的 "2006-01-02 15:04:05"**——这个参考时间是记忆 "01/02 03:04:05PM '06 -0700"，对应 1月2日3点4分5秒第06年第7时区。第一次看蒙圈很正常。

🏃 跑一下试试

保存为 time.go，然后：

```bash
go run time.go
```

预期输出大致如下（前几行包含你本机当前时间和时区）：

```text
2026-04-17 15:04:05.123456789 +0800 CST
2009-11-17 20:34:58.651387237 +0000 UTC
2009
November
17
20
34
58
651387237
UTC
Tuesday
true
false
false
<diff>         ← 从 2009-11-17 到 now 的时间段，类似 "142200h37m06.472069552s"
<hours float>
<minutes float>
<seconds float>
<nanos int64>
<now 时刻> ← then+diff
<2009 之前对应时刻> ← then-diff
```

注意前两行里具体 now 的值**取决于你运行时的时间和时区**，每次运行都不同。Duration 输出形如 `141936h29m1.472069552s` 这种"h+m+s"格式。

💡 师兄的碎碎念

time 包的生存手册：

1. **比较时间永远用 Before/After/Equal 不用 ==**。== 会比较所有字段包括时区指针，两个"同一时刻但来源不同的" Time 很可能 == 为 false。

2. **时区坑王**。Go 的 time 默认用本地时区，线上服务放在哪个机房就会是哪里。想要稳：要么所有 Time 存储时都转 UTC（`t.UTC()`），要么明确 LoadLocation。**永远记录时区，别假设**。

3. **time.Parse 的 layout 是 "2006-01-02 15:04:05"**——这是 Go 的独门写法，不是 yyyy-MM-dd。Go 1.20+ 开始提供 time.DateTime / time.DateOnly / time.TimeOnly 常量，直接用这些常量省得记。

4. **Duration 的类型陷阱**：`10 * time.Second` 是正确写法，但 `secondsVariable * time.Second` **会报错**——time.Duration 是 int64 的命名类型，int 和它乘要显式转换 `time.Duration(n) * time.Second`。

5. **time.Sleep(d)** 用来让当前 goroutine 睡一会儿，在定时任务、重试退避里常用。

6. **time.Ticker** 是定时触发，`time.NewTicker(5 * time.Second)` 每 5 秒来一次。和 for-range 配合写定时任务很爽。

7. **time.Since(t)** 是 `time.Now().Sub(t)` 的语法糖，做性能测量最方便。

吐槽一下 time 包的 layout 写法：第一次见 "2006-01-02 15:04:05" 真的想骂人。但 Go 设计者说这个日期 01/02 03:04:05PM '06 -0700 对英文用户来说有口诀可以记，咱们东亚用户靠查文档得了。

🎓 知识点清单

- time.Time 表示时刻，time.Duration 表示时长
- time.Now() 取当前、time.Date(...) 手动构造
- Year / Month / Day / Hour / Minute / Second / Nanosecond / Location / Weekday 抽字段
- Before / After / Equal 比大小（别用 ==）
- Sub 算时间差返回 Duration、Add 给 Time 加 Duration
- Hours / Minutes / Seconds 返 float64，Milliseconds / Nanoseconds 返 int64
- 时区三件套：UTC / Local / LoadLocation("Asia/Shanghai")
- 时间格式化用 layout "2006-01-02 15:04:05"（Go 独门）

➡️ 下一关

处理时间常常还要和**时间戳**（Unix 秒数）打交道——log 文件、数据库时间戳字段、API 接口参数都用它。下一关 55「时间戳 epoch」带你搞定 Time 和 Unix 时间戳之间的转换，走起！
