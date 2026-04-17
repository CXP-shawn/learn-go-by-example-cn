# 第 55 关：时间戳（师兄带你学 Go）

🎯 这一关你会学到
• 啥是 Unix 时间戳（epoch）：从 1970-01-01 UTC 开始的秒数
• time.Time 上三个方法：Unix() 秒、UnixNano() 纳秒、怎么自己算毫秒
• 反向构造：time.Unix(sec, nsec) 从数字恢复成 Time
• 生产里啥场景用秒、啥场景用毫秒、啥场景用纳秒

🤔 先想一个问题

你打开 12306 订票，想"2026-05-01 09:00:00 开抢"——这个时刻到底怎么存到服务器的数据库里？JSON 传输 API 里的"过期时间"怎么表示最省地方？答案是 **Unix 时间戳（Unix Timestamp，也叫 Unix epoch time）**——一个单一的整数（int64），从 **1970-01-01 00:00:00 UTC** 那一刻开始经过的秒数（或毫秒、纳秒）。整数比字符串省、比较快、跨时区没歧义。这一关就讲 Go 怎么在 time.Time 和 Unix 时间戳之间来回转。

## 📖 看代码

```go
// 一般程序会有获取 [Unix 时间](http://zh.wikipedia.org/wiki/UNIX%E6%97%B6%E9%97%B4)
// 的秒数，毫秒数，或者微秒数的需求。来看看如何用 Go 来实现。

package main

import (
	"fmt"
	"time"
)

func main() {

	// 分别使用 `time.Now` 的 `Unix` 和 `UnixNano`，
	// 来获取从 Unix 纪元起，到现在经过的秒数和纳秒数。
	now := time.Now()
	secs := now.Unix()
	nanos := now.UnixNano()
	fmt.Println(now)

	// 注意 `UnixMillis` 是不存在的，所以要得到毫秒数的话，
	// 你需要手动的从纳秒转化一下。
	millis := nanos / 1000000
	fmt.Println(secs)
	fmt.Println(millis)
	fmt.Println(nanos)

	// 你也可以将 Unix 纪元起的整数秒或者纳秒转化到相应的时间。
	fmt.Println(time.Unix(secs, 0))
	fmt.Println(time.Unix(0, nanos))
}
```

🔍 师兄给你逐行拆

**先说"Unix 纪元"是个啥**

**Unix 纪元（Unix Epoch）**的定义是 `1970-01-01 00:00:00 UTC`——这个时刻被 Unix 系统选作"**0 点**"。此后每一秒+1，到现在 2026 年大约是 **17.7 亿秒**（17.7 × 10^8）左右。为啥选 1970？因为 Unix 系统就是 1970 年代在贝尔实验室诞生的，设计师懒得定义更早的时间。这个"数字代表时刻"的约定成了整个互联网的基础。

注意几个单位：
- **秒（Unix time / epoch seconds）**：最常见，int64 能撑到公元 2925 年（过那之前咱们都早转世了）
- **毫秒（milliseconds）**：秒 × 1000。JavaScript 的 Date.now() 就是毫秒
- **微秒（microseconds）**：秒 × 10^6
- **纳秒（nanoseconds）**：秒 × 10^9，Go 内部存储时间用这个精度

**time.Now() + Unix()**

```go
now := time.Now()
secs := now.Unix()
nanos := now.UnixNano()
fmt.Println(now)
```

`now.Unix()` 返回 int64，是 Unix 秒时间戳。比如 2026-04-17 跑这段代码，secs 会是类似 `1771382523` 这种 17 亿左右的大整数。

`now.UnixNano()` 返回 int64，Unix 纳秒时间戳。大概是 `1771382523123456789` 这种 19 位数字。

第一行 `fmt.Println(now)` 先把 now 按人类可读格式打一下，做对照。

**算毫秒：自己除**

```go
millis := nanos / 1000000
fmt.Println(secs)   // 秒
fmt.Println(millis) // 毫秒
fmt.Println(nanos)  // 纳秒
```

Go 当初设计时故意没给 UnixMilli() 方法（标准库里秒和纳秒就够两极了）——所以想拿毫秒，得自己从纳秒除 1_000_000。**小彩蛋**：从 **Go 1.17** 开始标准库终于加了 `now.UnixMilli()` 和 `now.UnixMicro()` 方法（以及对应的 time.UnixMilli(m) / time.UnixMicro(u) 反向构造），不用再自己除了。本关源码是老版本写的所以手动除。

**反向构造：time.Unix**

```go
fmt.Println(time.Unix(secs, 0))
fmt.Println(time.Unix(0, nanos))
```

`time.Unix(sec, nsec)` 是反函数——从"秒 + 纳秒"这两个 int64 参数构造一个 Time。**第二个参数会被加到第一个参数上当作偏移**，所以：

- `time.Unix(secs, 0)` = 整秒部分，毫微秒 0
- `time.Unix(0, nanos)` = 秒给 0，所有信息都在 nanos 里（runtime 会自动归位）

这俩其实**得到的是同一个时刻**（因为 nanos = secs × 10^9 + 当前秒内的纳秒偏移）——就是演示两种构造方式都行。

返回的 Time 默认用本地时区（time.Local）。要 UTC 就 `time.Unix(secs, 0).UTC()`。

**为啥这个 API 对接口开发很重要？**

- 和数据库交互：MySQL 的 INT UNSIGNED 存 Unix 秒最省空间
- 和前端交互：JSON 里传数字比传字符串更省流量、不用解析日期格式
- 跨时区：Unix 时间戳天然不带时区，前端拿到后用本地时区展示
- 日志排序：日志前缀打时间戳，按字母序排序就是时间序
- 签名/token 过期时间：JWT 的 exp 字段就是 Unix 秒

🏃 跑一下试试

保存为 epoch.go，然后：

```bash
go run epoch.go
```

预期输出大致如下（数字会变，因为取的是"现在"）：

```text
2026-04-17 15:04:05.123456789 +0800 CST
1776474245
1776474245123
1776474245123456789
2026-04-17 15:04:05 +0800 CST
2026-04-17 15:04:05.123456789 +0800 CST
```

逐行看：
- 第 1 行：now 的默认格式
- 第 2 行：Unix 秒（10 位数）
- 第 3 行：Unix 毫秒（13 位数）
- 第 4 行：Unix 纳秒（19 位数）
- 第 5 行：`time.Unix(secs, 0)` 恢复的时间，**精度只到秒**所以没有纳秒部分
- 第 6 行：`time.Unix(0, nanos)` 恢复的时间，**带完整纳秒精度**，和第 1 行基本一致

💡 师兄的碎碎念

时间戳实战小贴士：

1. **单位统一是第一铁律**。前后端、各微服务间要约定死：用秒还是毫秒？混用会出大事。师兄见过线上 bug："订单过期时间用毫秒，但某个服务当秒处理，结果用户等了 1000 倍长的时间才自动退款"。

2. **Go 1.17+ 有 UnixMilli / UnixMicro**：`now.UnixMilli()` 代替 `now.UnixNano() / 1e6`。以后别手动除了。

3. **毫秒 vs 秒怎么判断**：13 位数字≈毫秒、10 位≈秒、19 位≈纳秒。不对就是单位搞错了。

4. **JavaScript 用毫秒**：前端代码 `Date.now()` 返回毫秒时间戳、`new Date(ms)` 也按毫秒构造。Go 后端往前端传 **0 点这种场景要 × 1000**。

5. **负数时间戳表示 1970 之前**：`time.Unix(-1, 0)` 是 1969-12-31 23:59:59 UTC。少见但确实能表示。

6. **2038 年问题**：32 位 int 存 Unix 秒会在 2038-01-19 溢出（著名的 **Y2K38 bug**）。现在大家都用 int64 不用担心，但老 C 代码里有隐患。

7. **time.Since(t)** 的底层实现就是 `time.Now().Sub(t)`——用时间戳算时间差比自己减纳秒更健壮（考虑到单调钟）。

吐槽一句：Unix 时间戳的单位混乱是全人类共通的噩梦，"到底这个 int 是秒还是毫秒"的讨论在每个团队都发生过。看字面长度 10/13/19 判断是民间约定俗成的土办法。

🎓 知识点清单

- Unix 时间戳 = 从 1970-01-01 UTC 起的秒/毫秒/纳秒
- time.Time 的 Unix() 拿秒、UnixNano() 拿纳秒
- Go 1.17+ 新增 UnixMilli() 和 UnixMicro()，别再手动除
- 反向构造用 time.Unix(sec, nsec)
- 数据库、API、日志里存时间戳整数远优于字符串
- 单位口径一定要团队统一（秒或毫秒）
- 位数速查：10 秒 / 13 毫秒 / 19 纳秒
- JavaScript 是毫秒，跟 Go 后端对接要转

🎉 第三波 13 关的终章

师兄带你从第 43 关的 sorting 排到了第 55 关的 epoch，中间横跨 panic/defer/recover 三位一体、字符串处理、格式化、模板、正则、JSON/XML、时间和时间戳——Go 日常开发里最常用的那一堆工具包全见了个遍。下面还有并发相关的进阶（有其他师兄负责）。先给自己加个鸡腿，庆祝打完这 13 关！继续冲！💪
