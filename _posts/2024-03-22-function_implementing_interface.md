---
title: Go中的接口型函数
date: 2024-03-22 22:56:00 +0800
categories: [Programing Language, Golang]
tags: [golang, design pattern]     # TAG names should always be lowercase
---
在Go中，函数也可以作为方法的接收者。也就是说，假如`f`是一个函数，`m`是一个以`f`为接收者的方法，我们可以调用`f.m()`.代码如下所示：
```go
type F func()

func (f F) m() {
	fmt.Println("calling m()")
}

func main() {
	var f = F(func() {
		fmt.Println("calling f()")
	})

	f()   // print: calling f()
	f.m() // print: calling m()
}
```

那么在什么时候我们会需要一个“函数的方法”，而不是直接拆成两个独立的函数呢？一个常见的场景就是**接口型函数**。
接下来，我们以标准库`net/http`为例。`net/http`中有一个`Handler`接口。
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
显然，我们可以通过实现这个接口来处理请求。例如
```go
type Counter int64

func (c *Counter) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	*c++
	fmt.Printf("Counter: %v", *c)
}

func main() {
	var c Counter
	http.Handle("/", &c)
	http.ListenAndServe(":8888", nil)
}
```
在这里，我们通过结构体`Counter`保存请求的次数，并且实现了`Handler`接口，在每次请求时把请求次数打印出来。这样已经足够简单。
但如果我们想要实现一些更简单的、没有任何状态的逻辑，比如每次请求时打印"Hi!"。上面的写法就会显得比较臃肿了。像那些无状态的函数，或许我们更希望能将匿名函数作为参数传给`http.Handle`，而不是又定义一个空结构体。**接口型函数**就可以优雅地做到这一点。

`net/http`中还有关于`HandlerFunc`的定义
```go
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
在这里，`HandlerFunc`作为接收者实现了`Handler`接口，实现的方法里就是简单地调用了自己。我们可以像下面一样方便地传入匿名函数。
```go
func main() {
	http.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("Hi!")
	}))
	http.ListenAndServe(":8888", nil)
}
```
在这里，通过`http.HandlerFunc()`把传入的匿名函数包装成了`http.ServeHTTP`的参数类型，从而复用了原来的方法来实现传入匿名函数。


总结一下，**接口型函数**的好处就是：让一个接口的参数，既可以使用实现了接口的对象，也可以使用单独的匿名函数。