# 第 52 关：JSON（师兄带你学 Go）

🎯 这一关你会学到
• encoding/json 包怎么把 Go 值变成 JSON：Marshal
• 怎么把 JSON 变回 Go 值：Unmarshal
• 结构体 tag `json:"page"` 怎么定制字段名
• 解码到 map[string]interface{} 的通用姿势 + 类型断言
• 流式编码 json.NewEncoder 和 HTTP/stdout 的对接

🤔 先想一个问题

你用美团点外卖，手机 APP 和美团服务器之间怎么传信息？订单、地址、菜单、价格——这些复杂数据在网线上怎么跑？答案是**序列化（serialization）**：把内存里的对象变成一串文本（或字节流），扔到网线上去，对面再**反序列化**还原。JSON 是当下最流行的序列化格式之一，简单、人类可读、几乎所有语言都支持。Go 标准库的 encoding/json 就是干这事儿的。这一关咱们把 Go 值和 JSON 的双向转换彻底打通。

## 📖 看代码

```go
// Go 提供内建的 JSON 编码解码（序列化反序列化）支持，
// 包括内建及自定义类型与 JSON 数据之间的转化。

package main

import (
	"encoding/json"
	"fmt"
	"os"
)

// 下面我们将使用这两个结构体来演示自定义类型的编码和解码。
type response1 struct {
	Page   int
	Fruits []string
}

// 只有 `可导出` 的字段才会被 JSON 编码/解码。必须以大写字母开头的字段才是可导出的。
type response2 struct {
	Page   int      `json:"page"`
	Fruits []string `json:"fruits"`
}

func main() {

	// 首先我们来看一下基本数据类型到 JSON 字符串的编码过程。
	// 这是一些原子值的例子。
	bolB, _ := json.Marshal(true)
	fmt.Println(string(bolB))

	intB, _ := json.Marshal(1)
	fmt.Println(string(intB))

	fltB, _ := json.Marshal(2.34)
	fmt.Println(string(fltB))

	strB, _ := json.Marshal("gopher")
	fmt.Println(string(strB))

	// 这是一些切片和 map 编码成 JSON 数组和对象的例子。
	slcD := []string{"apple", "peach", "pear"}
	slcB, _ := json.Marshal(slcD)
	fmt.Println(string(slcB))

	mapD := map[string]int{"apple": 5, "lettuce": 7}
	mapB, _ := json.Marshal(mapD)
	fmt.Println(string(mapB))

	// JSON 包可以自动的编码你的自定义类型。
	// 编码的输出只包含可导出的字段，并且默认使用字段名作为 JSON 数据的键名。
	res1D := &response1{
		Page:   1,
		Fruits: []string{"apple", "peach", "pear"}}
	res1B, _ := json.Marshal(res1D)
	fmt.Println(string(res1B))

	// 你可以给结构字段声明标签来自定义编码的 JSON 数据的键名。
	// 上面 `Response2` 的定义，就是这种标签的一个例子。
	res2D := &response2{
		Page:   1,
		Fruits: []string{"apple", "peach", "pear"}}
	res2B, _ := json.Marshal(res2D)
	fmt.Println(string(res2B))

	// 现在来看看将  JSON 数据解码为 Go 值的过程。
	// 这是一个普通数据结构的解码例子。
	byt := []byte(`{"num":6.13,"strs":["a","b"]}`)

	// 我们需要提供一个 JSON 包可以存放解码数据的变量。
	// 这里的 `map[string]interface{}` 是一个键为 string，值为任意值的 map。
	var dat map[string]interface{}

	// 这是实际的解码，以及相关错误的检查。
	if err := json.Unmarshal(byt, &dat); err != nil {
		panic(err)
	}
	fmt.Println(dat)

	// 为了使用解码 map 中的值，我们需要将他们进行适当的类型转换。
	// 例如，这里我们将 `num` 的值转换成 `float64` 类型。
	num := dat["num"].(float64)
	fmt.Println(num)

	// 访问嵌套的值需要一系列的转化。
	strs := dat["strs"].([]interface{})
	str1 := strs[0].(string)
	fmt.Println(str1)

	// 我们还可以将 JSON 解码为自定义数据类型。
	// 这样做的好处是，可以为我们的程序增加附加的类型安全性，
	// 并在访问解码后的数据时不需要类型断言。
	str := `{"page": 1, "fruits": ["apple", "peach"]}`
	res := response2{}
	json.Unmarshal([]byte(str), &res)
	fmt.Println(res)
	fmt.Println(res.Fruits[0])

	// 在上面例子的标准输出上，
	// 我们总是使用 byte和 string 作为数据和 JSON 表示形式之间的中介。
	// 当然，我们也可以像 `os.Stdout` 一样直接将 JSON 编码流传输到
	// `os.Writer` 甚至 HTTP 响应体。
	enc := json.NewEncoder(os.Stdout)
	d := map[string]int{"apple": 5, "lettuce": 7}
	enc.Encode(d)
}
```

🔍 师兄给你逐行拆

**两个结构体：response1 和 response2**

```go
type response1 struct {
    Page   int
    Fruits []string
}

type response2 struct {
    Page   int      `json:"page"`
    Fruits []string `json:"fruits"`
}
```

response1 是最朴素的结构体。response2 字段右边多了**反引号包着的"tag"**——这是 Go 结构体的**标签（tag）**机制，可以挂任意键值元数据。`json:"page"` 专门告诉 encoding/json 包："这个字段在 JSON 里叫 page"。

核心规则：**JSON 只编解码可导出字段**（首字母大写）。小写字段 JSON 当它不存在。所以 Page、Fruits 都大写是必须的。

**Marshal：Go → JSON**

```go
bolB, _ := json.Marshal(true)
fmt.Println(string(bolB))  // "true"
```

`json.Marshal(v)` 接一个任意类型的值，返回 `([]byte, error)`——JSON 的字节串加错误。string(bolB) 把 []byte 转成字符串打印。

基本类型编码结果很直观：
- `true` → `true`
- `1` → `1`
- `2.34` → `2.34`
- `"gopher"` → `"gopher"`（字符串在 JSON 里带双引号）

**切片和 map**

```go
slcD := []string{"apple", "peach", "pear"}
slcB, _ := json.Marshal(slcD)           // ["apple","peach","pear"]

mapD := map[string]int{"apple": 5, "lettuce": 7}
mapB, _ := json.Marshal(mapD)           // {"apple":5,"lettuce":7}
```

切片变成 JSON 数组，map 变成 JSON 对象（object）。注意 map 的键**必须是 string 类型**（或可编码为 string 的类型）才能编成 JSON。

**自定义结构体：response1**

```go
res1D := &response1{
    Page:   1,
    Fruits: []string{"apple", "peach", "pear"},
}
res1B, _ := json.Marshal(res1D)  // {"Page":1,"Fruits":["apple","peach","pear"]}
```

没加 tag 时，JSON 字段名直接用 Go 字段名——**大写 P 的 Page**。但一般 JSON 风格是小写（snake_case 或 camelCase），所以需要用 tag 改名。

**自定义结构体：response2 + tag**

```go
res2D := &response2{Page: 1, Fruits: []string{"apple", "peach", "pear"}}
res2B, _ := json.Marshal(res2D)  // {"page":1,"fruits":["apple","peach","pear"]}
```

有了 `json:"page"` tag，JSON 键名变成小写 page。这是实战里 99% 的写法——**前后端约定通常是 camelCase 或 snake_case，后端的 Go 代码字段必须大写但 JSON 要小写**，tag 就是这道翻译层。

tag 还能写更复杂的选项：
- `json:"-"`：这个字段不参与 JSON
- `json:"page,omitempty"`：零值时不输出这个字段
- `json:"page,string"`：强制把数字编成字符串

**Unmarshal：JSON → Go（map 版）**

```go
byt := []byte(`{"num":6.13,"strs":["a","b"]}`)
var dat map[string]interface{}
if err := json.Unmarshal(byt, &dat); err != nil {
    panic(err)
}
fmt.Println(dat)  // map[num:6.13 strs:[a b]]
```

`json.Unmarshal(data, &target)` 是反序列化。**注意要传 target 的指针**（& 取地址），因为要往里写东西。

这里用 `map[string]interface{}` 当"通用接收容器"——**interface{}（Go 1.18 后可用 any）**是"任意类型"，万金油。解码后 dat["num"] 是 `interface{}` 包着一个 float64（JSON 里所有数字默认解成 float64），dat["strs"] 是 `interface{}` 包着 `[]interface{}`。

**类型断言：把 interface{} 拆回具体类型**

```go
num := dat["num"].(float64)
fmt.Println(num)  // 6.13

strs := dat["strs"].([]interface{})
str1 := strs[0].(string)
fmt.Println(str1)  // "a"
```

`x.(T)` 是**类型断言**——"我断言 x 实际类型是 T，把它拆出来给我"。如果断言错了会 panic。安全版是 `v, ok := x.(T)` ——ok 是 bool 判断对不对，不 panic。

这段代码展示了 map 解码方式的缺点：**每次访问嵌套值都要做类型断言**，写起来烦、而且容易断错。所以绝大多数场景下推荐定义结构体解码。

**Unmarshal：JSON → 自定义结构体**

```go
str := `{"page": 1, "fruits": ["apple", "peach"]}`
res := response2{}
json.Unmarshal([]byte(str), &res)
fmt.Println(res)          // {1 [apple peach]}
fmt.Println(res.Fruits[0]) // apple
```

直接 Unmarshal 到 `&res`，res 里的字段按 tag "page"、"fruits" 对号入座填进去。访问字段 `res.Fruits[0]` 用的是 Go 原生语法，**不需要类型断言**，类型安全又干净。这就是生产代码里的标准姿势。

**流式编码：json.NewEncoder**

```go
enc := json.NewEncoder(os.Stdout)
d := map[string]int{"apple": 5, "lettuce": 7}
enc.Encode(d)  // 直接写到 stdout：{"apple":5,"lettuce":7}\n
```

前面的 Marshal 都是"编成 []byte 再自己处理"。流式编码器 `json.NewEncoder(w)` 接一个 `io.Writer`，`enc.Encode(v)` 直接把 JSON 写到 writer 里（末尾自动加 \n）。

最常见的用法是 HTTP 服务：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(myData)
}
```

不用先 Marshal 成字节再 Write，省内存、更快。

🏃 跑一下试试

保存为 json.go，然后：

```bash
go run json.go
```

预期输出：

```text
true
1
2.34
"gopher"
["apple","peach","pear"]
{"apple":5,"lettuce":7}
{"Page":1,"Fruits":["apple","peach","pear"]}
{"page":1,"fruits":["apple","peach","pear"]}
map[num:6.13 strs:[a b]]
6.13
a
{1 [apple peach]}
apple
{"apple":5,"lettuce":7}
```

逐段看：
- 前 4 行：基本类型 Marshal
- 第 5-6 行：切片、map Marshal
- 第 7 行 `{"Page":1,...`：response1 没 tag，字段名大写
- 第 8 行 `{"page":1,...`：response2 有 tag，字段名小写
- 第 9 行：map 解码结果（Go 的 fmt 把 map 打成 map[k:v k2:v2] 格式）
- `6.13` 和 `a`：类型断言后拿到的具体值
- `{1 [apple peach]}`：结构体解码，fmt 的默认打印风格
- 最后一行：流式编码直接打到 stdout

💡 师兄的碎碎念

师兄的 JSON 生存手册：

1. **永远定义结构体而不是用 map[string]interface{}**。结构体类型安全、自文档、IDE 能补全，map 方案就是挖坑给未来的自己。

2. **tag 的 omitempty 要谨慎**。零值也可能是有意义的（比如 "is_deleted": false），随便 omitempty 会把这种值从 JSON 里删掉。

3. **数字精度**：JSON 里没有 int 和 float 之分，Unmarshal 到 `interface{}` 时所有数字都变 float64。解析大整数（超过 2^53）要用 `json.Decoder.UseNumber()` 或者字段类型直接用 int64。

4. **时间**：time.Time 默认编码为 RFC3339 格式字符串。如果前端约定用 Unix 时间戳（integer），要自己实现 MarshalJSON / UnmarshalJSON 方法。

5. **性能敏感场景**：标准库 encoding/json 是反射驱动，速度中等。极致性能用 `github.com/bytedance/sonic` 或 `github.com/goccy/go-json`，兼容 API 但快几倍。

6. **安全**：Go 1.21+ 的 `json/v2` 正在设计中，未来会变更好。当前 encoding/json 有个历史包袱就是默认允许未知字段，生产环境建议 `Decoder.DisallowUnknownFields()` 严格化。

吐槽一下 Go 的 JSON tag 语法："json:\"name,omitempty\"" 一个 tag 一堆逗号分隔的选项，写起来丑但你还得用。

🎓 知识点清单

- json.Marshal(v) 把 Go 值编成 JSON []byte；成对的 json.Unmarshal(data, &v) 解码
- 只有可导出字段（首字母大写）才会参与 JSON
- 结构体 tag `json:"name"` 改 JSON 键名，`omitempty` 跳零值，`-` 完全跳过
- map[string]interface{} + 类型断言是通用解码姿势，但类型不安全
- 优先定义结构体解码，拿到的字段直接访问不用断言
- JSON 数字默认解成 float64，大整数要特殊处理
- json.NewEncoder(w).Encode(v) 流式编码，适合写到 HTTP/stdout/文件

➡️ 下一关

JSON 之外还有老牌的 **XML**——虽然现在主要在企业系统、配置文件里见，但 Go 的 encoding/xml 依然要会。下一关 53「XML」咱们看看 XML 的编解码长啥样，走起！
