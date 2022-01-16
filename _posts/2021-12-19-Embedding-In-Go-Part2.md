---
layout: post
title:  "Golang中的嵌套:接口嵌套"
date:   2021-12-19 23:00:00 +0800
categories: Golang
---

# Golang中的嵌套：接口嵌套

## 结构体嵌套
将一个接口嵌入到另一个接口中是`Go`中最简单的嵌入方式，因为接口只声明功能；它们实际上并没有为类型定义任何新数据或行为。

我们以[`Effective Go`](https://golang.org/doc/effective_go.html#embedding) 中的例子开始我们的话题，因为它展示了一个众所周知的在`Go`标准库中嵌入接口的案例。给定`io.Reader`和`io.Writer`接口：

```golang
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```
我们如何为既是读取器又是写入器的类型定义接口？一个明确的方法是：
```golang
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

除了在多个地方重复相同的方法声明这一明显问题之外，这阻碍了`ReadWriter`的可读性，因为它与其他两个接口的组合方式并不明显。您要么必须牢记每个方法的确切声明，要么继续回顾其他接口。

请注意，标准库中有许多这样的组合接口：`io.ReadCloser`、`io.WriteCloser`、`io.ReadWriteCloser`、`io.ReadSeeker`、`io.WriteSeeker`、`io.ReadWriteSeeker`以及其它包中的。仅`Read`方法的声明可能必须在标准库中重复**10**次以上。这很丢人，但幸运的是接口嵌入提供了完美的解决方案：

```golang
type ReadWriter interface {
  Reader
  Writer
}
```
除了防止重复外，该声明还以最清晰的方式表明了意图：为了实现`ReadWriter`，你必须实现`Reader`和`Writer`。

## 修复Go1.14中的重叠方法

嵌入接口是可组合的，并且可以按您的预期工作。例如，给定接口`A`、`B`、`C`和`D`，使得：
```golang
type A interface {
  Amethod()
}

type B interface {
  A
  Bmethod()
}

type C interface {
  Cmethod()
}

type D interface {
  B
  C
  Dmethod()
}
```
`D`的方法集将由`Amethod()`、`Bmethod()`、`Cmethod()`和`Dmethod()`组成。

然而假设`C`被这样定义：
```golang
type C interface {
  A
  Cmethod()
}
```

一般来说，这不应该改变`D`的方法集。但是，在Go1.14之前，这将导致`D`出现错误`"Duplicate method Amethod"`，因为`Amethod()`将被声明两次——一次通过`B`的嵌入，一次通过`C`的嵌入。

[Go1.14修改了这个问题](https://github.com/golang/proposal/blob/master/design/6977-overlapping-interfaces.md) ,在新示例可如我们期望的那样正常工作。`D`的方法集是由它嵌入的接口的方法集和它自己的方法集合在一起的。

一个更实际的例子来自标准库。`io.ReadWriteCloser`类型被定义为：
```golang
type ReadWriteCloser interface {
  Reader
  Writer
  Closer
}
```
但它可以更简洁地定义为：
```golang
type ReadWriteCloser interface {
  io.ReadCloser
  io.WriteCloser
}
```

在Go1.14之前，这是不可能的，因为从`io.ReadCloser`和从`io.WriteCloser`进来的方法`Close()`重复。

## 例子：net.Error

`net`包有自己的错误接口，因此声明：
```golang
// An Error represents a network error.
type Error interface {
  error
  Timeout() bool   // Is the error a timeout?
  Temporary() bool // Is the error temporary?
}
```
注意内置错误接口的嵌入。这个嵌入非常清楚地声明了意图：`net.Error`也是一个`error`。读者看到这个代码会有一个非常清晰的答案去思考这段代码，而不是寻找一个`Error()`方法的声明并在心里将其与错误的规范方法进行比较。

## 例子：heap.Interface
`heap`包具有为客户端类型定义的待实现接口，声明如下：
```golang
type Interface interface {
  sort.Interface
  Push(x interface{}) // add x as element Len()
  Pop() interface{}   // remove and return element Len() - 1.
}
```

所有实现`heap.Interface`的类型也必须实现`sort.Interface`；后者需要3种方法，因此编写没有任何嵌套结构体的`heap.Interface`将如下所示：
```golang
type Interface interface {
  Len() int
  Less(i, j int) bool
  Swap(i, j int)
  Push(x interface{}) // add x as element Len()
  Pop() interface{}   // remove and return element Len() - 1.
}
```
带有嵌入的版本在许多层面上都更胜一筹。最重要的是，它立即明确了一个类型必须首先实现`sort.Interface`；而较长版本的则明显更加复杂难以理解。