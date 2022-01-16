---
layout: post
title:  "Golang中的嵌套:结构体中的接口"
date:   2021-12-28 20:30:00 +0800
categories: Golang
---

# 结构体中的接口

这是Golang中最令人困惑的一种嵌套实现，在这篇文章中，我们将慢慢地研究这项技术，并展示几个真实世界的例子。最后，您会看到底层机制非常简单，并且该技术在各种场景中都很有用。

我们从最简单的例子开始
```golang
type Fooer interface {
  Foo() string
}

type Container struct {
  Fooer
}
```

`Fooer`是一个接口，并且嵌入到`Container`结构体中。回忆下，结构体的嵌入，会使得潜入的结构体方法提升（promotes）到被潜入的字段中。这类似于接口嵌套，就好像`Container`有这样的方法一样：

```golang
func (cont Container) Foo() string {
  return cont.Fooer.Foo()
}
```

但是`cont.Fooer`指向谁呢？它可以是任何实现`Fooer`接口的对象。这个对象从哪来的呢？在`Container`初始化时或稍后将其分配给`Fooer`字段。以下是例子：

```golang
// sink takes a value implementing the Fooer interface.
func sink(f Fooer) {
  fmt.Println("sink:", f.Foo())
}

// TheRealFoo is a type that implements the Fooer interface.
type TheRealFoo struct {
}

func (trf TheRealFoo) Foo() string {
  return "TheRealFoo Foo"
}
```

然后，我们执行：
```golang
co := Container{Fooer: TheRealFoo{}}
sink(co)
```

这会打印出`sink: TheRealFoo Foo`。

这是怎么回事？注意`Container`是如何初始化的；嵌入的`Fooer`字段被分配磕一个`TheRealFoo`类型的值。我们只能将实现`Fooer`接口的值分配给该字段——任何其他值都将被编译器拒绝。由于`Fooer`接口嵌入在`Container`中，所以它的方法被提升为`Container`的方法，这使得`Container`也实现了`Fooer`接口！这也是为什么我们可以把`Container`传递给`sink`。如果没有嵌入，`sink(co)`将无法编译，因为`co`不会实现`Fooer`。

你可能想知道如果`Container`的内嵌`Fooer`字段没有初始化会发生什么；这是一个好问题！发生的事情与您所期望的差不多——该字段保留其默认值，在接口的情况下为`nil`。所以这段代码：

```golang
co := Container{}
sink(co)
```

结果是：`runtime error: invalid memory address or nil pointer dereference` 。

这几乎涵盖了在结构中嵌入接口的工作原理。剩下的是更重要的问题——我们为什么需要这个？下面的示例将展示标准库中的几个用例，但我想从其他地方的一个用例开始，并演示在我看来，该技术在客户端代码中最重要的用途是什么。

## 例子：interface wrapper

假设我们想要有一个socket连接并具有其它功能，例如可以计算从连接中读取的字节总数。我们可以定义以下结构体：

```golang
type StatsConn struct {
  net.Conn

  BytesRead uint64
}
```

`StatsConn`现在实现了`net.Conn`接口，并且可以被用在任何需要`net.Conn`的地方。当`StatsConn`被初始化时，它会继承所有来自`net.Conn`的方法。然而，更关键的是，我们可以拦截我们希望的任何方法，但保持所有其他方法完好无损。在这个例子中，我们想拦截`Read`方法并记录读取的字节数：

```golang
func (sc *StatsConn) Read(p []byte) (int, error) {
  n, err := sc.Conn.Read(p)
  sc.BytesRead += uint64(n)
  return n, err
}
```

对于`StatsConn`的使用者而言，这个改变是透明的；我们仍然可以调用`Read`并用它做我们期望的事情（由于委托给`sc.Conn.Read`），但它也会做额外的操作。

如上一节所示，正确初始化 StatsConn 至关重要；例如：

```golang
conn, err := net.Dial("tcp", u.Host+":80")
if err != nil {
  log.Fatal(err)
}
sconn := &StatsConn{conn, 0}
```

在这，`net.Dial`返回了一个实现了`net.Conn`的值，所有我们可以用它来初始化`StatsConn`的内嵌字段。

我们现在可以将`sconn`传递给任何需要`net.Conn`参数的函数，例如:

```golang
resp, err := ioutil.ReadAll(sconn)
if err != nil {
  log.Fatal(err)
}
```

稍后我们可以访问其`BytesRead`字段来获取总数。

这是interface wrapper的示例。我们创建了一个实现现有接口的新类型，并重用了一个嵌入值来实现大部分功能。我们可以通过像这样一个显式的`conn`字段来实现这个而不嵌入：

```golang
type StatsConn struct {
  conn net.Conn

  BytesRead uint64
}
```

然后为`net.Conn`接口中的每个方法编写转发方法，例如：

```golang
func (sc *StatsConn) Close() error {
  return sc.conn.Close()
}
```

但是，`net.Conn`接口有8个方法。为所有这些编写转发方法是乏味和不必要的。嵌入接口为我们提供了所有这些转发方法，我们可以只重写我们需要的那些。

## 例子：sort.Reverse

标准库中，一个经典的接口嵌入结构体的例子是`sort.Reverse`。这个函数的用法常常让Go新手感到困惑，因为根本不清楚它是如何工作的。

让我们从一个简单的对整数切片进行排序的示例开始：
```golang
lst := []int{4, 5, 2, 8, 1, 9, 3}
sort.Sort(sort.IntSlice(lst))
fmt.Println(lst)
```

它会打印出`[1 2 3 4 5 8 9]`,那么它是如何运行的呢？`sort.Sort`接收一个实现了`sort.Interface`的参数，`sort.Interface`定义如下：
```golang
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

如果我们有个类型想要通过`sort.Sort`进行排序，那么就需要实现这个接口。对于`int`这类简单类型的我切片，标准库提供了现成的方法，例如`sort.IntSlice`。它们接受我们的值并在其上实现`sort.Interface`。

那么`sort.Reverse`是如何运行的呢？通过巧妙地使用嵌入在结构中的接口！`sort`包有这个（未导出的）类型来帮助完成任务:

```golang
type reverse struct {
  sort.Interface
}

func (r reverse) Less(i, j int) bool {
  return r.Interface.Less(j, i)
}
```

到目前为止，应该清楚这是做什么的；`reverse`通过嵌入的方式实现`sort.Interface`（只要它用实现接口的值初始化）,然后它从该接口拦截了一个方法——`Less`。然后它将它委托给嵌入值的`Less`，但颠倒参数的顺序。这个`Less`实际上是反向比较元素，这将使排序反向工作。

要完成解决方案，`sort.Reverse`函数很简单：

```golang
func Reverse(data sort.Interface) sort.Interface {
  return &reverse{data}
}
```

现在我们可以这样去做：

```golang
sort.Sort(sort.Reverse(sort.IntSlice(lst)))
fmt.Println(lst)
```

它会输出`[9 8 5 4 3 2 1]`,这里要理解的关键点是调用`sort.Reverse`本身不会对任何东西进行排序或反转。你可以把它视为一个高阶函数：它产生一个值来包装给它的接口并调整它的功能。对`sort.Sort`的调用是排序发生的地方。

## 例子 context.WithValue

`context`包有个函数`WithValue`：
```golang
func WithValue(parent Context, key, val interface{}) Context
```
这个函数会返回一个与`key`关联，值为`val`的`parent`的副本。让我们看看它是如何运行的。忽略错误检查，`WithValue`基本上可以归结为：
```golang
func WithValue(parent Context, key, val interface{}) Context {
  return &valueCtx{parent, key, val}
}
```
其中`valueCtx`是：
```golang
type valueCtx struct {
  Context
  key, val interface{}
}
```
这又是接口嵌入结构体的例子。`valueCtx`实现了`Context`接口并且可以轻松拦截`Context`的方法。它拦截`Value`方法：
```golang
func (c *valueCtx) Value(key interface{}) interface{} {
  if c.key == key {
    return c.val
  }
  return c.Context.Value(key)
}
```
其它的方法保持不变。

## 例子：degrading capability with a more restricted interface

这种技术相当先进，但它在整个标准库的许多地方都使用过。也就是说，我不认为客户端代码中通常需要它，所以如果你是Golang新手并且在第一次阅读时没有得到它，不要太担心。在您获得更多Golang经验后回到它。

我们先从`io.ReaderFrom`接口说起：
```golang
type ReaderFrom interface {
    ReadFrom(r Reader) (n int64, err error)
}
```

该接口可以被哪些想要从`io.Reader`中读取数据的类型实现。例如：`os.File`类型实现了该接口并且将读取器中的数据读取到它（`os.File`）代表的打开文件中。让我们看看是如何实现的：
```golang
func (f *File) ReadFrom(r io.Reader) (n int64, err error) {
  if err := f.checkValid("write"); err != nil {
    return 0, err
  }
  n, handled, e := f.readFrom(r)
  if !handled {
    return genericReadFrom(f, r)
  }
  return n, f.wrapErr("write", e)
}
```

它首先尝试使用特定于操作系统的`readFrom`方法从`r`读取。例如，在`Linux`上，它直接在内核中使用`copy_file_range`系统调用在两个文件之间进行非常快速的复制。

`readFrom`返回一个boolean表示是否成功（`handled`），如果没成功，那么`ReadFrom`会尝试通过`genericReadFrom`来执行一个"普通的"操作。代码如下：
```golang
func genericReadFrom(f *File, r io.Reader) (int64, error) {
  return io.Copy(onlyWriter{f}, r)
}
```

它使用`io.Copy`从`r`复制到`f`，到目前为止一切顺利。但是这个`onlyWriter`包装器是什么？
```golang
type onlyWriter struct {
  io.Writer
}
```
有趣的。所以这是我们现在熟悉的——将接口嵌入到结构中的技巧。但是如果我们在文件中四处搜索，我们将找不到`onlyWriter`上定义的任何方法，因此它不会拦截任何内容。那为什么需要它呢？

要理解为什么，我们需要先看看`io.Copy`做了什么。因为代码比较长，所以就不完整的写一遍了。但要注意的关键部分是，如果它的目标实现了`io.ReaderFrom`，它将调用`ReadFrom`。但这让我们回到了一个圈子，因为当`File.ReadFrom`被调用时，我们最终进入了`io.Copy`。这会导致无限递归！

现在开始明白为什么需要`onlyWriter`了。通过将`f`包裹在对`io.Copy`的调用中，`io.Copy`得到的不是实现`io.ReaderFrom`的类型，而只是实现了`io.Writer`的类型。然后它会调用我们`File`的`Write`方法，避免`ReadFrom`的无限递归陷阱。

正如我之前提到的，这种技巧是先进的。我觉得强调这一点很重要，因为它代表了“在结构中嵌入接口”工具的明显不同用途，并且在整个标准库中普遍使用。

`File`中的用法很好，因为它为`onlyWriter`提供了一个明确命名的类型，这有助于理解它的作用。标准库中的一些代码避开了这种自记录模式并使用匿名结构。例如，在`tar`包中，它完成了：
```golang
io.Copy(struct{ io.Writer }{sw}, r)
```