# learn-go-by-example-cn · 师兄带你学 Go 🚀

> 一份面向零基础的《Go by Example》中文讲解改写版。不讲八股，不背定义，让师兄用大白话 + 生活类比 + 偶尔吐槽，把 Go 的每个语言特性讲到你一眼看懂。

## 📖 这是什么

[Go by Example](https://gobyexample.com/) 是一份人尽皆知的 Go 入门教程，中文版在 [gobyexample-cn](https://gobyexample-cn.github.io/) 维护。这两个项目都非常优秀，但代码旁边的讲解更偏"官方文档"风格，对纯小白来说还是有点生硬。

本仓库做一件事：**保留原始案例的代码不动**，把每个案例旁边的讲解用"计算机系学长坐你旁边手把手讲"的风格**原创重写**。每一关都按统一模板组织：

- 🎯 这一关你会学到
- 🤔 先想一个问题（生活化场景切入）
- 📖 看代码（完整的 .go 源文件）
- 🔍 师兄给你逐行拆（核心讲解）
- 🏃 跑一下试试
- 💡 师兄的碎碎念（踩坑/延伸）
- 🎓 这一关的知识点清单（面试/复习用）
- ➡️ 下一关

## 🗺️ 目录（本批：前 5 关试水）

| 序号 | 案例 | 讲什么 |
| :---: | --- | --- |
| 01 | [Hello World](./01-hello-world/) | 第一个 Go 程序，package/import/func main 三件套 |
| 02 | [Values](./02-values/) | 字符串、整数、浮点、布尔的最基本用法 |
| 03 | [Variables](./03-variables/) | `var` 声明、`:=` 简写、零值机制 |
| 04 | [Constants](./04-constants/) | `const` 声明、无类型常量、运行期推断 |
| 05 | [For](./05-for/) | Go 唯一的循环关键字 `for` 的四种写法 + break/continue |

> 💡 **目前只发布了前 5 关**，是师兄先放出来试水风格。欢迎在 Issue 区告诉我哪里不好懂、哪里可以更生动——我会根据反馈再补完后面 70+ 关（if/else、switch、array、slice、map、函数、闭包、goroutine、channel …）。

## 🛠️ 怎么把案例跑起来

1. 装 Go：前往 [go.dev/dl](https://go.dev/dl/) 下载对应平台的安装包，或用包管理器（`brew install go` / `apt install golang-go`）。
2. 克隆仓库：
   ```bash
   git clone https://github.com/CXP-shawn/learn-go-by-example-cn.git
   cd learn-go-by-example-cn
   ```
3. 进入某一关的文件夹，直接跑：
   ```bash
   cd 01-hello-world
   go run hello-world.go
   ```

## ⚠️ 致谢 & 版权声明

- 原始案例代码与教学结构来源：[mmcgrana/gobyexample](https://github.com/mmcgrana/gobyexample)（MIT License，Copyright © Mark McGranaghan）。
- 中文化代码注释来源：[gobyexample-cn/gobyexample](https://github.com/gobyexample-cn/gobyexample)（MIT License）。
- 本仓库的 **讲解文字全部原创重写**，代码部分继承原仓库许可。
- 本仓库整体以 **MIT License** 发布，使用自由，但请保留以上致谢。

如果你觉得对自己有帮助，顺手给 [mmcgrana/gobyexample](https://github.com/mmcgrana/gobyexample) 和 [gobyexample-cn/gobyexample](https://github.com/gobyexample-cn/gobyexample) 点个 Star，真正支撑起这个系列的是他们 🙏

## 📬 反馈

- 发现讲解有误、有更好的类比、或者觉得哪节太生硬？欢迎开 Issue。
- 觉得风格还不错？去 Star ⭐ 一下这个仓库，然后告诉师兄："后面继续发车吧！"
