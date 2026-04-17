# 第 44 关：使用函数自定义排序（师兄带你学 Go）

🎯 这一关你会学到
• 什么是「命名类型（named type）」，为什么要给 []string 取一个新名字
• 如何实现 sort.Interface 接口的三个方法：Len、Swap、Less
• 接口的「鸭子类型」本质：Go 不看身份证，只看你会不会做这三件事
• 用 byLength(fruits) 做类型转换，让 sort.Sort 帮你按字符串长度排序

🤔 先想一个问题

你去便利店买东西，收银台的店员不管你是学生、上班族还是外卖小哥，只要你能「扫码付款」，她就让你结账走人。她根本不问你叫什么名字、从哪里来——只要你会做那一个动作，她就认可你。

Go 的 sort.Sort 也是这个脾气：它不管你的数据是字符串切片、数字切片还是自定义结构体，只要你能告诉它「我有多少个元素、怎么交换两个、怎么比大小」这三件事，它就帮你排好序。这三件事，在 Go 里叫做 sort.Interface 接口。这一关，咱们就来搞懂怎么让自己的类型「学会」这三件事。

## 📖 看代码

```go
// 有时候，我们可能想根据自然顺序以外的方式来对集合进行排序。
// 例如，假设我们要按字符串的长度而不是按字母顺序对它们进行排序。
// 这儿有一个在 Go 中自定义排序的示例。

package main

import (
	"fmt"
	"sort"
)

// 为了在 Go 中使用自定义函数进行排序，我们需要一个对应的类型。
// 我们在这里创建了一个 `byLength` 类型，它只是内建类型 `[]string` 的别名。
type byLength []string

// 我们为该类型实现了 `sort.Interface` 接口的 `Len`、`Less` 和 `Swap` 方法，
// 这样我们就可以使用 `sort` 包的通用 `Sort` 方法了，
// `Len` 和 `Swap` 在各个类型中的实现都差不多，
// `Less` 将控制实际的自定义排序逻辑。
// 在这个的例子中，我们想按字符串长度递增的顺序来排序，
// 所以这里使用了 `len(s[i])` 和 `len(s[j])` 来实现 `Less`。
func (s byLength) Len() int {
	return len(s)
}
func (s byLength) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}
func (s byLength) Less(i, j int) bool {
	return len(s[i]) < len(s[j])
}

// 一切准备就绪后，我们就可以通过将切片 `fruits` 强转为 `byLength` 类型的切片，
// 然后对该切片使用 `sort.Sort` 来实现自定义排序。
func main() {
	fruits := []string{"peach", "banana", "kiwi"}
	sort.Sort(byLength(fruits))
	fmt.Println(fruits)
}
```

🔍 师兄给你逐行拆

**第一步：先看整体脉络**

这段代码干的事情，用一句话说就是：把一个字符串切片 ["peach", "banana", "kiwi"] 按照字符串的长度从短到长排序，最后输出 [kiwi peach banana]。但它没有用什么神奇的内置函数，而是自己定义了一套「排序规则」挂上去。咱们一行一行来拆。

---

**type byLength []string ——命名类型是什么？**

首先要搞清楚一个概念：**命名类型（named type）**。Go 里面，[]string 是一个「已有的类型」，但它是匿名的、通用的，没有自己的名字。当你写 `type byLength []string`，你就是在说：「我要创建一个新类型，名字叫 byLength，它的底层结构和 []string 完全一样。」

为什么要这么做？因为 Go 的方法（method）只能定义在「你自己包里声明的命名类型」上，不能直接定义在 []string 这种内置类型或者外包类型上。如果你想给 []string 绑定三个方法，你必须先给它取个名字，才能在它身上挂方法。这就好比你想给外卖平台的配送系统加一个「按距离排序」的功能，平台原版代码你改不了，所以你继承一个子类、给子类加功能——Go 里的命名类型就是这个思路，只不过更简单直接。

---

**为什么要实现三个方法？——认识 sort.Interface**

Go 标准库的 sort.Sort 函数，它的函数签名长这样：

```go
func Sort(data Interface)
```

这个 Interface 不是随便说说的，它是 sort 包里定义的一个接口（interface）。接口是 Go 里的一个核心概念，**接口（interface）就是一组方法签名的集合，它规定了「要成为这个接口的一员，你必须实现哪些方法」**。sort.Interface 长这样：

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

就这三个方法。sort.Sort 函数只认这个接口，不管你传进来的是什么类型，只要你实现了这三个方法，sort.Sort 就能帮你排序。这就是 Go 著名的「鸭子类型（duck typing）」哲学：**不看你是什么，只看你能做什么**。就像那个便利店收银员，她不看你的身份证，只要你会扫码付款，你就能结账。

---

**Len() int ——告诉我有多少个**

```go
func (s byLength) Len() int {
    return len(s)
}
```

这里出现了一个新语法：**方法接收者（receiver）**。`(s byLength)` 这一段放在 func 关键字后面、方法名前面，意思是「这个方法属于 byLength 类型，调用时，当前的 byLength 值会被绑定到变量 s 上」。你可以把它理解成其他语言里的 this 或 self，但 Go 把它显式地写出来，更清晰。

Len 方法返回切片的长度。因为 byLength 的底层就是 []string，所以直接 len(s) 就搞定了。sort.Sort 在排序前需要知道「这堆数据一共有几个」，就是靠调用 Len() 来获取的。简单，但必不可少。

---

**Swap(i, j int) ——把第 i 和第 j 个对调**

```go
func (s byLength) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}
```

Swap 方法接受两个下标 i 和 j，把对应位置的元素互换。这是 Go 里典型的「多重赋值」写法，一行搞定两个变量的交换，不需要临时变量，非常简洁。排序算法在工作时，会不断地比较和交换元素，Swap 就是「执行交换」的那个动作。

注意这里接收者 s 是值类型（byLength，不是 *byLength），但 byLength 底层是切片，切片本身是引用类型，所以对 s[i] 和 s[j] 的修改会直接反映到原始的 fruits 上。这是 Go 切片的一个关键特性，师兄要特别提醒你记住：**切片作为方法接收者时，修改元素是有效的，因为切片头部包含指向底层数组的指针**。

---

**Less(i, j int) bool ——这才是排序规则的灵魂**

```go
func (s byLength) Less(i, j int) bool {
    return len(s[i]) < len(s[j])
}
```

Less 是这三个方法里最重要的一个，它定义了「什么叫做更小」。sort.Sort 在比较两个元素时会调用 Less(i, j)，如果返回 true，说明第 i 个元素应该排在第 j 个元素前面。

在这里，咱们的规则是：比较 s[i] 和 s[j] 的字符串长度，谁短谁排前面。所以 "kiwi"（4个字母）会排在 "peach"（5个字母）前面，"peach" 排在 "banana"（6个字母）前面。

这就是自定义排序最灵活的地方：**你可以在 Less 里写任何你想要的比较逻辑**。按名字字母顺序？按价格高低？按游戏角色的战斗力？只要在 Less 里改一行，整个排序规则就变了。标准库的 sort.Sort 完全不关心你的比较逻辑是什么，它只管「你告诉我谁小，我就把小的往前移」。

---

**main 里的类型转换 byLength(fruits)**

```go
fruits := []string{"peach", "banana", "kiwi"}
sort.Sort(byLength(fruits))
```

fruits 是一个 []string，但 sort.Sort 需要的是 sort.Interface。咱们的 byLength 实现了 sort.Interface，但 fruits 本身的类型是 []string，不是 byLength。所以这里用 `byLength(fruits)` 做了一次**类型转换（type conversion）**：把 []string 转成 byLength。

因为 byLength 的底层类型就是 []string，这个转换是零成本的，不会复制数据，只是换了一个「外壳」，让编译器把它当成 byLength 来处理。转换完之后，byLength 实现了那三个方法，sort.Sort 就认可它了，可以开始工作。

排序完成后，因为底层数组是同一个，fruits 这个变量看到的内容也已经排好序了。最后 fmt.Println(fruits) 输出的就是 [kiwi peach banana]。

---

**和上一关 sort.Ints 对比——有什么区别？**

上一关用的 sort.Ints([]int{...})，那是标准库已经帮你把 int 切片的 Len/Swap/Less 都实现好了，你直接调用就行，方便但不灵活。这一关的 sort.Sort + 自定义类型，是「自己动手」的版本：麻烦一点，但可以实现任意规则。

打个游戏类比：sort.Ints 是游戏里的「自动战斗」，简单省事；sort.Sort + 自定义接口是「手动操作」，你可以控制每一个细节，打出更漂亮的组合技。

---

**总结一下这一关的核心链路**

1. 用 `type byLength []string` 创建命名类型，才能在上面挂方法
2. 为 byLength 实现 Len、Swap、Less 三个方法，满足 sort.Interface 接口
3. 用 `byLength(fruits)` 做类型转换，让 sort.Sort 认识它
4. sort.Sort 内部调用这三个方法完成排序，fruits 底层数组被就地修改

记住鸭子类型这句话：**你不需要显式声明「我实现了某个接口」，只要你把接口要求的方法都实现了，Go 编译器自动认可你**。这是 Go 接口设计最优雅的地方，也是你以后写 Go 代码会反复用到的核心思维。师兄我每次用这个特性都觉得特别爽，希望你也能感受到这种简洁的力量。

🏃 跑一下试试

```bash
go run sorting-by-functions.go
```

```text
[kiwi peach banana]
```

// kiwi=4个字母、peach=5个字母、banana=6个字母
// Less 里写的是 len(s[i]) < len(s[j])，所以短的排前面
// 最终从短到长：4 < 5 < 6，输出顺序就是 kiwi → peach → banana

💡 师兄的碎碎念

这一关用的是「实现接口」那套老办法，定义一个新类型、写三个方法，确实清晰，但说实话写起来挺啰嗦的——光是为了按长度排个字符串切片，就要专门搞一个 `byLength` 类型，再挂三个方法上去，代码一下子多了好多行。

其实日常工作里更常见的写法是 `sort.Slice()`，直接传切片和一个匿名函数，一行闭包就搞定了：

```go
sort.Slice(fruits, func(i, j int) bool {
    return len(fruits[i]) < len(fruits[j])
})
```

不用定义任何新类型，逻辑写在原地，看起来干净多了。不过要注意，`sort.Slice` 是不稳定排序，如果你有「相等的元素要保持原来的顺序」这种需求，得换成 `sort.SliceStable()`，别踩坑。

Go 1.21 之后还出了 `slices.SortFunc`，泛型加持，连切片类型都推断好了，是目前最现代的写法，新项目推荐优先用这个。

三种方法各有场景：接口写法适合大型项目里封装复用、逻辑复杂的时候；`sort.Slice` 适合快速一次性排序；`slices.SortFunc` 是 Go 1.21+ 的新宠。师兄建议你三种都跑一遍，心里有数，面试考到不慌。

🎓 知识点清单

• sort.Interface 三件套：Len() / Less() / Swap()，实现这三个方法就能让任意类型参与排序
• 自定义排序的核心是 Less(i, j int) bool，返回 true 表示 i 排在 j 前面
• sort.Sort(data Interface) 接收实现了接口的值，调用你写的逻辑排序
• sort.Slice(slice, func(i, j int) bool) 是更轻便的写法，不用定义新类型，一行闭包搞定
• Go 1.21+ 引入 slices.SortFunc，泛型加持，类型更安全，是最新推荐姿势
• sort.Stable() 保证相等元素的原始顺序不变，需要稳定性时别忘了它
• 字符串长度用 len() 取字节数，ASCII 场景够用；Unicode 场景记得换 utf8.RuneCountInString()

➡️ 下一关

第 45 关「Panic」——Go 里没有异常，但有 panic。程序跑着跑着突然炸了？别慌，下一关带你认识 panic 是个啥、啥时候该用、啥时候绝对不该用，还有 recover 怎么兜底。准备好了吗？冲！
