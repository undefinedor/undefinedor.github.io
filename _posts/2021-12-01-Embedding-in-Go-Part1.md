---
layout: post
title:  "Golang中的嵌套:结构体嵌套"
date:   2021-12-01 23:00:00 +0800
categories: Golang
---

# Golang中的嵌套：结构体嵌套

众所周知，Golang不支持经典意义上的继承，它鼓励通过组合来实现类型拓展，这并不是Golang独有的概念，组合优于继承是OOP的一个原则，在很多设计模式书籍的第一张会被提及。

嵌套是Golang一个非常重要的特性，使得组合更加的方面与有用。虽然Golang力求简单，但嵌套却是一个略显复杂的地方。在这篇文章中，我们会讲解Golang中不同类型的嵌套，并提供一些真实的代码（大部分来自于标准库）

在Golang中有三种类型的嵌套

1. 结构体嵌套
2. 接口嵌套
3. 结构体中的接口

## 结构体嵌套

首先，我们通过一个示例来呈现一个结构体嵌套在另一个结构体中：

```golang
type Base struct {
  b int
}


type Container struct {     // Container 是被嵌入的结构体
  Base                      // Base 嵌入进来的结构体
  c string
}
```
现在，`Container`的实例也具有了`b`（来自于`Base`）这个字段。在规范中，我们称这种字段为*提升*字段。我们可以像访问`c`字段那样访问`b`字段：
```golang
co := Container{}
co.b = 1
co.c = "string"
fmt.Printf("co -> {b: %v, c: %v}\n", co.b, co.c)
```
然而，当使用结构体字面量时，我们必须将嵌入结构体作为一个整体进行初始化，而不是它的字段。提升字段不可以在声明结构体字面量的时候作为被嵌入结构体的字段名：
```golang
co := Container{Base: Base{b: 10}, c: "foo"}
fmt.Printf("co -> {b: %v, c: %v}\n", co.b, co.c)
```
注意，通过`co.b`的方式是一种语法糖；我们也可以通过更明确的方式`co.Base.b`访问`b`字段

## 方法

结构体嵌套同样适用于方法。设想下，`Base`上有这么一个方法：
```golang
func (base Base) Describe() string {
  return fmt.Sprintf("base %d belongs to us", base.b)
}
```
我们现在可以在`Container`的实例上调用它，就好像它也有这个方法一样：
```golang
fmt.Println(cc.Describe())
```
为了更好的理解调用的机制，设想下，`Container`有一个`Base`类型的字段,有一个`Describe`来转发调用：
```golang
type Container struct {
  base Base
  c string
}

func (cont Container) Describe() string {
  return cont.base.Describe()
}
```

在这个`Container`上调用`Describe`的效果类似于我们原来使用嵌入的效果。

这个例子还呈现了嵌入字段方法一个重要的微妙之处。当`Base`的`Describe`被调用时，无论通过哪个嵌入结构调用它,它都会传递一个`Base`接受者(方法定义最左边的`(...)`)。这与Python和C++等其他语言中的继承不同，继承的方法获得对它们被调用的子类的引用。这是Golang结构体嵌套与传统继承之间重要的区别。

## 嵌入字段的屏蔽

如果嵌套结构体有一个字段`x`并嵌入一个也有一个字段`x`的结构体，会发生什么？在这种情况下，当通过`embedding struct`访问`x`时，我们得到了`embedding struct`的字段；嵌入结构的`x`被屏蔽。

示例如下：
```golang
type Base struct {
  b   int
  tag string
}

func (base Base) DescribeTag() string {
  return fmt.Sprintf("Base tag is %s", base.tag)
}

type Container struct {
  Base
  c   string
  tag string
}

func (co Container) DescribeTag() string {
  return fmt.Sprintf("Container tag is %s", co.tag)
}
```

当我们这样书写：
```golang
b := Base{b: 10, tag: "b's tag"}
co := Container{Base: b, c: "foo", tag: "co's tag"}

fmt.Println(b.DescribeTag())
fmt.Println(co.DescribeTag())
```
它会打印：
```golang
Base tag is b's tag
Container tag is co's tag
```
注意，在访问`co.tag`时，我们得到的是`Container`的`tag`字段，而不是通过`Base`的`shadowing`进来的。不过，我们可以使用`co.Base.tag`显式访问另一个。

## 例子：`sync.Mutex`
以下示例均来自Golang标准库。

Golang中一个经典的结构体嵌套例子就是`sync.Mutex`。[crypto/tls/common.go](https://golang.org/src/crypto/tls/common.go)中的`lruSessionCache`:
```golang
type lruSessionCache struct {
  sync.Mutex
  m        map[string]*list.Element
  q        *list.List
  capacity int
}
```
注意`sync.Mutex`的嵌入；现在假设`cache`是`lruSessionCache`的一个实例，那么我们就可以调用`cache.Lock`与`cache.Unlock`。这在一些场景下是非常有用的，如果锁定是结构的公共`API`的一部分，那么将互斥锁嵌入很方便，并且不需要显式转发方法。

但是，锁可能仅由结构的方法在内部使用，而不向其用户公开。在这种情况下，我不会嵌入`sync.Mutex`，而是将其设为未导出的字段（如`mu sync.Mutex`）。

## 例子：elf.FileHeader
`sync.Mutex`的嵌入很好地证明了结构体嵌套嵌入以获得新的行为。另一个例子涉及数据的嵌入。在[`debug/elf/file.go`](https://golang.org/src/debug/elf/file.go)可以找到描述`ELF`文件的结构：
```golang
// A FileHeader represents an ELF file header.
type FileHeader struct {
  Class      Class
  Data       Data
  Version    Version
  OSABI      OSABI
  ABIVersion uint8
  ByteOrder  binary.ByteOrder
  Type       Type
  Machine    Machine
  Entry      uint64
}

// A File represents an open ELF file.
type File struct {
  FileHeader
  Sections  []*Section
  Progs     []*Prog
  closer    io.Closer
  gnuNeed   []verneed
  gnuVersym []byte
}
```

`elf`包开发人员可以直接在`File`中列出所有头字段，但是将其放在单独的结构中是自记录数据分区的一个很好的例子。用户代码可能希望与`File`分开初始化和操作文件头，嵌入设计使这很自然。

类似的例子可以在[`compress/gzip/gunzip.go`](https://golang.org/src/compress/gzip/gunzip.go) 中找到，其中`gzip.Reader`嵌入了`gzip.Header`。这是一个非常好的嵌入数据重用示例，因为`gzip.Writer`还嵌入了`gzip.Header`，这有助于避免复制粘贴。

## 例子：bufio.ReadWriter

由于嵌入结构“继承”（但不是在经典意义上）嵌入结构的方法，嵌入可以成为实现接口的有用工具。

思考下`bufio`包，它有个类型`bufio.Reader`。指向此类型的指针实现`io.Reader`接口。这同样适用于实现 io.Writer 的 *bufio.Writer。我们如何创建实现`io.ReadWriter`接口的`bufio`类型？

使用嵌套非常简单：
```golang
type ReadWriter struct {
  *Reader
  *Writer
}
```

该类型继承了`*bufio.Reader`和`*bufio.Writer`的方法，从而实现了`io.ReadWriter`。这是在不给字段显式名称（它们不需要）和不编写显式转发方法的情况下完成的。

一个稍微复杂的例子是`context`包中的`timerCtx`：
```golang
type timerCtx struct {
  cancelCtx
  timer *time.Timer

  deadline time.Time
}
```
为了实现`Context`接口，`timerCtx`嵌入了`cancelCtx`，它实现了所需的4个方法中的3个（`Done`、`Err` 和`Value`）。然后它自己实现第四种方法——`Deadline`。