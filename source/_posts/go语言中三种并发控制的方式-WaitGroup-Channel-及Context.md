---
title: Go语言中三种并发控制的方式：WaitGroup, Channel, 及 Context
date: 2025-07-15 17:59:12
tags:
  - Go
  - 并发
  - WaitGroup
  - Channel
  - Context
---

Go 语言以其简洁高效的并发模型（Goroutine 和 Channel）而闻名。然而，仅仅启动成百上千的 Goroutine 是不够的，我们还需要有效的机制来管理和控制它们的生命周期。本文将详细介绍 Go 中最核心的三种并发控制方式：`sync.WaitGroup`、`Channel` 和 `context.Context`。

## 1. `sync.WaitGroup`：等待一组任务完成

`WaitGroup` 是 `sync` 包提供的一个非常基础的并发原语。它就像一个计数器，用于等待一组 Goroutine 全部执行完毕。

### 核心方法

-   `Add(delta int)`：计数器增加 `delta`。通常在启动 Goroutine 前调用，`delta` 的值就是 Goroutine 的数量。
-   `Done()`：计数器减一。通常在 Goroutine 的末尾通过 `defer` 调用。
-   `Wait()`：阻塞当前 Goroutine，直到计数器归零。

### 使用场景

当你需要启动多个并行的“子任务”，并且主任务需要等待所有子任务都完成后再继续执行时，`WaitGroup` 是最简单、最直接的选择。它不关心子任务的执行结果，只关心它们是否“完成”。

### 代码示例

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, wg *sync.WaitGroup) {
	// 在函数退出时通知 WaitGroup 任务已完成
	defer wg.Done()

	fmt.Printf("Worker %d starting\n", id)
	// 模拟耗时操作
	time.Sleep(time.Second)
	fmt.Printf("Worker %d done\n", id)
}

func main() {
	// 创建一个 WaitGroup
	var wg sync.WaitGroup

	// 启动 5 个 worker Goroutine
	for i := 1; i <= 5; i++ {
		// 每启动一个 Goroutine，计数器加 1
		wg.Add(1)
		go worker(i, &wg)
	}

	// 等待所有 Goroutine 完成
	fmt.Println("Main: Waiting for workers to finish...")
	wg.Wait()
	fmt.Println("Main: All workers have finished.")
}
```

## 2. `Channel`：Goroutine 间的通信与同步

`Channel` 是 Go “不要通过共享内存来通信，而要通过通信来共享内存” 哲学的核心体现。它是一个类型化的管道，可以安全地在多个 Goroutine 之间传递数据。

### 核心特性

-   **类型安全**：每个 Channel 只能传递一种类型的数据。
-   **阻塞性**：默认情况下，发送和接收操作都是阻塞的：
    -   向一个无缓冲的 Channel 发送数据，会阻塞直到有另一个 Goroutine 来接收。
    -   从一个无缓冲的 Channel 接收数据，会阻塞直到有另一个 Goroutine 来发送。
    -   对于有缓冲的 Channel，发送操作仅在缓冲区满时阻塞，接收操作仅在缓冲区空时阻塞。
-   **同步**：阻塞特性使得 Channel 天然具备同步功能。

### 使用场景

1.  **数据传递**：当一个 Goroutine 需要将处理结果发送给另一个 Goroutine 时。
2.  **任务分发**：经典的 "Worker Pool"（工作池）模式，主 Goroutine 将任务通过 Channel 发送给多个 Worker Goroutine。
3.  **信号通知**：使用 Channel 来通知某个事件的发生，例如任务完成或需要退出。

### 代码示例（生产者-消费者模型）

```go
package main

import (
	"fmt"
	"time"
)

// producer 向 channel 发送数据
func producer(ch chan<- int) {
	for i := 0; i < 5; i++ {
		fmt.Printf("Producing: %d\n", i)
		ch <- i // 将数据发送到 channel
		time.Sleep(500 * time.Millisecond)
	}
	close(ch) // 数据发送完毕，关闭 channel
}

// consumer 从 channel 接收数据
func consumer(ch <-chan int, done chan<- bool) {
	fmt.Println("Consumer waiting for data...")
	// 使用 for-range 循环不断从 channel 接收数据，直到 channel 被关闭
	for num := range ch {
		fmt.Printf("Consuming: %d\n", num)
	}
	fmt.Println("Channel closed, consumer finished.")
	done <- true // 通知主 goroutine，消费已完成
}

func main() {
	dataChan := make(chan int)
	doneChan := make(chan bool)

	go producer(dataChan)
	go consumer(dataChan, doneChan)

	// 等待 consumer 完成
	<-doneChan
	fmt.Println("Main process finished.")
}
```


## 3. `context.Context`：控制 Goroutine 的生命周期

`Context` 是 Go 1.7 引入的标准库，主要用于控制那些处理请求或需要长时间运行的 Goroutine。它提供了一种优雅的方式来传递**取消信号**、**超时时间**、**截止日期**以及**请求范围的值**。

### 核心功能

-   **取消（Cancellation）**：当一个操作不再需要时（例如，用户取消了请求），可以通过 `Context` 通知所有相关的 Goroutine 停止工作并退出，从而释放资源。
-   **超时（Timeout/Deadline）**：可以为一组操作设置一个总的超时时间或截止日期。一旦超时，所有相关的 Goroutine 都会收到取消信号。
-   **值传递（Value Passing）**：可以在一个请求的处理链中传递一些请求范围的数据，如用户身份、Trace ID 等。

### 使用场景

-   处理 HTTP 请求，当客户端断开连接时，需要取消后端的数据库查询等操作。
-   调用下游微服务时，需要设置超时时间，避免长时间等待导致雪崩。
-   一个复杂的任务由多个 Goroutine 协作完成，需要一个统一的机制来控制它们的启动和停止。

### 代码示例（手动取消）

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// worker 监听 Context 的取消信号，并使用 WaitGroup 来通知主程序它已完成清理
func worker(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("Worker: started")

	for {
		select {
		case <-ctx.Done():
			// 收到取消信号，进行清理并退出
			fmt.Println("Worker: received cancellation signal. Cleaning up...")
			// 模拟清理工作
			time.Sleep(500 * time.Millisecond)
			fmt.Println("Worker: finished cleanup.")
			return
		default:
			// 正常工作
			fmt.Println("Worker: doing work...")
			time.Sleep(1 * time.Second)
		}
	}
}

func main() {
	// 创建一个可取消的 Context 和一个 WaitGroup
	ctx, cancel := context.WithCancel(context.Background())
	var wg sync.WaitGroup

	wg.Add(1)
	go worker(ctx, &wg)

	// 运行一段时间后，从外部发出取消信号
	fmt.Println("Main: waiting for 3 seconds before cancelling.")
	time.Sleep(3 * time.Second)

	fmt.Println("Main: sending cancellation signal.")
	cancel() // 发出取消信号

	// 等待 worker goroutine 完成其清理工作
	wg.Wait()
	fmt.Println("Main: worker has finished. Exiting.")
}
```

## 总结与对比

| 特性 | `sync.WaitGroup` | `Channel` | `context.Context` |
| :--- | :--- | :--- | :--- |
| **主要目的** | 等待一组 Goroutine 完成 | Goroutine 间的安全数据交换和同步 | 控制 Goroutine 的生命周期（取消、超时） |
| **数据流** | 无 | 双向或单向 | 单向（从父到子），用于传递信号和元数据 |
| **控制方式** | 计数器 | 阻塞式发送/接收 | 信号通知（通过 `Done()` channel） |
| **适用场景** | 简单的“等待-完成”模式 | 需要数据交换、工作池、精细同步 | 需要取消、超时、截止日期控制的场景 |

### 如何选择？

-   如果只是想简单地等待一组不相关的任务执行完毕，使用 `WaitGroup`。
-   如果需要在 Goroutine 之间传递数据或进行复杂的同步，使用 `Channel`。
-   如果要控制一个请求范围内的所有 Goroutine（尤其是涉及 I/O、RPC 调用），或者需要实现优雅的取消和超时机制，必须使用 `Context`。

在实际开发中，这三者往往会结合使用，以构建健壮、高效的并发程序。例如，你可以使用 `Context` 来控制一组 Worker Goroutine 的生命周期，使用 `Channel` 来给它们分发任务，并使用 `WaitGroup` 来确保在主程序退出前所有 Worker 都已清理完毕。