# 第 50 关：文本模板（师兄带你学 Go）

🎯 这一关你会学到
• Go 标准库的 text/template 包怎么玩：把占位符 `{{.}}` 填上真实数据
• Parse / Must / Execute 三板斧的分工
• 如何访问结构体字段（`{{.FieldName}}`）和 map 键值
• if/else、range 这两个常用模板动作
• 右侧 `-` 修饰符去掉多余空格的小技巧

🤔 先想一个问题

你给女朋友写情书——"亲爱的 XXX，今天是咱们在一起 XX 天"——每次换个人、换个日期，正文就得重写？当然不是，咱们用"**模板**"：中间名字、天数留个空，每次填进去就行。Go 的 text/template 就是这把模板枪，能让你一次写好骨架，N 次填不同数据，批量产出文本。邮件、配置文件、代码生成、HTML 页面（用 html/template 配套）都靠它。这一关咱们把模板玩法摸个底。

## 📖 看代码

```go
// Go 使用 `text/template` 包为创建动态内容或向用户显示自定义输出提供了内置支持。
// 一个名为 `html/template` 的兄弟软件包提供了相同的 API，但具有额外的安全功能，被用于生成 HTML。

package main

import (
	"os"
	"text/template"
)

func main() {

	// 我们可以创建一个新模板，并从字符串解析其正文。
	// 模板是静态文本和包含在”动作“中用于动态插入内容的混合体。
	t1 := template.New("t1")
	t1, err := t1.Parse("Value is {{.}}\n")
	if err != nil {
		panic(err)
	}

	// 或者，我们可以使用 `template.Must` 函数，在 `Parse` 错误时返回 panic。
	// 这对于在全局作用域中初始化的模板非常有用。
	t1 = template.Must(t1.Parse("Value: {{.}}\n"))

	// 通过“执行”模板，我们生成其文本，其中包含其“动作”的具体值。
	// `{{.}}` 动作被作为参数传递给 `Execute` 的值所代替。
	t1.Execute(os.Stdout, "some text")
	t1.Execute(os.Stdout, 5)
	t1.Execute(os.Stdout, []string{
		"Go",
		"Rust",
		"C++",
		"C#",
	})

	// 我们将在下面使用辅助函数。
	Create := func(name, t string) *template.Template {
		return template.Must(template.New(name).Parse(t))
	}

	// 如果数据是一个结构体，我们可以使用 `{{.FieldName}}` 动作来访问其字段。
	// 这些字段应该是导出的，以便在模板执行时可访问。
	t2 := Create("t2", "Name: {{.Name}}\n")

	t2.Execute(os.Stdout, struct {
		Name string
	}{"Jane Doe"})

	// 这同样适用于 map；在 map 中没有限制键名的大小写。
	t2.Execute(os.Stdout, map[string]string{
		"Name": "Mickey Mouse",
	})

	// if/else 提供了条件执行模板。如果一个值是类型的默认值，例如 0、空字符串、空指针等，
	// 则该值被认为是 false。
	// 这个示例演示了另一个模板特性：使用 `-` 在动作中去除空格。
	t3 := Create("t3",
		"{{if . -}} yes {{else -}} no {{end}}\n")
	t3.Execute(os.Stdout, "not empty")
	t3.Execute(os.Stdout, "")

	// range 块允许我们遍历切片，数组，映射或通道。在 range 块内，`{{.}}` 动作被设置为迭代的当前项。
	t4 := Create("t4",
		"Range: {{range .}}{{.}} {{end}}\n")
	t4.Execute(os.Stdout,
		[]string{
			"Go",
			"Rust",
			"C++",
			"C#",
		})
}
```

🔍 师兄给你逐行拆

**text/template 是 Go 的原生模板引擎**

标准库里一共两个模板包：`text/template` 纯文本场景用，`html/template` 同一个 API 但多了**自动转义**（防 XSS），输出 HTML 时用它。师兄这里强调：凡是往浏览器送的内容**务必用 html/template**，否则你会成为黑客的提款机。

模板语法很简单：静态文本里嵌 `{{...}}` 就是"**动作（action）**"。动作里可以放变量、函数调用、条件判断、循环。`{{.}}` 是最基础的一个——代表"当前传进来的那整块数据"。

**t1：New + Parse 的标准姿势**

```go
t1 := template.New("t1")
t1, err := t1.Parse("Value is {{.}}\n")
if err != nil {
    panic(err)
}
```

先 `template.New("t1")` 创建一个名字叫 "t1" 的空模板对象（名字主要用于错误信息和嵌套模板查找），然后 `t1.Parse(模板字符串)` 把模板字符串解析进去。**Parse 返回 (*Template, error)**，解析失败（比如动作语法错）会返回错，代码里用 if err != nil panic 兜住。

然后下一行：
```go
t1 = template.Must(t1.Parse("Value: {{.}}\n"))
```

**template.Must** 是个辅助函数——它接收 (t, err) 两个参数，如果 err 非 nil 就直接 panic，否则返回 t。这对"初始化时必须成功、失败就没救了"的场景极好用，省得到处写 if err != nil。注意：Must 会让同一个 t1 **替换掉之前的模板内容**（Parse 在同一个 Template 上调用会覆盖）。所以这行之后 t1 的模板变成 `"Value: {{.}}\n"`。

**Execute：真正填坑的那一刻**

```go
t1.Execute(os.Stdout, "some text")
t1.Execute(os.Stdout, 5)
t1.Execute(os.Stdout, []string{"Go", "Rust", "C++", "C#"})
```

Execute 接两个参数：**第一个是输出目标（io.Writer，os.Stdout 是标准输出）**，第二个是"数据"。模板里的 `{{.}}` 就代表这个数据。传字符串 "some text" → 输出 `Value: some text`；传整数 5 → 输出 `Value: 5`；传 string 切片 → 输出 `Value: [Go Rust C++ C#]`。注意模板自带 \n 所以每次换行。

**t2：访问结构体字段 / map 键**

```go
Create := func(name, t string) *template.Template {
    return template.Must(template.New(name).Parse(t))
}
t2 := Create("t2", "Name: {{.Name}}\n")
```

先定义一个闭包 Create 当工厂，一步到位 New + Must + Parse。

然后 t2 模板是 `"Name: {{.Name}}\n"` —— `{{.Name}}` 表示访问"当前数据的 Name 字段"。

```go
t2.Execute(os.Stdout, struct {
    Name string
}{"Jane Doe"})
```

这里传的是一个**匿名结构体**（就地定义并实例化）。字段 Name 赋值 "Jane Doe"。模板渲染出 `Name: Jane Doe`。

**⚠️ 坑：结构体字段必须是导出的**（首字母大写），template 用**反射**（reflect 包，运行时检查类型的机制）读字段。你写 `name string` 小写，template 根本看不到这个字段。

接下来：
```go
t2.Execute(os.Stdout, map[string]string{
    "Name": "Mickey Mouse",
})
```

同样的模板 `{{.Name}}` 传给 map 也能工作——template 会按"**先找字段、再找 map 键、再找方法**"的顺序解析。而且 map 的键**不要求大写**，因为不走反射字段导出规则。

**t3：if/else 和空格修饰符 `-`**

```go
t3 := Create("t3",
    "{{if . -}} yes {{else -}} no {{end}}\n")
t3.Execute(os.Stdout, "not empty")
t3.Execute(os.Stdout, "")
```

`{{if .}} ... {{else}} ... {{end}}` 是条件分支。判断规则是 Go 的"**零值为假**"：nil、false、0、空字符串、空切片、空 map 都算 false，其他都算 true。

`-` 这个小修饰符很关键：写在动作**左**侧（`{{- ...`）表示**吃掉左边相邻的空白**；写在**右**侧（`... -}}`）表示**吃掉右边的空白**。这里 `{{if . -}}` 是吃掉 `{{if .}}` 这个动作到下一个非空白字符之间的空格/换行。为啥要这么干？因为你在模板里为了可读性会加换行/缩进，但不希望这些空白出现在最终输出里，`-` 就是"**消除多余空白**"的工具。

输出：
- 传 "not empty" → `yes \n`（前面那个空格是 `{{if . -}} yes` 里"yes"前面留的一个空格）
- 传 "" → `no \n`（空字符串算 false，走 else 分支）

**t4：range 遍历**

```go
t4 := Create("t4",
    "Range: {{range .}}{{.}} {{end}}\n")
t4.Execute(os.Stdout, []string{"Go", "Rust", "C++", "C#"})
```

`{{range .}}...{{end}}` 是循环结构，可遍历切片、数组、map、channel。**range 内部的 `{{.}}` 表示当前这一项**，不再是外层的整个数据。

输出：`Range: Go Rust C++ C# \n`（每项后带一个空格，所以末尾有个多余空格）

🏃 跑一下试试

保存为 text-templates.go，然后：

```bash
go run text-templates.go
```

预期输出：

```text
Value: some text
Value: 5
Value: [Go Rust C++ C#]
Name: Jane Doe
Name: Mickey Mouse
 yes 
 no 
Range: Go Rust C++ C# 
```

逐行看：
- 前三行是 t1 的三次 Execute，分别把字符串、整数、切片填进 `Value: {{.}}` 模板
- 第 4-5 行是 t2 分别对结构体和 map 渲染 `Name: {{.Name}}`
- 第 6 行 " yes "：传 "not empty" 走 if 分支；开头空格是 `{{if . -}}` 右侧的 `-` 吃掉了 if 后换行，但 "yes" 前面还有一个显式空格
- 第 7 行 " no "：传空字符串走 else
- 第 8 行：range 迭代切片，每项后跟一个空格

💡 师兄的碎碎念

text/template 入门就这些，但生产环境有几个高阶技巧师兄提一下：

1. **文件模板优先**。别把模板字符串写在 Go 源码里。用 `template.ParseFiles("xxx.tmpl")` 从外部文件加载，改模板不用重编译代码。

2. **嵌套模板用 `{{template "name" .}}`**。拆成 header / footer / main 三块，复用性爆棚。

3. **自定义函数用 `Funcs(...)`**。想在模板里用 `{{.Price | formatMoney}}` 这种管道函数，提前注册 Funcs。

4. **web 场景一律 html/template**。它会对输出自动做 HTML 转义，避免 XSS（跨站脚本攻击）。别为了"简单"用 text/template 返回 HTML。

5. **性能**：模板 Parse 一次、Execute 多次，不要每次请求都 Parse。

吐槽一句：Go 的模板语法不如 Python 的 Jinja2 / Django 强大，循环里加个 `index` 都得自己搞，习惯了 Jinja 的同学刚上手会觉得憋屈。但胜在内置、零依赖、跨平台。

🎓 知识点清单

- text/template 是纯文本模板，html/template 是 HTML 专用（带自动转义）
- 模板里 `{{...}}` 叫"动作"，`{{.}}` 表示当前数据
- 三板斧：template.New(name) / Parse(src) / Execute(w, data)
- template.Must 把 (t, err) 的错误直接 panic，初始化场景好用
- 访问结构体字段 `{{.Field}}`（字段必须导出）、访问 map `{{.Key}}`（键不限大小写）
- if/else：零值算 false
- range 遍历：内部 `{{.}}` 是当前项
- 空格修饰符 `-`：`{{- ...` 吃左边空白，`... -}}` 吃右边空白
- 生产环境用 ParseFiles 从文件加载、Funcs 注册自定义函数、Execute 前 Parse 一次

➡️ 下一关

会填模板之后，咱们回到字符串处理家族——下一关 51「正则表达式 regular-expressions」带你玩转 Go 的 regexp 包，抓取邮箱、手机号、URL 这些结构化片段再也不怕，走起！
