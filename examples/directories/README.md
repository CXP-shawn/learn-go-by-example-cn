# 第 66 关：目录（师兄带你学 Go）

## 🎯 这一关你会学到

掌握 Go 里目录操作的常用武器：os.Mkdir、os.MkdirAll、os.RemoveAll、os.ReadDir、os.Chdir，以及 filepath.Walk/WalkDir 递归遍历目录树的姿势。

## 🤔 先想一个问题

写程序总要和文件系统打交道——临时目录、日志目录、上传文件落盘、递归扫描目录找配置……这些场景都逃不开「建目录、列目录、换目录、递归走目录」这几板斧。本关把 os 包和 filepath.Walk 里最常用的函数一次性讲透。

## 📖 看代码

```go
// 对于操作文件系统中的 *目录* ，Go 提供了几个非常有用的函数。

package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {

	// 在当前工作目录下，创建一个子目录。
	err := os.Mkdir("subdir", 0755)
	check(err)

	// 创建这个临时目录后，一个好习惯是：使用 `defer` 删除这个目录。
	// `os.RemoveAll` 会删除整个目录（类似于 `rm -rf`）。
	defer os.RemoveAll("subdir")

	// 一个用于创建临时文件的帮助函数。
	createEmptyFile := func(name string) {
		d := []byte("")
		check(os.WriteFile(name, d, 0644))
	}

	createEmptyFile("subdir/file1")

	// 我们还可以创建一个有层级的目录，使用 `MkdirAll` 函数，并包含其父目录。
	// 这个类似于命令 `mkdir -p`。
	err = os.MkdirAll("subdir/parent/child", 0755)
	check(err)

	createEmptyFile("subdir/parent/file2")
	createEmptyFile("subdir/parent/file3")
	createEmptyFile("subdir/parent/child/file4")

	// `ReadDir` 列出目录的内容，返回一个 `os.DirEntry` 类型的切片对象。
	c, err := os.ReadDir("subdir/parent")
	check(err)

	fmt.Println("Listing subdir/parent")
	for _, entry := range c {
		fmt.Println(" ", entry.Name(), entry.IsDir())
	}

	// `Chdir` 可以修改当前工作目录，类似于 `cd`。
	err = os.Chdir("subdir/parent/child")
	check(err)

	// 当我们列出 *当前* 目录，就可以看到 `subdir/parent/child` 的内容了。
	c, err = os.ReadDir(".")
	check(err)

	fmt.Println("Listing subdir/parent/child")
	for _, entry := range c {
		fmt.Println(" ", entry.Name(), entry.IsDir())
	}

	// `cd` 回到最开始的地方。
	err = os.Chdir("../../..")
	check(err)

	// 当然，我们也可以遍历一个目录及其所有子目录。
	// `Walk` 接受一个路径和回调函数，用于处理访问到的每个目录和文件。
	fmt.Println("Visiting subdir")
	err = filepath.Walk("subdir", visit)
}

// `filepath.Walk` 遍历访问到每一个目录和文件后，都会调用 `visit`。
func visit(p string, info os.FileInfo, err error) error {
	if err != nil {
		return err
	}
	fmt.Println(" ", p, info.IsDir())
	return nil
}
```

## 🔍 师兄给你逐行拆

好，第 66 关，今天聊目录操作。说白了就是你在代码里用程序手动"右键新建文件夹"那一套，只不过用 os 包来搞定。

---

**os.Mkdir：老实人，一次只建一层**

os.Mkdir 是最基础的目录创建函数，用法长这样：

```go
err := os.Mkdir("mydir", 0755)
```

它有个脾气：父目录必须已经存在，否则直接报错给你看。比如你想建 `a/b/c`，但 `a/b` 还不存在，Mkdir 就会甩你一脸错误，一点都不客气。所以它适合那种"我知道父目录肯定在"的场景，老老实实用就行。

第二个参数 `0755` 是 Unix 权限位，注意那个**开头的 0**，这是 Go（和 C）里表示八进制数的写法。没有这个 0，Go 会当成十进制 755，完全是两码事，这个坑踩过的同学应该印象深刻。`0755` 展开是 `rwxr-xr-x`：目录所有者可读可写可执行（进入），同组用户和其他人可读可执行但不能写。这是目录的标准权限，基本上你闭眼写 0755 就对了。

---

**os.MkdirAll：体贴的那种，mkdir -p**

MkdirAll 就厚道多了，等价于 shell 里的 `mkdir -p`：

```go
err := os.MkdirAll("a/b/c", 0755)
```

`a`、`a/b`、`a/b/c` 一路全给你建好，中间哪层不存在就顺手创建，已经存在的层它也不报错，直接跳过。写临时目录、测试目录、日志目录这种场景，基本上都该用 MkdirAll 而不是 Mkdir，省心。

---

**os.RemoveAll：rm -rf，用完即焚**

os.RemoveAll 就是代码里的 `rm -rf`，不管里面有多少层子目录和文件，一口气全删光：

```go
err := os.RemoveAll("a")
```

它有一个超级经典的搭配用法，就是 `defer`：

```go
os.MkdirAll("tmpdir", 0755)
defer os.RemoveAll("tmpdir")
```

建完目录，马上 defer 一个 RemoveAll，函数结束时自动清理，测试代码里大量使用这个模式，叫做"用完即焚"。你写单元测试、集成测试的时候不用手动收拾烂摊子，Go 测试跑完自己擦屁股，非常优雅。顺带一提，如果路径不存在，RemoveAll 也不会报错，行为比 Remove 宽容得多。

---

**os.WriteFile：顺手建个文件做测试**

有时候你建了目录，还想在里面放个文件验证一下，os.WriteFile 最方便：

```go
os.WriteFile("tmpdir/hello.txt", []byte("hi"), 0644)
```

注意这里权限是 `0644`，即 `rw-r--r--`：所有者可读可写，其他人只读，没有执行位。文件一般都用 0644，目录用 0755，两个数字记住就行。**文件不需要执行权限，目录需要执行权限才能进入**，这是关键区别，别搞混。

---

**小结**

一句话速记：单层用 Mkdir，多层用 MkdirAll，清理用 RemoveAll 配 defer，文件权限 0644，目录权限 0755，开头那个 0 是八进制，绝对不能省。目录操作就这几板斧，熟了写起来行云流水。

好，师弟师妹们，第 66 关我们聊目录操作。上一段把 os.ReadDir 的基本用法过了一遍，这段我们往深里挖，把几个容易踩坑的地方都说透。

**os.ReadDir 返回的是什么？**

os.ReadDir 给你返回一个 os.DirEntry 的切片，切片里每个元素代表目录里的一个条目——文件也好，子目录也好，符号链接也好，统统在里面。每个 DirEntry 有四个方法：Name 返回文件名字符串（注意只是名字，不含路径）；IsDir 告诉你这东西是不是目录；Type 返回文件模式里的类型位，可以用来判断是不是符号链接；Info 返回一个 os.FileInfo，但这个方法是

## 🏃 跑一下试试

跑 go run directories.go，程序先用 os.MkdirAll 建出 subdir/parent/child 三层目录，再用 os.Create 在各层放好 file1–file4。os.ReadDir("subdir/parent") 列出两条：child（IsDir=true）、file2、file3（均 false）。os.Chdir("subdir") 切换工作目录后，ReadDir("parent/child") 只看到 file4。最后 filepath.Walk("subdir") 按字典序打印全部节点：subdir、subdir/file1、subdir/parent、subdir/parent/child、subdir/parent/child/file4、subdir/parent/file2、subdir/parent/file3。运行结束时 defer os.RemoveAll("subdir") 自动清理，工作目录也 defer 切回原处，运行后本地不留任何痕迹。

## 💡 师兄的碎碎念

💡 几个要点：① os.Mkdir 只能建单层，父目录不存在直接报 ENOENT，所以多层结构必须用 os.MkdirAll；② MkdirAll 是幂等的，目录已存在不报错，CI 脚本里反复调用很安全；③ os.Chdir 修改的是整个进程的工作目录，多 goroutine 并发时会互相干扰，生产代码里尽量用绝对路径或 os.DirFS 替代；④ os.ReadDir 直接返回已排序的 DirEntry 切片，比老式 File.Readdir 更轻量，不会一次性把所有 FileInfo 读进内存；⑤ filepath.Walk 回调里 return filepath.SkipDir 可以跳过整个子树，遇到 vendor 或 node_modules 这类目录时非常实用，Go 1.16+ 的 filepath.WalkDir 性能更好，优先选它。

## 🎓 知识点清单

- os.MkdirAll("subdir/parent/child", 0755) 一次性建好三层，中间层不存在也没关系
- os.ReadDir 返回 []os.DirEntry，每条可调 .Name() 和 .IsDir()
- os.Chdir("subdir") 之后相对路径都基于新 cwd，记得 defer 回来
- filepath.Walk 按字典序深度优先遍历，回调签名 func(path string, info fs.FileInfo, err error) error
- Walk 里对目录本身也会触发一次回调，info.IsDir() 可区分
- defer os.RemoveAll("subdir") 保证测试完不留垃圾，即使中途 panic 也执行
- err != nil 每步都要检查，Mkdir 失败后续操作全部无意义

## ➡️ 下一关

目录操作搞定了，下一关我们看「临时文件和目录 (temporary-files-and-directories)」——os.CreateTemp 和 os.MkdirTemp 帮你自动生成唯一名字，省去手动命名和冲突烦恼，继续跟上！
