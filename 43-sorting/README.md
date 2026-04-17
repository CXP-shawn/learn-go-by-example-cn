# 第 43 关：排序（师兄带你学 Go）

🎯 这一关你会学到
• 用 sort.Strings / sort.Ints 对内置类型切片做一行排序
• 理解「原地排序」的含义：排完直接改原切片，不会冒出新切片
• 用 IntsAreSorted 快速检查一个切片是不是已经升序排好了
• 感受 sort 包底层靠「接口」驱动的设计思路，为下一关自定义排序打好基础

🤔 先想一个问题

你有没有玩过「炉石传说」这种卡牌游戏？每次对手打出一堆随从，你脑子里会自动按照攻击力从低到高排列一遍，想好先打哪个。这个「在脑子里重新排列」的过程，就是排序。但问题来了——你是把对手的随从「在原地重新站队」，还是「另开一张纸把顺序抄下来」？这两种做法的结果看起来一样，但成本差别可大了。Go 的 sort 包默认走的是第一种：直接在原地把队伍排好，省内存、省事、不废话。这一关咱们就把这事彻底搞清楚。

## 📖 看代码

```go
// Go 的 `sort` 包实现了内建及用户自定义数据类型的排序功能。
// 我们先来看看内建数据类型的排序。

package main

import (
	"fmt"
	"sort"
)

func main() {

	// 排序方法是针对内置数据类型的；
	// 这是一个字符串排序的例子。
	// 注意，它是原地排序的，所以他会直接改变给定的切片，而不是返回一个新切片。
	strs := []string{"c", "a", "b"}
	sort.Strings(strs)
	fmt.Println("Strings:", strs)

	// 一个 `int` 排序的例子。
	ints := []int{7, 2, 4}
	sort.Ints(ints)
	fmt.Println("Ints:   ", ints)

	// 我们也可以使用 `sort` 来检查一个切片是否为有序的。
	s := sort.IntsAreSorted(ints)
	fmt.Println("Sorted: ", s)
}
```

🔍 师兄给你逐行拆

**先说 import**

代码顶部引入了两个包：fmt 和 sort。fmt 老朋友了，负责打印。sort 是这关的主角——它是 Go 标准库自带的排序工具包，不用你自己写冒泡、快排，直接拿来用就行。这年头谁还手写排序啊，sort 包就是你的外卖小哥，你只管点单，它负责送到你手边。

**什么是切片（Slice）？先定义一下**

切片（Slice）是 Go 里最常用的动态数组结构。你可以把它理解成一个可以伸缩的购物车：里面能放任意数量的同类商品，可以随时往里加、往外取。和数组不一样，切片的长度不是固定的。代码里的 strs 和 ints 都是切片，一个装字符串，一个装整数。

**sort.Strings(strs) 这行在干嘛？**

```go
strs := []string{"c", "a", "b"}
sort.Strings(strs)
fmt.Println("Strings:", strs)
```

首先，strs 初始化成了 ["c", "a", "b"]，顺序是乱的。然后调用 sort.Strings(strs)，这个函数会按照字母表（也就是字典序）从小到大排列字符串。执行完之后，strs 变成了 ["a", "b", "c"]。

这里有个关键点要敲黑板：sort.Strings 没有返回值！你没看错，它不会还给你一个「排好序的新切片」，它是直接在原来那个 strs 上动手术。这就是所谓的「原地排序」。

**什么是原地排序（In-place Sorting）？给你定义清楚**

原地排序（In-place Sorting）的意思是：排序操作直接在原始数据上进行，不额外开辟一块同等大小的内存来存放结果。排完了，原来那块内存里的数据顺序就变了，函数本身不返回任何新数据。

用生活类比就是：你把书架上的书直接重新摆顺序，而不是另买一个新书架再照着新顺序摆一遍。前者省地方，后者费钱。

这意味着什么？意味着你调用完 sort.Strings(strs) 之后，原来那个 strs 变量就已经是排好序的了。你不需要写 strs = sort.Strings(strs)，这样写反而会报错，因为 sort.Strings 根本没有返回值让你接。很多新手在这里踩坑，觉得「我排序了怎么没生效」，结果发现自己在等一个根本不存在的返回值。

**sort.Ints(ints) 是一个道理**

```go
ints := []int{7, 2, 4}
sort.Ints(ints)
fmt.Println("Ints:   ", ints)
```

ints 初始是 [7, 2, 4]，调用 sort.Ints 之后变成 [2, 4, 7]。同样是原地排序，同样是升序（从小到大），同样没有返回值。

这里要说一下，sort 包对内置类型的排序默认全部是「升序」，也就是从小到大。字符串按字典序从 a 到 z，整数按数值从小到大。如果你想要降序，sort 包也支持，不过要用别的姿势，那是下一关的事情，这关先记住默认是升序。

**sort.IntsAreSorted 是什么鬼？**

```go
s := sort.IntsAreSorted(ints)
fmt.Println("Sorted: ", s)
```

sort.IntsAreSorted 是一个「检查员」函数，它会遍历整个整数切片，看看是不是从头到尾每个元素都不比下一个大。如果是，就返回 true；只要有一个地方乱了，就返回 false。

在这段代码里，ints 在上一步已经被 sort.Ints 排好序了，所以 s 的值是 true，打印出来就是 Sorted: true。

你可能会问：这个函数有什么用？我都排完序了还用检查吗？其实在实际项目里，你经常会收到别人传过来的数据，你不确定对方有没有帮你排好序，这时候先用 IntsAreSorted 检查一下，能节省一次不必要的排序操作。就像你收到快递之前先看看物流状态，确认「已送达」就不用再催了。

**原地排序 vs 返回新切片，再对比一次**

很多语言（比如 Python）的排序有两个版本：一个是原地排序（list.sort()），一个是返回新列表（sorted(list)）。Go 的 sort 包走的是「原地」路线，好处是内存效率高，坏处是如果你还想保留原来没排序前的数据，就得自己提前备份一份。

比如你想同时保留排序前和排序后的数据：

```go
original := []int{7, 2, 4}
copy_slice := make([]int, len(original))
copy(copy_slice, original)  // 手动复制一份
sort.Ints(copy_slice)       // 只排复制的那份
// original 还是 [7, 2, 4]，copy_slice 变成 [2, 4, 7]
```

这不是什么奇怪操作，项目里经常用到，记住就行。

**稳定排序是什么？顺带提一下**

稳定排序（Stable Sorting）的定义是：如果两个元素的排序依据（比如数值、字母）相等，排序后它们的相对顺序和排序前保持一致。比如两个人都叫「张三」，稳定排序保证他俩的先后顺序不会因为排序而调换。

sort.Ints 和 sort.Strings 背后用的算法不保证稳定性。如果你需要稳定排序，Go 提供了 sort.Stable 方法，不过那是进阶内容，这关先了解概念就好。

**sort 包底层靠接口驱动，给你点拨一下**

sort 包能对 int、string 排序，还能对你自己定义的结构体排序，靠的不是写了一大堆重复代码，而是 Go 的「接口（Interface）」机制。接口就是一份「能力协议」：只要你的类型实现了 Len()、Less()、Swap() 这三个方法，sort 包就能帮你排序。

string 和 int 的排序之所以这么简单，是因为 Go 标准库已经帮你把这三个方法实现好了，封装成了 sort.Strings 和 sort.Ints 这样的快捷函数。这一关你先感受一下「用」的感觉，下一关咱们就来自己实现接口，搞自定义排序，那才是重头戏。

**小结**

这一关核心就三件事：第一，sort.Strings 和 sort.Ints 是原地排序，改的是原切片，没有返回值；第二，默认升序，想要降序得换姿势；第三，IntsAreSorted 是用来检查的，不是排序的，别搞混了。理解了这三点，这关就过了。

```bash
go run sorting.go
```

```text
Strings: [a b c]
Ints:    [2 4 7]
Sorted:  true
```

逐行来看一下：

- `Strings: [a b c]` — 原来是 `["c", "a", "b"]`，调用 `sort.Strings()` 之后按字典序排，a < b < c，所以变成 `[a b c]`
- `Ints:    [2 4 7]` — 原来是 `[7, 2, 4]`，调用 `sort.Ints()` 之后按升序排，2 < 4 < 7，所以变成 `[2 4 7]`
- `Sorted:  true` — 用 `sort.IntsAreSorted()` 检查 `[2 4 7]` 是不是已经有序，显然是升序排好的，所以返回 `true`

💡 师兄的碎碎念

Go 的 sort 包看起来简单，背后其实有一套优雅的设计。它的核心是 sort.Interface 这个接口，只要你的类型实现了 Len()、Less()、Swap() 三个方法，sort.Sort() 就能帮你排序，完全不挑食。sort.Ints()、sort.Strings() 这些快捷函数，底层也是把 []int、[]string 包装成实现了该接口的类型再调用 sort.Sort()，帮你省去了写样板代码的麻烦。

默认排序是升序，这点要记牢。想要降序？别自己瞎写 Less 反过来，直接用 sort.Sort(sort.Reverse(sort.IntSlice(nums)))，sort.Reverse() 会把 Less 的结果取反，干净利落。

值得一提的是，Go 1.21 推出了 slices 标准包，slices.Sort()、slices.SortFunc() 借助泛型实现，类型安全、写法更简洁，编译器能帮你抓更多类型错误。如果你的项目已经用上 Go 1.21+，新代码优先选这套写法，老的 sort 包方式还能用但渐渐会变成「历史风格」。

最后有个坑师兄要提醒你：排序是原地操作，函数一执行完，原 slice 的元素顺序就变了。如果程序里还有别的变量引用着同一个底层数组，它们看到的数据也会跟着变。所以排序之前，如果你还需要保留原始顺序，记得先 copy 一份出来，别到时候找半天 bug 才发现是被排序「悄悄改了」。

🎓 知识点清单

• sort.Ints() / sort.Strings() / sort.Float64s() 是最常用的内置快捷排序，直接对 slice 原地升序排列
• sort 包底层依赖 sort.Interface 接口，包含 Len()、Less()、Swap() 三个方法，所有排序逻辑围绕这三个方法展开
• 默认排序均为升序；想要降序，用 sort.Sort(sort.Reverse(sort.IntSlice(nums))) 包一层即可
• sort.Search() 提供二分查找能力，但要求 slice 已经有序，否则结果不可预期
• Go 1.21+ 引入 slices 标准包，slices.Sort()、slices.SortFunc() 是更现代的写法，类型安全且性能更好，新项目推荐优先使用
• 排序是原地操作（in-place），函数返回后原 slice 的顺序已被改变，如果其他地方还持有同一底层数组的引用，数据会一起变动，使用前注意 copy 备份
• sort.Slice() 允许传入匿名函数定义比较逻辑，比实现完整接口更轻量，适合一次性排序需求

➡️ 下一关

会用 sort.Ints() 排数字只是第一步，真实业务里往往要按结构体的某个字段排、甚至多字段组合排——这就轮到「使用函数自定义排序 sorting-by-functions」登场了。第 44 关我们一起用 sort.Slice() 和 slices.SortFunc() 把排序玩出花来，走起！
