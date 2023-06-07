# 互斥锁

互斥锁是并发控制的一个基本手段，是为了避免竞争而建立的一种并发控制机制    

waitGroup是做并发控制，可以让主进程等待goroutine执行一些耗时的操作

## sync.Mutex
互斥锁Mutex就提供两个方法Lock和Unlock; 进入临界区之前调用Lock方法，退出临界区的时候调用Unlock方法    

当一个goroutine通过调用Lock方法获得了这个锁的拥有权后，其它请求锁的goroutine就会阻塞在Lock方法的调用上，直到锁被释放并且获取到这个锁的拥有权

```go

package main

import (
	"sync"
	"fmt"
)

func main() {
	var wg sync.WaitGroup
	var count = 0
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				count++
			}
		}()
	}
	wg.Wait()
	fmt.Println(count)
}
```

在上面的程序当中，会发现每次执行的结果都不同，这是由于`count++`不是一个原子操作，因此可能会存在并发问题    

针对这个问题，Go提供了一个检测并发访问共享资源是否有问题的工具：`race detector`， 它可以帮助我们自动发现程序有没有data race的问题, 例如执行
`go run -race main.go`    

在上面的程序中，临界区就是`count++`，我们只需要在`count++`的前面加上`Lock`, 在其后面加上`Unlock`即可达到我们想要的效果输出`1000000`, 代码如下
```go

package main

import (
	"sync"
	"fmt"
)

func main() {
	var mu sync.Mutex
	var count = 0
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				mu.Lock()
				count++
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Println(count)
}
```


在很多情况下, *Mutex会嵌入到其它struct中使用*，比如下面的方式
```go
package main

import (
	"sync"
	"fmt"
)

type Counter struct {
	mu sync.Mutex
	Count uint64
}

func main() {
	var wg sync.WaitGroup
	var counter Counter
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				counter.mu.Lock()
				counter.Count++
				counter.mu.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Println(counter.Count)
}
```

## sync.Mutex使用错误的常见4种场景
使用Mutex常见的错误场景有4类，分别是Lock/Unlock不是成对出现、Copy已经使用的锁、重入和死锁


### Lock/Unlock没有成对出现
Lock/Unlock没有成对出现，就意味着会出现死锁的情况，或者是因为Unlock一个未加锁的Mutex而导致panic   

缺少Unlock的场景，常见的有三种情况:
- 代码中有太多if-else分支，可能在某个分支中漏写了Unlock
- 在重构的时候把Unlock给删除了
- Unlock误写成Lock   

在上面的这种情况，锁被获取之后，就不会被释放了，这也意味着其它的goroutine永远都没机会获取到锁。    


缺少Lock的场景一般就是误删了Lock


### Copy已使用的Mutex
sync包在使用之后是不能被复制的，Mutex也是不能复制的。    

原因在于, Mutex是一个有状态的对象，它的state字段记录这个锁的状态。如果要复制一个已经加锁的Mutex给一个新的变量，那么新的刚初始化的变量就已经被枷锁了，这显然不是我们想要的，因为我们想要的是一个零值的Mutex

```go
package main

import (
	"sync"
	"fmt"
)

type Counter struct {
	mu sync.Mutex
	Count uint64
}

func main() {
	var c Counter
	c.mu.Lock()
	defer c.mu.Unlock()
	c.Count++
	foo(c) // 复制锁
}

func foo(c Counter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	fmt.Println("in foo")
}
```
上面的程序会导致panic, 原因就是使用一个加锁的锁进行初始化

### 重入

**重入锁**： 当一个线程获取锁时，如果没有其它线程拥有这个锁，那么，这个线程就能成功获取到这个锁。之后，如果其它线程再请求这把锁的话，就会出现阻塞等待的状态。当时，如果拥有这把锁的线程再请求这把锁的话，不会阻塞，而是成功返回，所以叫可重入锁。    

需要注意的是: **Mutex**是不可重入锁, 例如
```go

package main

import (
	"sync"
	"fmt"
)

func main() {
	l := &sync.Mutex{}
	foo(l)
}

func foo(l sync.Locker) {
	fmt.Println("in foo")
	l.Lock()
	bar(l)
	l.Unlock()
}

func bar(l sync.Locker) {
	l.Lock()
	fmt.Println("in bar")
	l.Unlock()
}


```

上面的程序会报错: fatal error: all goroutines are asleep - deadlock    

如何实现一个可重入锁，有两种方案：
1. 通过hacker的方式获取到goroutine id， 记录下获取锁的goroutine id, 它可实现Locker接口
2. 调用Lock/Unlock方法时，由goroutine提供一个token，用来标识它自己，而不是我们通过hacker的方式获取到goroutine id。但是，这样一来，就不满足Locker接口了   


可重入锁(递归锁)解决了代码重入或者递归调用带来的死锁问题


#### goroutine id

这个方案的关键一步是获取goroutine id, 方式有两种，分别是简单方式和hacker方式    

简单方式：获取goroutine的方式可以通过runtime.Stack方法获取，例如
```go
func GoID() int {
	var buf [64]byte
	n := runtime.Stack(buf[:], false)

	idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine"))
	id, err := strconv.Atoi(idField[0])
	if err != nil {
		panic(fmt.Sprintf("cannot get goroutine id: %v", err))
	}
	return id

}
```

通过hacker的方式获取：   
获取运行时的g指针， 反解出对应的g结构。每个运行的goroutine结构的g指针保存在当前的goroutine的一个叫TLS对象中。    
第一步: 获取到TLS对象；    
第二步：再从TLS中获取goroutine结构的g指针
第三步：再从g指针取出goroutine id

可以使用pertermattis/goid第三方库获取goroutine id, 下面实现一个可重入锁
```go
package main

import (
	"fmt"
	"github.com/petermattis/goid"
	"sync"
	"sync/atomic"
)

type RecursiveMutex struct {
	sync.Mutex
	owner     int64 // 当前持有锁的goroutine id
	recursion int32  // 这个goroutine重入的次数
}

func (m *RecursiveMutex) Lock() {
	gid := goid.Get()
  // 当前持有锁的goroutine就是这次调用的goroutine， 说明重入
	if atomic.LoadInt64(&m.owner) == gid {
		m.recursion++
		return
	}
	m.Mutex.Lock()
  // 获得锁的goroutine第一次调用，记录下它的goroutine id, 调用次数加一
	atomic.StoreInt64(&m.owner, gid)
	m.recursion = 1
}

func (m *RecursiveMutex) Unlock() {
	gid := goid.Get()
  // 非持有锁的goroutine尝试释放锁，错误的使用
	if atomic.LoadInt64(&m.owner) != gid {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", m.owner, gid))
	}
	m.recursion--
  // 如果这个goroutine还没有完全释放，则直接返回
	if m.recursion != 0 {
		return
	}
  // 此goroutine最后一次调用，需要释放锁
	atomic.StoreInt64(&m.owner, -1)
	m.Mutex.Unlock()
}

func main() {
	l := &RecursiveMutex{}
	foo(l)
}

func foo(l *RecursiveMutex) {
	fmt.Println("in foo")
	l.Lock()
	bar(l)
	l.Unlock()
}

func bar(l *RecursiveMutex) {
	l.Lock()
	fmt.Println("in bar")
	l.Unlock()
}
```

#### token
调用者自己提供一个token， 获取锁的时候把这个token传入, 释放锁的时候也需要把这个token传入。通过用户传入的token替换方案一中的goroutine

```go
type TokenRecursiveMutex struct {
	sync.Mutex
	token     int64
	recursion int32
}

func (m *TokenRecursiveMutex) Lock(token int64) {
	// 如果传入的token和持有锁的token一致
	if atomic.LoadInt64(&m.token) == token { 
		m.recursion++
		return
	}
	// 传入的token不一致，说明不是递归调用
	m.Mutex.Lock()
	// 抢到锁之后记录这个token
	atomic.StoreInt64(&m.token, token)
	m.recursion = 1
}

func (m *TokenRecursiveMutex) Unlock(token int64) {
	if atomic.LoadInt64(&m.token) != token {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", m.token, token))
	}
	m.recursion--
	if m.recursion != 0 {
		return
	}
	// 没有递归调用，释放锁
	atomic.StoreInt64(&m.token, 0)
	m.Mutex.Unlock()
}
```

### 死锁

两个或两个以上的进程(或线程，goroutine)在执行的过程中，因争夺共享资源而处于一种相互等待的状态，如果没有外部干涉，它们都将无法推进下去，此时，我们称系统处于死锁状态或系统产生了死锁。死锁产生的条件有：
- 互斥：至少有一个资源是被排他性独享的，其它线程必须处于等待状态，直到资源被释放
- 持有和等待： goroutine持有一个资源，并且还在请求其它goroutine持有的资源
- 不可剥夺： 资源只能由持有它的goroutine来释放
- 环路等待：一般来说，存在一组等待进程，P={P1,P2,P3...}，P1等待P2持有的资源，P2等待P3持有的资源，依此类推，最后PN等待P1持有的资源，这就形成了一个环路等待的死结

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var mu1 sync.Mutex
	var mu2 sync.Mutex

	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()

		mu1.Lock()
		defer mu1.Unlock()

		time.Sleep(5 * time.Second)

		mu2.Lock()
		mu2.Unlock()
	}()

	go func() {
		defer wg.Done()

		mu2.Lock()
		defer mu2.Unlock()

		time.Sleep(5 * time.Second)
		mu1.Lock()
		mu1.Unlock()
	}()

	wg.Wait()
	fmt.Println("成功完成")
}
```

上面的程序由于mu1和mu2发生环路等待，因此导致死锁，所以该程序无法运行成功


## Mutex进阶

锁是性能下降的“罪魁祸首”之一，所以有效地降低所得竞争，就能够很好地提高性能。因此，监控关键互斥锁上等待的goroutine的数量，是分析锁竞争的激烈程度的一个指标

###  TryLock

当一个goroutine调用这个TryLock方法请求锁的时候，如果这把锁没有被其它goroutine所持有，那么这个goroutine就持有了这把锁，并返回true; 如果这把锁已经被其它goroutine所持有，或者是正在准备交给某个被唤醒的goroutine，那么，这个请求锁的goroutine就直接返回false， 不会阻塞在方法调用上。

```go



func main() {
	var mu Mutex
	go func() {
		mu.Lock()
		time.Sleep(time.Duration(rand.Intn(2)) * time.Second)
		mu.Unlock()
	}()
	
	time.Sleep(time.Second)

	ok := mu.TryLock()

	if ok {
		fmt.Println("got the lock")
		mu.Unlock()
		return
	}

	fmt.Println("can not get the lock")
}

const (
	mutexLocked      = 1 << iota // 加锁标识位置1 =》2的0次方
	mutexWoken                   // 唤醒标识位置2 =》2的1次方
	mutexStarving                // 锁饥饿标识位置4 =》2的2次方
	mutexWaiterShift = iota      // 标识waiter的起始bit位置3
)

type Mutex struct {
	sync.Mutex
}

func (m *Mutex) TryLock() bool {
	if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), 0, mutexWaiterShift) {
		return true
	}
	old := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
	if old&(mutexLocked|mutexStarving|mutexWoken) != 0 {
		return false
	}
	new := old | mutexLocked
	return atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), old, new)
}

```

### 使用Mutex实现一个线程安全的队列

```go
package main

import "sync"

type SliceQueue struct {
	data []interface{}
	mu sync.Mutex
}

func NewSliceQueue(n int) (q *SliceQueue) {
	return &SliceQueue{data: make([]interface{}, 0, n)}
}

func (q *SliceQueue) Enqueue(v interface{}) {
	q.mu.Lock()
	q.data = append(q.data, v)
	q.mu.Unlock()
}

func (q *SliceQueue) Dequeue() interface{} {
	q.mu.Lock()
	if len(q.data) == 0 {
		q.mu.Unlock()
		return nil
	}
	v := q.data[0]
	q.data = q.data[1:]
	q.mu.Unlock()
	return v
}
```