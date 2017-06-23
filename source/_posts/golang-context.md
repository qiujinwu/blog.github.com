---
title: Golang Context
date: 2017-6-23 22:23:11
tags:
 - golang
categories:
 - Golang
---

从go1.7开始，golang.org/x/net/context包正式作为context包进入了标准库。官方的说明如下
>Package context defines the Context type, which carries deadlines, cancelation signals, and other request-scoped values across API boundaries and between processes.

参考
1. <https://blog.golang.org/context>
2. <https://godoc.org/golang.org/x/net/context>

Context接口非常简单
``` go
// A Context carries a deadline, a cancelation signal, and other values across
// API boundaries.
//
// Context's methods may be called by multiple goroutines simultaneously.
type Context interface {
	// 是否有超时，超时context才有
	Deadline() (deadline time.Time, ok bool)
    // 是否已经结束，通过从channel中读取数据判断，用于被监听的routine
	Done() <-chan struct{}
    // 当Done返回的ch读取数据（或者被close），可以获取到错误（取消，超时等）
	Err() error
    // value Context的值，通过key获取
	Value(key interface{}) interface{}
}
```

Context的核心通过channel来作同步，为了理解，最好了解go的同步机制


# [Go语言并发模型](Go语言并发模型：像Unix Pipe那样使用channel)
参考[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)

和其他语言相比，go对并发的控制源于非常简单，但功能并不弱，常用的包括（但不限于）
1. sync.Mutex 互斥
2. sync.WaitGroup
3. channel


官方推荐channel，并且对此有很好的优化

## pipeline
连续单个的管道
``` go
package main

import "fmt"

func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

func main() {
	// 设置流水线
	c := gen(2, 3)
	out := sq(c)

	// 消费输出结果
	fmt.Println(<-out) // 4
	fmt.Println(<-out) // 9
}
```

多个并行的pipe并行处理，并汇聚到下一个pipe，核心是sync.WaitGroup作屏障
``` go
package main

import (
	"fmt"
	"sync"
)

func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

func merge(cs ...<-chan int) <-chan int {
	var wg sync.WaitGroup
	out := make(chan int)

	// 为每一个输入channel cs 创建一个 goroutine output
	// output 将数据从 c 拷贝到 out，直到 c 关闭，然后 调用 wg.Done
	output := func(c <-chan int) {
		for n := range c {
			out <- n
		}
		wg.Done()
	}
	wg.Add(len(cs))
	for _, c := range cs {
		go output(c)
	}

	// 启动一个 goroutine，用于所有 output goroutine结束时，关闭 out 
	// 该goroutine 必须在 wg.Add 之后启动
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

func main() {
	in := gen(2, 3)

	// 启动两个 sq 实例，即两个goroutines处理 channel "in" 的数据
	c1 := sq(in)
	c2 := sq(in)

	// merge 函数将 channel c1 和 c2 合并到一起，这段代码会消费 merge 的结果
	for n := range merge(c1, c2) {
		fmt.Println(n) // 打印 4 9, 或 9 4
	}
}
```

## 处理中断
通过传递一个用于控制是否退出的channel来实现和谐的中断，这也是Context的核心所在，为了控制多个channel，只需要**close channel写端**即可唤醒所有的channel读端
``` go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // 为 cs 的每一个 channel 创建一个 goroutine
    // 这个 goroutine 运行 output，它将数据从 c
    // 拷贝到 out，直到 c 关闭，或者 接收到 done
    // 的关闭信号。人啊后调用 wg.Done()
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
```

关于中断多个channel，参考<https://chilts.org/2017/06/12/cancelling-multiple-goroutines>

channel作为一种同步机制，有必要了解go的内存模型和（非）缓存的同步差异

# [go内存模型](http://blog.csdn.net/ywh147/article/details/10942603)
参考<https://golang.org/ref/mem>

## Happens Before/After
Happens-before用来指明Go程序里的内存操作的局部顺序。如果一个内存操作事件e1 happens-before e2，则e2 happens-after e1也成立；如果e1不是happens-before e2,也不是happens-after e2，则e1和e2是并发的。

下面这段是核心，也很费解

>在这个定义之下，如果以下情况满足，则对变量（v）的内存写操作（w）对一个内存读操作（r）来说允许可见的：
   1. r不在w开始之前发生（可以是之后或并发）；
   2. w和r之间没有另一个写操作(w’)发生；

>为了保证对变量（v）的一个特定写操作（w）对一个读操作（r）可见，就需要确保w是r唯一允许的写操作，于是如果以下情况满足，则对变量（v）的内存写操作（w）对一个内存读操作（r）来说保证可见的：
   1. w在r开始之前发生；
   2. 所有其它对v的写操作只在w之前或r之后发生；


简单地说，对于【**允许可见**】表示有可能读到正确的写数据，但是由于存在读脏数据的可能，（假定读操作是有意义的）所以极端情况，编译器甚至优化掉写代码逻辑

而【**保证可见**】就确保一定能读到正确的写数据，所以编译器会确保写操作在读操作前执行

## Channel 同步
1. 对一个Channel的发送操作(send) happens-before 相应Channel的接收操作完成
2. 关闭一个Channel happens-before 从该Channel接收到最后的返回值0
3. 不带缓冲的Channel的接收操作（receive） happens-before 相应Channel的发送操作完成

``` go
package main

var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}
func main() {
	go f()
	<-c
	print(a)
}
```
上述代码可以确保输出hello, world，因为【a = "hello, world"】 **happens-before** 【c <- 0】，【print(a)】 **happens-after** 【<-c】， 根据上面的规则1）以及happens-before的可传递性，a = "hello, world" happens-beforeprint(a)。

``` go
package main

var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	close(c) ///////
}
func main() {
	go f()
	<-c
	print(a)
}
```
根据规则2）把c<-0替换成close(c)也能保证输出hello,world，因为关闭操作在<-c接收到0之前发送。
``` go
package main

var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
```
根据规则3），因为c是不带缓冲的【Channel，a = "hello, world"】 **happens-before** 【<-c】 **happens-before** 【<- 0】 **happens-before** 【print(a)】， 

**但如果c是缓冲队列，如定义c = make(chan int, 1), 那结果就不确定了。**

# Context 解决的问题
在并发模型中，经常会开启几个go routine来并发处理任务，这就提出了能够中断（取消）的需求，虽然前面提到的方案都是可行的（实际上Context也就是对上述方案的封装），但抽象得并不好，

还有一个很大的问题是，如果调用的go routine内部又开启了多个go routine，如何和谐的递归触发中断呢？

除了中断，另外一个常见的需求是超时，以及传递参数
> go routine可以直接传参数，Context valueCtx的优势在于，父routine的value可以继承到子routine（若key不冲突），只需要（以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。）

为此，Context提供了几个工具函数
``` go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```
1. WithTimeout对WithDeadline的一个简单转换封装，就是相对时间/绝对实际的差别，它继承自CancelCtx
2. CancelCtx是可以取消Context，另一个参数是一个函数，调用函数实际上就是关闭（close）Done的channel。
3. Context提供了空Context（emptyCtx），无实际意义，除了做了根Context

``` go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```

## 使用
参考<http://www.flysnow.org/2017/05/12/go-in-action-go-context.html>

``` go
package main

import (
	"fmt"
	"time"
	"context"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(2 * time.Second)
			}
		}
	}(ctx)
	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}
```
Context控制多个goroutine
``` go
package main

import (
	"fmt"
	"time"
	"context"
)

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")
	go watch(ctx,"【监控3】")
	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

```

## CancelCtx
``` go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	done chan struct{} // closed by the first cancel call.

	mu       sync.Mutex
    // 管理routine树状结构（bool无意义，只因无类似stl set的数据结构）
	children map[canceler]bool // set to nil by the first cancel call
    // 退出后，标记原因的err
	err      error             // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
	// 用于子Routine监听退出的channel
	return c.done
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.err
}
```

chilren存的是一个canceler接口
``` go
// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
    // cancel确保可以递归的cancel子routine
	cancel(removeFromParent bool, err error)
    // 参见propagateCancel
	Done() <-chan struct{}
}
```

### 关闭接口
``` go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
    // 返回的第二个参数是调用了cancel函数的函数
	return &c, func() { c.cancel(true, Canceled) }
}

// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
    // 设置了err就表示已经cancel过，不能重复cancel
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
    // 关闭channel，唤醒所有监听的子routine
	close(c.done)
    // 递归cancel子 routine
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	// 这里作为一个bool参数，是因为并非每种ctx都需要removeChild。比如valueCtx就不需要，它只能寄居在另外两种里面
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

正因为如此，所以提供了工具函数，不停地网上遍历，找到最近的一个具有chilren的context
``` go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:  // 人家继承自cancelCtx
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context
		default:
			return nil, false
		}
	}
}
```

从父context中删除就使用了这个工具函数，也就是说他并不是他parent的child
``` go
// removeChild removes a context from its parent.
func removeChild(parent Context, child canceler) {
    // 这里的p并不一定是他的直接parent
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)
	}
	p.mu.Unlock()
}
```

### 构建树
``` bash
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
    // parent无可退出，那自认也就不能包含什么chilren
	if parent.Done() == nil {
		return // parent is never canceled
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
            // parent已经cancel，立即cancel child
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
        	//  构建context树
			if p.children == nil {
				p.children = make(map[canceler]bool)
			}
			p.children[child] = true
		}
		p.mu.Unlock()
	} else {
    	// 按理不会走到这里来，既然parent.Done()不为空，那么肯定实现了cancel/timerCtx
        // 然后到为了确保自行扩展导致的异常情况？？？？？
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

## timerCtx
``` go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
    // 如果当前的过期时间比parent还后，那么就不需要过期了，简单地做个cancelCtx就行了
	if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  deadline,
	}
	propagateCancel(parent, c)
	d := deadline.Sub(time.Now())
	if d <= 0 {
        // 立即执行
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
         // 回调执行
		c.timer = time.AfterFunc(d, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

## valueCtx
``` go

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) String() string {
	return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
    // 往上递归，实现累加传递参数
	return c.Context.Value(key)
}
```
