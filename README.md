# 🎓 Go by Example · 师兄带你学

> 一份写给纯小白的 Go 语言入门教程。改编自 [Go by Example 中文版](https://gobyexample-cn.github.io/)，**79 关全部用「师兄带你学」的风格原创重写讲解** 🎉

## 🍜 这是个什么项目？

如果你是编程小白，或者从别的语言转过来想快速上手 Go，这个仓库就是为你准备的。

咱们把 Go by Example 的 **79 个经典小案例**，每一个都像学长坐在你旁边一样重新讲了一遍 —— 带生活化类比、带踩坑故事、带吐槽感，但也保住了专业性。只要你跟着一关一关走，就能从 `Hello World` 一路学到处理信号、进程退出的系统编程。

**每一关都包含：**
- 📄 完整可运行的 `.go` 源码（原汁原味，包含原中文注释）
- 🎯 一句话说清这一关解决什么问题
- 🤔 一个生活化的小问题引入概念（外卖、宿舍、快递柜、奶茶店……）
- 📖 看代码（完整源码一字不改）
- 🔍 师兄给你逐行拆代码（核心讲解）
- 🏃 跑一下试试（命令 + 预期输出）
- 💡 师兄的碎碎念（踩坑提醒 + 冷知识）
- 🎓 知识点清单（面试 / 复习用）
- ➡️ 下一关（顺着链接一路学下去）

## 🗺️ 79 关学习路线

### 🌱 第一章 · 入门基础（第 01–16 关）

从 Hello World 开始，搞定 Go 的基本语法、控制结构、容器类型和函数基础。

- [第 01 关 · Hello World](./01-hello-world/)
- [第 02 关 · Values](./02-values/)
- [第 03 关 · Variables](./03-variables/)
- [第 04 关 · Constants](./04-constants/)
- [第 05 关 · For](./05-for/)
- [第 06 关 · If/Else 分支](./06-if-else/)
- [第 07 关 · Switch 分支结构](./07-switch/)
- [第 08 关 · 数组](./08-arrays/)
- [第 09 关 · 切片](./09-slices/)
- [第 10 关 · Map](./10-maps/)
- [第 11 关 · range 遍历](./11-range/)
- [第 12 关 · 函数](./12-functions/)
- [第 13 关 · 多返回值](./13-multiple-return-values/)
- [第 14 关 · 变参函数](./14-variadic-functions/)
- [第 15 关 · 闭包](./15-closures/)
- [第 16 关 · 递归](./16-recursion/)

### 🧱 第二章 · 数据结构与抽象（第 17–24 关）

指针、字符串、结构体、方法、接口、嵌入、泛型、错误——Go 类型系统的骨架。

- [第 17 关 · 指针](./17-pointers/)
- [第 18 关 · 字符串和 rune 类型](./18-strings-and-runes/)
- [第 19 关 · 结构体](./19-structs/)
- [第 20 关 · 方法](./20-methods/)
- [第 21 关 · 接口](./21-interfaces/)
- [第 22 关 · Embedding](./22-embedding/)
- [第 23 关 · 泛型](./23-generics/)
- [第 24 关 · 错误处理](./24-errors/)

### ⚡ 第三章 · 并发编程（第 25–42 关）

goroutine、channel、select、定时器、worker pool、WaitGroup、互斥锁——Go 最出名的并发武器库。

- [第 25 关 · 协程](./25-goroutines/)
- [第 26 关 · 通道](./26-channels/)
- [第 27 关 · 通道缓冲](./27-channel-buffering/)
- [第 28 关 · 通道同步](./28-channel-synchronization/)
- [第 29 关 · 通道方向](./29-channel-directions/)
- [第 30 关 · 通道选择器](./30-select/)
- [第 31 关 · 超时处理](./31-timeouts/)
- [第 32 关 · 非阻塞通道操作](./32-non-blocking-channel-operations/)
- [第 33 关 · 通道的关闭](./33-closing-channels/)
- [第 34 关 · 通道遍历](./34-range-over-channels/)
- [第 35 关 · Timer](./35-timers/)
- [第 36 关 · Ticker](./36-tickers/)
- [第 37 关 · 工作池](./37-worker-pools/)
- [第 38 关 · WaitGroup](./38-waitgroups/)
- [第 39 关 · 速率限制](./39-rate-limiting/)
- [第 40 关 · 原子计数器](./40-atomic-counters/)
- [第 41 关 · 互斥锁](./41-mutexes/)
- [第 42 关 · 状态协程](./42-stateful-goroutines/)

### 🧪 第四章 · 标准库实战（第 43–72 关）

排序、panic/defer/recover、字符串/正则、JSON/XML、时间、散列、文件、命令行等 Go 日常开发离不开的 stdlib。

- [第 43 关 · 排序](./43-sorting/)
- [第 44 关 · 使用函数自定义排序](./44-sorting-by-functions/)
- [第 45 关 · Panic](./45-panic/)
- [第 46 关 · Defer](./46-defer/)
- [第 47 关 · Recover](./47-recover/)
- [第 48 关 · 字符串函数](./48-string-functions/)
- [第 49 关 · 字符串格式化](./49-string-formatting/)
- [第 50 关 · 文本模板](./50-text-templates/)
- [第 51 关 · 正则表达式](./51-regular-expressions/)
- [第 52 关 · JSON](./52-json/)
- [第 53 关 · XML](./53-xml/)
- [第 54 关 · 时间](./54-time/)
- [第 55 关 · 时间戳](./55-epoch/)
- [第 56 关 · 时间的格式化和解析](./56-time-formatting-parsing/)
- [第 57 关 · 随机数](./57-random-numbers/)
- [第 58 关 · 数字解析](./58-number-parsing/)
- [第 59 关 · URL 解析](./59-url-parsing/)
- [第 60 关 · SHA256 散列](./60-sha256-hashes/)
- [第 61 关 · Base64 编码](./61-base64-encoding/)
- [第 62 关 · 读文件](./62-reading-files/)
- [第 63 关 · 写文件](./63-writing-files/)
- [第 64 关 · 行过滤器](./64-line-filters/)
- [第 65 关 · 文件路径](./65-file-paths/)
- [第 66 关 · 目录](./66-directories/)
- [第 67 关 · 临时文件和目录](./67-temporary-files-and-directories/)
- [第 68 关 · 单元测试和基准测试](./68-testing-and-benchmarking/)
- [第 69 关 · 命令行参数](./69-command-line-arguments/)
- [第 70 关 · 命令行标志](./70-command-line-flags/)
- [第 71 关 · 命令行子命令](./71-command-line-subcommands/)
- [第 72 关 · 环境变量](./72-environment-variables/)

### 🖥️ 第五章 · 系统与网络编程（第 73–79 关）

HTTP 客户端与服务端、Context、进程、信号、退出码——把 Go 跑成一个真正的系统级程序。

- [第 73 关 · HTTP 客户端](./73-http-clients/)
- [第 74 关 · HTTP 服务端](./74-http-servers/)
- [第 75 关 · Context](./75-context/)
- [第 76 关 · 生成进程](./76-spawning-processes/)
- [第 77 关 · 执行进程](./77-execing-processes/)
- [第 78 关 · 信号](./78-signals/)
- [第 79 关 · 退出](./79-exit/)

## 🚀 怎么用这个仓库

1. **装 Go**：前往 [go.dev/dl](https://go.dev/dl/)，或用包管理器（`brew install go` / `apt install golang-go`）。
2. **Clone 仓库**：
   ```bash
   git clone https://github.com/CXP-shawn/learn-go-by-example-cn.git
   cd learn-go-by-example-cn
   ```
3. **进任意一关的文件夹跑代码**：
   ```bash
   cd 01-hello-world
   go run hello-world.go
   ```
4. **打开同目录的 `README.md`**，跟着师兄的节奏学。

### 📌 推荐学习节奏

- **一天 1–3 关**，一个月内能通关；
- **别囫囵吞枣**——每关的"💡 师兄的碎碎念"部分通常比代码本身更值钱，那里藏着大量踩坑提醒；
- **按顺序学**最好，并发那几关强烈依赖前面的 goroutine/channel 基础；
- **动手跑**每一个例子，不要只看不跑。

## 🙏 致谢

这个仓库站在两个巨人的肩膀上：

- **原项目**：[mmcgrana/gobyexample](https://github.com/mmcgrana/gobyexample) ——英文原版 Go by Example，Mark McGranaghan 出品，MIT License。
- **中文版**：[gobyexample-cn/gobyexample](https://github.com/gobyexample-cn/gobyexample) —— 中文翻译与代码注释。

**原案例的代码与选题来自以上两个项目**，本仓库在此基础上对讲解文字进行了**完全原创的重写**（「师兄带你学」风格），并以 MIT 协议发布。

如果你觉得这个项目对你有帮助，顺手给这三个项目都点个 Star 🌟 支持一下。

## 📬 反馈 & 贡献

- 发现讲解有误、有更好的类比、或者觉得哪一节太生硬？**欢迎开 Issue**；
- 觉得风格还行，**欢迎 Star + 转发给身边正在学 Go 的同学**。

## 📜 License

MIT License — 使用自由，但请保留致谢部分。

---

🎉 **79 关全部通关，祝你学得愉快！**
