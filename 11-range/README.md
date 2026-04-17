# 第 11 关：range 遍历（师兄带你学 Go）

## 🎯 这一关你会学到

- `range` 关键字的基本用法和核心思想
- 如何用 `range` 遍历 slice、数组、map 和字符串
- 空白标识符 `_` 的妙用——忽略你不需要的返回值
- `range` 在字符串上迭代时返回的是 Unicode 码点，而不是字节
- 什么时候只取索引、只取值，或者两个都要

---

## 🤔 先想一个问题

在你学 Go 之前，可能写过这样的 C 风格循环：

```c
for (int i = 0; i < len; i++) {
    sum += arr[i];
}
```

或者 Python 风格的：

```python
for i, v in enumerate(my_list):
    print(i, v)
```

Go 语言有没有更优雅的方式来做这件事呢？答案就是 `range`。它让遍历变得简洁、安全，而且统一适用于多种数据结构。带着这个问题，我们进入今天的学习！

---

## 📖 看代码

```go
// _range_ 用于迭代各种各样的数据结构。
// 让我们来看看如何在我们已经学过的数据结构上使用 `range`。

package main

import "fmt"

func main() {

	// 这里我们使用 `range` 来对 slice 中的元素求和。
	// 数组也可以用这种方法初始化并赋初值。
	nums := []int{2, 3, 4}
	sum := 0
	for _, num := range nums {
		sum += num
	}
	fmt.Println("sum:", sum)

	// `range` 在数组和 slice 中提供对每项的索引和值的访问。
	// 上面我们不需要索引，所以我们使用 _空白标识符_ `_` 将其忽略。
	// 实际上，我们有时候是需要这个索引的。
	for i, num := range nums {
		if num == 3 {
			fmt.Println("index:", i)
		}
	}

	// `range` 在 map 中迭代键值对。
	kvs := map[string]string{"a": "apple", "b": "banana"}
	for k, v := range kvs {
		fmt.Printf("%s -> %s\n", k, v)
	}

	// `range` 也可以只遍历 map 的键。
	for k := range kvs {
		fmt.Println("key:", k)
	}

	// `range` 在字符串中迭代 unicode 码点(code point)。
	// 第一个返回值是字符的起始字节位置，然后第二个是字符本身。
	for i, c := range "go" {
		fmt.Println(i, c)
	}
}
```

---

## 🔍 师兄给你逐行拆

好，代码不长，但每一行都有值得深挖的东西。师兄带你一行一行地过。

### 第一段：用 range 遍历 slice 并求和

```go
nums := []int{2, 3, 4}
sum := 0
for _, num := range nums {
    sum += num
}
fmt.Println("sum:", sum)
```

这里 `nums` 是一个包含三个整数的 slice。`for _, num := range nums` 这行代码是核心。

拆开来看：
- `range nums`：对 slice `nums` 进行迭代。每次迭代，`range` 都会返回两个值：**当前元素的索引**和**当前元素的值**。
- `_`（空白标识符）：我们不需要索引，所以用 `_` 把它丢掉。在 Go 里，如果你声明了一个变量但不使用，编译器会报错。`_` 就是专门用来"接收但忽略"某个值的，它告诉编译器：我知道这里有个值，但我不关心它。
- `num`：接收当前元素的值，每次循环都会被赋值为 `nums[i]`。

所以这段代码等价于：

```go
for i := 0; i < len(nums); i++ {
    sum += nums[i]
}
```

但 `range` 的写法更简洁、更安全，不用担心越界问题。

运行结果：`sum: 9`，也就是 2 + 3 + 4 = 9，完全正确。

### 第二段：带索引的 range 遍历

```go
for i, num := range nums {
    if num == 3 {
        fmt.Println("index:", i)
    }
}
```

这次我们同时接收了索引 `i` 和值 `num`。我们想找到值等于 3 的元素，并打印它的索引。

`nums` 是 `[]int{2, 3, 4}`，所以：
- 第一次循环：`i=0, num=2`，不满足条件
- 第二次循环：`i=1, num=3`，满足！打印 `index: 1`
- 第三次循环：`i=2, num=4`，不满足条件

输出：`index: 1`

这个用法非常实用。比如你要找某个元素的位置，或者根据下标做不同的处理，就需要同时拿到索引和值。

### 第三段：range 遍历 map

```go
kvs := map[string]string{"a": "apple", "b": "banana"}
for k, v := range kvs {
    fmt.Printf("%s -> %s\n", k, v)
}
```

这里 `kvs` 是一个 map，键和值都是字符串类型。`range` 用在 map 上时，每次迭代返回**键**和**值**（注意不是索引和值了）。

`k` 接收键，`v` 接收对应的值。`fmt.Printf("%s -> %s\n", k, v)` 格式化打印出键值对关系。

**注意**：map 的遍历顺序在 Go 中是**不确定的**！每次运行程序，输出的顺序可能不同。这是 Go 语言的设计决策，因为 map 的底层实现是哈希表，遍历顺序依赖于哈希结果。所以如果你需要按顺序输出，必须先把键取出来排序，再遍历。

输出示例（顺序可能不同）：
```
a -> apple
b -> banana
```

### 第四段：只遍历 map 的键

```go
for k := range kvs {
    fmt.Println("key:", k)
}
```

这里只写了一个变量 `k`，没有用 `k, v`。当你只需要 map 的键，不需要值时，可以只写一个变量。Go 会把每次迭代的键赋给它。

等价于 Python 中的 `for k in my_dict:`。

同样，由于 map 无序，输出顺序不固定：
```
key: a
key: b
```

### 第五段：range 遍历字符串

```go
for i, c := range "go" {
    fmt.Println(i, c)
}
```

这是最容易让初学者困惑的部分！

`range` 遍历字符串时，不是按字节遍历，而是按 **Unicode 码点（rune）**遍历。每次迭代：
- `i`：该字符在字符串中的**字节偏移量**（起始位置）
- `c`：该字符对应的 **Unicode 码点值**（整数类型 rune，也就是 int32）

字符串 `"go"` 包含两个字符：
- `'g'`：Unicode 码点是 103，字节位置是 0
- `'o'`：Unicode 码点是 111，字节位置是 1

所以输出是：
```
0 103
1 111
```

这里打印出来的是数字而不是字母，因为 `c` 的类型是 `rune`（int32），`fmt.Println` 会把它当整数打印。如果你想打印字符本身，可以用 `fmt.Printf("%c\n", c)`。

**为什么要用 Unicode 码点而不是字节？**

因为 Go 字符串天然支持 UTF-8 编码，中文字符会占用多个字节（通常是 3 字节）。如果按字节遍历，中文会被拆散。用 `range` 遍历字符串，Go 会自动处理多字节字符，确保你每次拿到的是一个完整的 Unicode 字符，非常安全。

---

## 🏃 跑一下试试

```bash
$ go run range.go
sum: 9
index: 1
a -> apple
b -> banana
key: a
key: b
0 103
1 111
```

注意：map 相关的输出（`a -> apple`、`b -> banana`、`key: a`、`key: b`）的顺序每次运行可能不同，这是正常现象！

---

## 💡 师兄的碎碎念

**关于空白标识符 `_`**

`_` 是 Go 里的一个特殊标识符，读作"空白标识符"或"下划线"。它的作用是"接收并丢弃"。Go 编译器规定：声明的变量必须被使用，否则编译报错。但如果你用 `_`，就相当于告诉编译器：我知道这里有个值，但我不需要它，帮我忽略掉。

你会在很多地方看到它：
- `for _, v := range slice`：不要索引
- `result, _ := someFunc()`：不要错误
- `import _ "pkg"`：只执行包的 init，不使用包内标识符

**关于 map 遍历顺序**

很多 Go 初学者会在面试中被问到：为什么 map 遍历顺序不固定？

原因：Go 的 map 底层是哈希表，键的存储位置取决于哈希值。为了防止程序员依赖某个特定的遍历顺序（导致脆弱的代码），Go 语言在运行时会对 map 的起始遍历位置做随机化处理。这是一个刻意的设计决策。

---

## 🎓 这一关的知识点清单

| 知识点 | 说明 |
|--------|------|
| `for _, v := range slice` | 遍历 slice，忽略索引 |
| `for i, v := range slice` | 遍历 slice，同时获取索引和值 |
| `for k, v := range map` | 遍历 map 的键值对 |
| `for k := range map` | 只遍历 map 的键 |
| `for i, c := range string` | 遍历字符串的 Unicode 码点 |
| 空白标识符 `_` | 忽略不需要的返回值，避免编译器报错 |
| map 遍历无序 | Go map 遍历顺序是随机的，每次可能不同 |
| rune 类型 | range 遍历字符串时，字符值类型是 rune（int32） |

---

## ➡️ 下一关

掌握了 `range` 遍历，你现在可以优雅地处理各种数据结构了！接下来我们要学习 Go 中最基础也最重要的概念之一——**函数**。

[下一关：函数 →](../12-functions/)
