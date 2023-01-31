# golang/Context

## 初体验

假设一个场景：发起一个网络请求Request, 一般每个Request会开启一个goroutine去完成一些工作，而且，这个goroutine也有可能会再起一个goroutine。那么，我们想要管理这些复杂关系的goroutine的时候，就比较麻烦了。**官方实现的Context库可以将上述场景中使用的多个goroutine生成一个 树结构 去记录他们之间的关系 和 各个协程使用到的上下文，从而更方便的去管理这些goroutine。**

### 事例

#### 需求

我们在main函数中，起两个goroutine，假设这两个goroutine一直在listening,现在我们想要主动的通知这两个goroutine结束，应该如何实现控制？

#### 使用channel + select

```go
package main

import (
	
	"fmt"
	"time"
)

func main(){
	stop := make(chan bool)
	
	go func() {
		for{
			select {
			case <-stop:
				fmt.Println("1 listen stop.")
			default:
				fmt.Println("1 listening...")
				time.Sleep(1* time.Second)
			}
		}
	}()
	
	go func() {
		for{
			select {
			case <-stop:
				fmt.Println("2 listen stop.")
			default:
				fmt.Println("2 listening...")
				time.Sleep(1* time.Second)
			}
		}
	}()
	
	time.Sleep( 5 * time.Second)
	fmt.Println("we will stop listen")
	stop <- true
	time.Sleep(5 * time.Second)
}
```

可以看到，我们会对goroutine有各种各样的控制需求。channel + select 就是官方提供的一种**并发控制**实现。

下面再来看

#### Context实现方式

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("1 listen stop.")
				return
			default:
				fmt.Println("1 listen...")
				time.Sleep(2 * time.Second)
			}
		}
	}(ctx)
	
	go func(ctx context.Context) {
		for{
			select {
			case <-ctx.Done():
				fmt.Println("2 listen stop.")
				return
			default:
				fmt.Println("2 listen...")
				time.Sleep(2 * time.Second)
			}
		}
	}(ctx)

	time.Sleep(10 * time.Second)
	fmt.Println("we will stop listen")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}
```

这里使用Context也实现了预期的效果。

总的来说：context是官方提供的一组**并发控制**的工具。

那和其他的并发控制的实现有什么区别呢？请看下文。

## 具体实现

go SDK/src/context/context.go的实现简洁且比较独立，直接看源码结构。

![1637631799272](C:\Users\turli\AppData\Roaming\Typora\typora-user-images\1637631799272.png)

### Context interface

首先Context就是一个接口，有四个方法。

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

#### Done

可以看到返回值是一个**只读**的channel, 一般的，我们用这个channel传递context被取消的信号。此channel会在当前工作完成或者上下文被取消后关闭。

#### Err

`Err()` 返回一个错误，表示 channel 被关闭的原因。例如是被取消，还是超时。

#### Deadline

`Deadline()` 返回 context 的截止时间，被取消的时间。

#### Value

此方法可以获取key对应的值，一般化理解，就是可以让协程之间传递请求特定的数据。

简单看一下就好了。接合下面的再来更好的理解。

### 对Context接口的实现

#### emptyCtx

emptyCtx的定义如下

```go
type emptyCtx int
```

##### 相关代码：

```go
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

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
    return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
    return todo
}
```

emptyCtx实现了Context接口。但是看实现的方法中，都是`return nil`。我们就知道其实emptyCtx不具备任何特定的功能。那么他存在的意义是啥呢？  -- 为了创建**协程树结构的根节点。**

##### Background & TODO 

此两个函数，返回了emptyCtx的实例。我们通常使用Background函数创建context的根节点。上面事例中Context实现方式中的

`ctx, cancel := context.WithCancel(context.Background())`就是这个含义。

TODO函数一般用来暂时不太确定传递什么context的情况。

#### cancelCtx

```go
type cancelCtx struct {
   Context

   mu       sync.Mutex            // protects following fields
   done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
   children map[canceler]struct{} // set to nil by the first cancel call
   err      error                 // set to non-nil by the first cancel call
}
```

我们经常用cancelCtx类型去实现 手动取消，超时取消等并发控制的实例。

mu是并发控制安全。done是用来通知。children是线程安全的map，记录子协程的一些信息。

##### 相关代码：

```go
func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})
    }
    d := c.done
    c.mu.Unlock()
    return d
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

##### Done



##### cancel(重点)



#### timerCtx

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

在`cancelCtx` 基础上增加了字段 `timer` 和 `deadline`

`timer` 触发自动cancel的定时器

`deadline` 标识最后执行cancel的时间

##### 相关代码

```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) String() string {
    return fmt.Sprintf("%v.WithDeadline(%s [%s])", c.cancelCtx.Context, c.deadline, time.Until(c.deadline))
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

#### valueCtx

```go
type valueCtx struct {
	Context
	key, val interface{}
}
```

在`Context` 的基础上增加了元素 `key` 和 `value`

通常用于在层级协程之间传递数据。

##### 相关代码：

```go
func (c *valueCtx) String() string {
   return contextName(c.Context) + ".WithValue(type " +
      reflectlite.TypeOf(c.key).String() +
      ", val " + stringify(c.val) + ")"
}

func (c *valueCtx) Value(key interface{}) interface{} {
   if c.key == key {
      return c.val
   }
   return c.Context.Value(key)
}
```

### 实例化

**其中涉及到的函数逻辑细节见<a href="">连接</a>**



#### Background & TODO

创建最 `emptyCtx` 实例 ,通常是作为根节点和不暂时不确定的节点，由于简单上面已经说过了。

#### withCancel

创建cancelCtx实例

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   c := newCancelCtx(parent)
   propagateCancel(parent, &c)
   return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
   return cancelCtx{Context: parent}
}

// goroutines counts the number of goroutines ever created; for testing.
var goroutines int32

// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
   done := parent.Done()
   if done == nil {
      return // parent is never canceled
   }

   select {
   case <-done:
      // parent is already canceled
      child.cancel(false, parent.Err())
      return
   default:
   }

   if p, ok := parentCancelCtx(parent); ok {
      p.mu.Lock()
      if p.err != nil {
         // parent has already been canceled
         child.cancel(false, p.err)
      } else {
         if p.children == nil {
            p.children = make(map[canceler]struct{})
         }
         p.children[child] = struct{}{}
      }
      p.mu.Unlock()
   } else {
      atomic.AddInt32(&goroutines, +1)
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

##### propagateCancel



#### withValue

创建valueCtx实例

```go
func WithValue(parent Context, key, val interface{}) Context {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   if key == nil {
      panic("nil key")
   }
   if !reflectlite.TypeOf(key).Comparable() {
      panic("key is not comparable")
   }
   return &valueCtx{parent, key, val}
}

// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
   Context
   key, val interface{}
}
```

#### withDeadline

创建timerCtx实例

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   if cur, ok := parent.Deadline(); ok && cur.Before(d) {
      // The current deadline is already sooner than the new one.
      return WithCancel(parent)
   }
   c := &timerCtx{
      cancelCtx: newCancelCtx(parent),
      deadline:  d,
   }
   propagateCancel(parent, c)
   dur := time.Until(d)
   if dur <= 0 {
      c.cancel(true, DeadlineExceeded) // deadline has already passed
      return c, func() { c.cancel(false, Canceled) }
   }
   c.mu.Lock()
   defer c.mu.Unlock()
   if c.err == nil {
      c.timer = time.AfterFunc(dur, func() {
         c.cancel(true, DeadlineExceeded)
      })
   }
   return c, func() { c.cancel(true, Canceled) }
}
```

#### withTimeout

实际是调用了withDeadline

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   return WithDeadline(parent, time.Now().Add(timeout))
}
```
