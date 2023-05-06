---
title: "【译】如何避免协程泄漏"
date: 2023-05-05T11:46:21+08:00
draft: false
isCJKLanguage: true
tags :
    - "go"
    - "goroutine"
    - "协程"
categories:
    - "go"
---


Go语言最大的优势之一就是协程并发，但是，能力越大责任越大。
使用协程虽然很简单，但是如果不小心很容易引入难以排查的bug。
协程泄漏就是其中之一，随着协程数量的增长，最终会导致应用程序出现问题。
这篇文章列举了协程泄漏的各种场景，以及如何规避。
<!--more-->

## 什么是协程泄漏

当创建一个新的协程时，计算机会在堆中分配内存，并在执行完成后释放它们。
协程泄漏是一种内存泄漏，当协程未正常终止，并在应用程序的生命周期内一直挂在后台时发生。

先看一个简单的示例:

```go
func goroutineLeak(ch chan int) {
    data := <- ch
    fmt.Println(data)
}

func handler() {
    ch := make(chan int)

    go goroutineLeak(ch)
    return
}
```

当 handler 方法返回时，协程仍然在后台存活，阻塞并等待接收数据。但是并没有往携程发送数据的代码，因此导致了协程泄漏。

在这篇文章中，我将介绍两种常见的携程泄漏的场景。

- 发送者遗漏
- 接收者遗漏

Jacob Walker 在他的文章中非常好地解释了这些场景。在这篇[文章](https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html)中，我将通过更多示例来解释他们。

## 1.被遗忘的发送者

被遗忘的发送者发生在发送者被阻塞时，因为没有接收者在通道的另一端等待接收数据。

```go
func forgottenSender(ch chan int) {
    data := 3
  
    // 因为没有人正在接收数据，所以这里回阻塞
    ch <- data
}

func handler () {
    ch := make(chan int)
  
    go forgottenSender(ch)
    return
}
```

虽然乍一看似乎微不足道，但在以下两种情况下很容易被忽视。

### Context 的不恰当使用

```go
func forgottenSender(ch chan int) {
    data := networkCall()
  
    ch <- data
}

func handler() error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
    defer cancel()
  
    ch := make(chan int)
    go forgottenSender(ch)
  
    select {
        case data := <- ch: {
            fmt.Printf("Received data! %s", data)
      
            return nil
        }
    
        case <- ctx.Done(): {
            return errors.New("Timeout! Process cancelled. Returning")
        }
    }
}
```

在上面的例子中，我们伪造了一个常见的 web 请求处理器。
我们定义了一个在 10 毫秒后发出超时的 Context ，然后是一个异步进行网络调用的协程。

`select`语句等待多个channel操作，它会一直阻塞直到它的其中一个case可以执行。
如果超时在网络调用之前完成。`case <- ctx.Done()` 将被执行，处理器将会返回一个错误。

在处理器返回之后，就没有接收者在接收数据了。`forgottenSender`将被阻塞，等待接收数据，但是这是不可能发生的。
这就导致了协程泄漏。

### 在错误检查之后接收数据

下面是另一个典型的场景。

```go
func forgottenSender(ch chan int) {
    data := networkCall()
  
    ch <- data
}

func handler() error {
    ch := make(chan int)
    go forgottenSender(ch)
  
    err := continueToValidateOtherData()
    if err != nil {
        return errors.New("Data is invalid! Returning.")
    }
  
    data := <- ch
  
    return nil
}
```

在上面的示例中，我们定义了一个处理程序并生成一个新的协程用来发器异步网络调用。
在等待调用返回的同时，我们继续处理其他验证逻辑。
可以看出来，当`continueToValidateOtherData`返回了错误，handler会直接返回，这个时候就会发生协程泄漏。
没有执行等待接收数据的逻辑，`forgottenSender`将被永远阻塞。

### 如何避免被遗忘的发送者

使用有缓冲的channel

如果你还记得的话，被遗忘的发送者的发生是因为另一边没有接受者。 阻塞问题的罪魁祸首是无缓冲的通道！
一个无缓冲的通道，在向其发送数据时，同时需要一个接收者，否则发送者将会被阻塞。这是在没有为channel指定容量的时候发生的。

```go
func forgottenSender(ch chan int) {
    data := 3
  
    // This will NOT block
    ch <- data
}

func handler() {
    // Declare a BUFFERED channel
    ch := make(chan int, 1)
  
    go forgottenSender(ch)
    return
}
```

通过为channel指定容量，在上面的例子中容量是1，这将缓解上面所有提到的问题。
发送者可以发送数据给channel，而不需要有接收者。

## 2.0 被遗弃的接收者

顾名思义，被遗弃的接收者则完全相反。

当接收方被阻塞时会发生这种情况，因为另一方没有发送者发送数据。

```go
func abandonedReceiver(ch chan int) {
// This will be blocked
    data := <- ch
  
    fmt.Println(data)    
}

func handler() {
    ch := make(chan int)
  
    go abandonedReceiver(ch)
  
    return
}
```

第三行将被永远阻塞，因为没有发送者发送数据。
让我们来看看两个经常被忽视的常见场景。

### 未关闭的channel

```go
func abandonedWorker(ch chan string) {
    for data := range ch {
        processData(data)
    }
  
    fmt.Println("Worker is done, shutting down")
}

func handler(inputData []string) {
    ch := make(chan string, len(inputData))
  
    for _, data := range inputData {
        ch <- data
    }
  
    go abandonedWorker(ch)
    
    return
}
```

在上面的例子中，handler接收一个字符串切片，创建了一个channel，并发送了数据。
之后handler通过协程创建了一个worker。这个worker将处理数据，并在所有数据处理完的时候结束。

然后，worker将永远不可能执行第6行代码，尽管所有数据都已经被消费且处理。

这里的channel，虽然空了，但是没有关闭。没有关闭，worker就会继续等待之后可能的数据。因此，他会在这里永远等待。

### 在错误检查之后发送数据

这个和前一个例子非常相似

```go
func abandonedWorker(ch chan []int) {
    data := <- ch

    fmt.Println(data)
}

func handler() error {
    ch := make(chan []int)
    go abandonedWorker(ch)

    records, err := getFromDB()
    if err != nil {
        return errors.New("Database error. Returning")
    }

    ch <- records

    return nil
}
```

在上面的例子中，handler首先创建了worker协程来消费数据。

handler之后的逻辑是从数据库里查询数据，在之后是将查到的数据传入channel中供worker消费。

如果在查询数据库的时候出现错误，handler会立即返回。那么将不会有任何数据发送到channel。

因此，这里的worker将在后台一直执行。

### 如何避免“被遗弃的接收者”

在以上两个场景中，接收器都处于挂起状态，因为它们“认为”channel将被传入数据。 因此，他们会阻塞并永远等待。

解决方案是一行简单的代码

```go
defer close(ch)
```

最佳实践就是就是，当你启动一个新的协程时，同时用defer来关闭channel。这可以确保channel被正确关闭，当发送数据结束或者函数退出时。

接收者也会接收到channel的关闭信息，并终止执行。

```go
func abandonedReceiver(ch chan int) {
    // This will NOT be blocked FOREVER
    data := <- ch
  
    fmt.Println(data)    
}

func handler() {
    ch := make(chan int)
  
      // Defer the CLOSING of channel
    defer close(ch)
  
    go abandonedReceiver(ch)
    return
}
```

## 总结

这就是协程泄漏。
尽管它不像任何其他协程错误那么严重，但此类泄漏仍会显着耗尽应用程序的内存。

请记住，能力越大，责任越大。

保护我们的应用程序免受错误影响的责任在你我身上——开发人员！

下次见。

## 相关文章

[Goroutine Leaks - The Forgotten Sender
](https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html)

[Goroutine Leaks - The Abandoned Receivers
](https://www.ardanlabs.com/blog/2018/12/goroutine-leaks-the-abandoned-receivers.html)

## 原文

[原文](https://betterprogramming.pub/common-goroutine-leaks-that-you-should-avoid-fe12d12d6ee)
