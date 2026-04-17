# 第 53 关：XML（师兄带你学 Go）

🎯 这一关你会学到
• encoding/xml 包的 Marshal / MarshalIndent / Unmarshal 三板斧
• XML 结构体 tag 和 JSON tag 的差异（XMLName、attr、嵌套路径）
• XML 头部 `<?xml version="1.0" ...?>` 怎么加
• `parent>child>plant` 这种嵌套路径标签的用法
• XML 属性（attribute）和子元素（element）的区别

🤔 先想一个问题

你打开一份老板发的 Office 合同文件，其实 docx 本质就是一堆 XML 文件打包成 zip。再比如 Maven 项目的 pom.xml、Spring 的配置文件、安卓 AndroidManifest.xml——XML 在企业软件圈至今**根基深厚**。虽然新系统首选 JSON，但老接口、大厂遗产、配置文件里还是经常遇到 XML。Go 的 encoding/xml 包让你跟 XML 打交道不用像 Java 那样拽出一堆解析器。这一关咱们看看 Go 怎么优雅处理 XML。

## 📖 看代码

```go
// Go 通过 `encoding.xml` 包为 XML 和 类-XML 格式提供了内建支持。

package main

import (
	"encoding/xml"
	"fmt"
)

//	`Plant` 结构将被映射到 `XML` 。
//
// 与 `JSON` 示例类似，字段标签包含用于编码器和解码器的指令。
// 这里我们使用了 `XML` 包的一些特性：
//
//	`XMLName` 字段名称规定了该结构的 `XML` 元素名称；
//	`id,attrr` 表示 `Id` 字段是一个 `XML` 属性，而不是嵌套元素。
type Plant struct {
	XMLName xml.Name `xml:"plant"`
	Id      int      `xml:"id,attr"`
	Name    string   `xml:"name"`
	Origin  []string `xml:"origin"`
}

func (p Plant) String() string {
	return fmt.Sprintf("Plant id=%v, name=%v, origin=%v",
		p.Id, p.Name, p.Origin)
}

func main() {
	coffee := &Plant{Id: 27, Name: "Coffee"}
	coffee.Origin = []string{"Ethiopia", "Brazil"}

	// 传入我们声明了 XML 的 Plant 类型。
	// 使用 `MarshalIndent` 生成可读性更好的输出结果。
	out, _ := xml.MarshalIndent(coffee, " ", "  ")
	fmt.Println(string(out))

	// 明确的为输出结果添加一个通用的 XML 头部信息。
	fmt.Println(xml.Header + string(out))

	// 使用 `Unmarshal` 将 `XML` 格式字节流解析到 `Plant` 结构。
	// 如果 `XML` 格式错误或无法映射到 `Plant` 结构，将返回一个描述性错误。
	var p Plant
	if err := xml.Unmarshal(out, &p); err != nil {
		panic(err)
	}
	fmt.Println(p)

	tomato := &Plant{Id: 81, Name: "Tomato"}
	tomato.Origin = []string{"Mexico", "California"}

	// `parent>child>plant` 字段标签告诉编码器，将 `Plants` 中的元素嵌套在 `<parent><child>` 里面。
	type Nesting struct {
		XMLName xml.Name `xml:"nesting"`
		Plants  []*Plant `xml:"parent>child>plant"`
	}

	nesting := &Nesting{}
	nesting.Plants = []*Plant{coffee, tomato}

	out, _ = xml.MarshalIndent(nesting, " ", "  ")
	fmt.Println(string(out))
}
```

🔍 师兄给你逐行拆

**Plant 结构体和 XML tag 解剖**

```go
type Plant struct {
    XMLName xml.Name `xml:"plant"`
    Id      int      `xml:"id,attr"`
    Name    string   `xml:"name"`
    Origin  []string `xml:"origin"`
}
```

逐字段解释：

- **XMLName** 是一个特殊字段（类型 xml.Name），它的 tag `xml:"plant"` 告诉编码器"这个结构体序列化时根元素名叫 `<plant>`"。没有 XMLName 字段的话编码器会用结构体类型名（Plant）作为元素名，有了就以 tag 为准。
- **Id** 的 tag `xml:"id,attr"` —— 后面那个 `,attr` 非常关键，意思是"Id 编码成 **XML 属性**（attribute）而不是子元素"。结果会是 `<plant id="27">...</plant>`，id 放在开标签里面。
- **Name** 的 `xml:"name"` 是普通子元素，结果 `<name>Coffee</name>`。
- **Origin** 是字符串切片，tag 是 `xml:"origin"`。切片里每一项会被展开成**多个同名子元素**：如果 Origin 有两项 ["Ethiopia", "Brazil"]，会生成 `<origin>Ethiopia</origin><origin>Brazil</origin>`。

**XML 属性 vs 子元素 是什么概念？**

XML 里有两种表达方式：
- **属性（attribute）**：`<plant id="27" type="fruit">...</plant>` —— 放在开标签里，一个元素上多个属性是空格分隔的 key=value
- **子元素（element）**：`<plant><id>27</id></plant>` —— 一个嵌套标签

两者语义相近但有区别：属性通常放**简单标识**（id、type、version），子元素放**复杂/重复数据**。用哪个看协议约定。

**String() 方法：让 Plant 有自定义打印格式**

```go
func (p Plant) String() string {
    return fmt.Sprintf("Plant id=%v, name=%v, origin=%v",
        p.Id, p.Name, p.Origin)
}
```

这是 Go 的 **Stringer 接口**——任何实现了 `String() string` 方法的类型，fmt.Println 打印时会优先调用 String() 而不是走默认格式。这里给 Plant 加上之后，下面 `fmt.Println(p)` 就会自动打 `Plant id=27, name=Coffee, origin=[Ethiopia Brazil]`，比默认 `{... 27 Coffee [Ethiopia Brazil]}` 好看得多。

**main：coffee 实例 + MarshalIndent**

```go
coffee := &Plant{Id: 27, Name: "Coffee"}
coffee.Origin = []string{"Ethiopia", "Brazil"}

out, _ := xml.MarshalIndent(coffee, " ", "  ")
fmt.Println(string(out))
```

`xml.MarshalIndent(v, prefix, indent)` 和 xml.Marshal 功能相同，但会**美化输出**——每行前面加 prefix，嵌套每层加一次 indent。这里 prefix=" "（一个空格）、indent="  "（两个空格）。输出类似：

```xml
 <plant id="27">
   <name>Coffee</name>
   <origin>Ethiopia</origin>
   <origin>Brazil</origin>
 </plant>
```

注意 Id 跑到了开标签的属性位置、Origin 切片被展开成多行。

**添加 XML 头部**

```go
fmt.Println(xml.Header + string(out))
```

**xml.Header** 是包里预定义的常量 `"<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n"`。正式的 XML 文档都要加这行头部声明版本和编码。

**Unmarshal：XML → Plant**

```go
var p Plant
if err := xml.Unmarshal(out, &p); err != nil {
    panic(err)
}
fmt.Println(p)  // Plant id=27, name=Coffee, origin=[Ethiopia Brazil]
```

和 JSON 那节一样，`xml.Unmarshal(data, &target)` 解析 XML 字节到结构体指针。解析时会按 tag 回填。因为 Plant 实现了 String() 方法，fmt.Println 走自定义格式。

**嵌套路径：parent>child>plant**

```go
type Nesting struct {
    XMLName xml.Name `xml:"nesting"`
    Plants  []*Plant `xml:"parent>child>plant"`
}
nesting := &Nesting{}
nesting.Plants = []*Plant{coffee, tomato}

out, _ = xml.MarshalIndent(nesting, " ", "  ")
fmt.Println(string(out))
```

关键是这个 `xml:"parent>child>plant"` tag —— 用 **`>`** 表示嵌套路径。意思是："Plants 切片里每一项序列化时都要被 `<parent><child>...</child></parent>` 包起来"。输出大概长这样：

```xml
 <nesting>
   <parent>
     <child>
       <plant id="27">
         <name>Coffee</name>
         ...
       </plant>
       <plant id="81">
         <name>Tomato</name>
         ...
       </plant>
     </child>
   </parent>
 </nesting>
```

这个语法在处理那些**深度嵌套但 Go 端不想也跟着嵌套**的 XML schema 时特别爽——不用为 parent、child 分别建结构体，一行 tag 搞定。

🏃 跑一下试试

保存为 xml.go，然后：

```bash
go run xml.go
```

预期输出大致如下（前缀/缩进和原始代码参数保持一致）：

```text
 <plant id="27">
   <name>Coffee</name>
   <origin>Ethiopia</origin>
   <origin>Brazil</origin>
 </plant>
<?xml version="1.0" encoding="UTF-8"?>
 <plant id="27">
   <name>Coffee</name>
   <origin>Ethiopia</origin>
   <origin>Brazil</origin>
 </plant>
Plant id=27, name=Coffee, origin=[Ethiopia Brazil]
 <nesting>
   <parent>
     <child>
       <plant id="27">
         <name>Coffee</name>
         <origin>Ethiopia</origin>
         <origin>Brazil</origin>
       </plant>
       <plant id="81">
         <name>Tomato</name>
         <origin>Mexico</origin>
         <origin>California</origin>
       </plant>
     </child>
   </parent>
 </nesting>
```

分三段看：
- 第一段：单个 Plant 的 MarshalIndent 输出
- 第二段：加 xml.Header 之后，顶部多了 `<?xml version="1.0" encoding="UTF-8"?>`
- 第三段：Unmarshal 回 Plant 再 Println，走 String() 方法
- 第四段：Nesting 结构用嵌套路径 tag，自动嵌套三层标签

💡 师兄的碎碎念

XML 的坑比 JSON 多，师兄提醒：

1. **XML 没 JSON 那么"原生数字"**。所有内容都是文本，`<age>27</age>` 里的 27 是字符串。Go 的 xml 包会根据结构体字段类型尝试转换，失败就报错。

2. **命名空间（namespace）**。生产 XML 经常带 `xmlns:soap`、`xmlns:xsi` 之类的命名空间。xml 包通过 tag `xml:"ns name"`（空格分隔）处理，水挺深。

3. **CDATA 区块**：`<![CDATA[任意文本]]>` 在 xml 包里自动处理，正常字段拿到的就是原文本。

4. **XML 比 JSON 冗长得多**——标签开关要对、属性要引号，同样数据 XML 大几倍。这也是现代系统弃 XML 选 JSON 的主因。

5. **生产解析 XML 用流式 `xml.NewDecoder(r).Token()`**，大文件别整个 Unmarshal 吃内存。

6. **html 的坑**：XML 解析器严格，HTML 不一定合法 XML（比如不闭合的 `<br>`）。解析 HTML 用 `golang.org/x/net/html` 或 `goquery`。

吐槽一句：XML 的 tag 语法 `xml:"id,attr"` 读起来就像行话，新人看到一脸懵。不过熟了之后会觉得挺紧凑。

🎓 知识点清单

- encoding/xml 的 Marshal / MarshalIndent / Unmarshal 和 JSON 三板斧对称
- 结构体字段 tag 语法：`xml:"name"` 子元素、`xml:"id,attr"` 属性、`xml:"parent>child>name"` 嵌套路径
- 特殊字段 XMLName（xml.Name 类型）指定根元素名
- xml.Header 是预定义的 XML 声明头部常量
- 字符串切片字段会被展开成多个同名子元素
- 实现 Stringer 接口（String() string）可自定义 fmt.Println 格式
- XML 命名空间 tag 用空格分隔：`xml:"ns tagname"`
- 大文件解析优先用流式 Decoder，别整个 Unmarshal

➡️ 下一关

数据格式讲完，咱们来看看另一个每天都打交道的东西——**时间**。下一关 54「时间 time」带你玩转 Go 的 time 包：取现在、解析日期字符串、计算时间差、时区处理一条龙，走起！
