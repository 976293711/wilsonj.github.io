---
title: "concurrency 并发"
date: 2020-12-13T14:29:04+08:00
draft: false
toc: false
isCJKLanguage: true
images:
tags: 
  - golang
---

## Goroutine

1. 协程需要托管生命周期  （需要在自己可控的范围内）
2. 需要处理panic 不然会导致整个程序终止
2. go 关键字 最好显示调用


## 内存模式


1. count++并不是原子的 翻译成汇编语言就是 先读取count的值，然后进行+1操作，最后赋值
2. 有空的时候可以看看 [并发时会出现的情况](https://www.jianshu.com/p/5e44168f47a3) 里面总结了很多并发的时候会遇到的坑

## Sync同步原语

#### atomic
1. atomic.Value的使用 (读多写少) 【copy on write 面试可能会问 】 redis bgsave
> 基本用法就是: 读的场景多 写的场景少 每次写都全部覆盖写入 
2. 可以了解一下atomic的用法


#### Copy on write
1. 多个调用者同时请求相同的资源，它们会共同获取相应的指针指向相同的资源，知道某个调用者试图修改资源内容时，系统才会真正赋值一个专用副本给调用者，而其他调用者所见到的最初的资源任然保持不变。

2. fork()之后，kernel把父进程中所有的内存页的权限都设为read-only，然后子进程的地址空间指向父进程。当父子进程都只读内存时，相安无事。当其中某个进程写内存时，CPU硬件检测到内存页是read-only的，于是触发页异常中断（page-fault），陷入kernel的一个中断例程。中断例程中，kernel就会把触发的异常的页复制一份，于是父子进程各自持有独立的一份。


### Mutex
1. 写多读少 使用Mutex
2. 锁的实现原理

**go 1.8之前锁的实现 Barging 和 Spinning的结合**

当试图获取已经被持有的锁时，如果本地队列为空并且 P(协程队列) 的数量大于1，goroutine 将自旋几次(用一个 P 旋转会阻塞程序)。自旋后，goroutine park（拿不到锁时，会进入到锁队列）。在程序高频使用锁的情况下，它充当了一个快速路径。首先g1获取锁的协 得到锁了 

然后 在这期间 g2请求同一个锁时 会把协程加入到 FIFO（先进先出）的队列中当g1完成工作时，会释放锁。然后通知队列唤醒g2，将g2标记为可运行的状态， 等待Go调度程序在线程上运行

**go 1.9之后锁的实现**

通过添加一个新的饥饿模式来解决先前解释的问题，该模式将会在释放时候触发 handsoff。所有等待锁超过一毫秒的 goroutine(也称为有界等待)将被诊断为饥饿。当被标记为饥饿状态时，unlock 方法会 handsoff 把锁直接扔给第一个等待者。
 
在饥饿模式下，自旋也被停用，因为传入的goroutines 将没有机会获取为下一个等待者保留的锁。


**`Barging`**

这种模式是为了提高吞吐量，当锁被释放时，它会唤醒第一个等待者，然后把锁给第一个等待者或者给第一个请求锁的人。

**`Handsoff`**

 当锁释放时候，锁会一直持有直到第一个等待者准备好获取锁。它降低了吞吐量，因为锁被持有，即使另一个 goroutine 准备获取它。

**`Spinning`**  pause自己研究一下 自旋也试试

自旋是指当一个线程在获取锁的时候，如果锁已经被其他线程获取，那么该线程将循环等待，然后不断地判断是否能够被成功获取，知直到获取到锁才会退出循环。

自旋在等待队列为空或者应用程序重度使用锁时效果不错。Parking 和 Unparking goroutines 有不低的性能成本开销，相比自旋来说要慢得多。

对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，“自旋”一词就是因此而得名。

####  [ErrorGroup](https://pkg.go.dev/golang.org/x/sync/errgroup)
并发原语正好适合在这种场景下使用，它在WaitGroup的功能基础上还提供了，错误传播以及上下文取消的功能。[出自此处](https://mp.weixin.qq.com/s/AUNXmKkQGI9uotwPoweg3A)

`ErrorGroup`有一个特点是会返回所有执行任务的`goroutine`遇到的第一个错误
```go
package main

import (
	"context"
	"fmt"
	"github.com/go-kratos/kratos/pkg/sync/errgroup"
	"log"
	"time"
)

func main() {
	eg := errgroup.WithCancel(context.Background())

	for i := 0; i < 100; i++ {

		i := i
		eg.Go(func(ctx context.Context) error {
			time.Sleep(2 * time.Second)

			select {
			case <-ctx.Done() :
				fmt.Println("Canceled:", i)
				return nil
			default:
				if i > 90 {
					fmt.Println("Error:", i)
					return fmt.Errorf("error: %d", i)
				}
				fmt.Println("End:", i)
				return nil
			}
		})
	}
	if err := eg.Wait(); err != nil {
		log.Fatal(err)
	}
}
```

`Go`方法单独开启的`goroutine`在执行参数传递进来的函数时，如果函数返回了错误，会对`ErrorGroup`持有的`err`字段进行赋值并及时调用`cancel`函数，通过上下文通知其他子任务取消执行任务。
```text
Error: 98
Canceled: 99
Canceled: 92
Canceled: 93
Canceled: 94
Canceled: 95
Canceled: 96
Error: 91
Canceled: 2
Canceled: 55
Canceled: 70
...
2020/12/23 15:42:42 error: 98
```

#### 在使用时，我们也需要注意它的两个特点：
 
- `errgroup.Group`在出现错误或者等待结束后都会调用 `Context`对象 的 `cancel` 方法同步取消信号。

- 只有第一个出现的错误才会被返回，剩余的错误都会被直接抛弃。


### `sync.Pool` 降低GC的压力，减少内存的分配 （了解）
`sync.Pool`的场景是用来保存和复用临时对象，以减少内存分配，降低 GC 压力(Request-Driven 特别合适)。


## `Context` 尝试使用！
- 使得跨 API 边界的请求范围元数据、取消信号和截止日期很容易传递给处理请求所涉及的所有 goroutine(显示传递)。

**使用`context`的注意事项**
1. 对服务器的传入请求应该创建context
2. 显式传入每个context
3. 函数调用需要传入context
4. 使用`WithCancel`, `WithDeadline`, `WithTimeout`, `WithValue`来替换`context`
5. 当一个`context`被取消时，所有派生他的`context`也被取消
6. 相同的`context`可以传递给运行在不同goroutines中的函数。`context`对于多个goroutines同时使用是安全的。
7. 不要传递nil`context`，即使函数允许这样做。如果不确定使用哪个`context`，则传递一个TODO`context`。
8. context 不应该存储业务数据。

 
### `context.withValue` 允许上下文携带请求范围的数据 
- `context.Value(key)` 会一只递归寻找key 一直寻找到父`context` 最后返回对应的值 或者空

- 一般可以用于携带跟踪的信息


## `Chan`

 