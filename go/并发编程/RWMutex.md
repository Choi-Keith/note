# 读写锁
在写少读多的情况下，如果使用Mutex来保证读写共享资源的安全性时，可能会造成性能下降，并且没有充分发挥并发特性。

## RWMutex

RWMutex是一个互斥锁，RWMutex在某一时刻只能由任意数量的reader持有，或者是只被单个writer持有。RWMutex的方法由几个：
- Lock/Unlock: **写操作调用的方法**。如果锁已被reader或writer持有，那么Lock方法会一直阻塞，直到能获取锁；Unlock则是配对的释放锁的方法
- RLock/RUnlock: **读操作调用的方法**。如果锁已被writer持有的话，RLock方法会一直阻塞，直到能获取锁，否则则直接返回；而RUnlock是reader释放锁的方法。
- RLocker: 这个方法的作用是为读操作返回一个Locker接口的对象。它的Lock方法会调用RWMutex的RLock方法，它的Unlock方法RWMutex的RUnlock方法

下面以计数器为例，如何使用RWMutex保护共享资源。
```go

package main

import (
	"fmt"
	"sync"
	"time"
)

type Counter struct {
	mu    sync.RWMutex
	Count uint64
}

func main() {
	var counter Counter
	for i := 0; i < 10; i++ {
		go func() {
			for {
				num := counter.read()
				fmt.Printf("%d\n", num)
			}
		}()
	}

	for {
		counter.Incr()
		time.Sleep(time.Second)
	}
}


func (c *Counter) Incr() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.Count++
}

func (c *Counter) read() uint64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.Count
}

```
使用RWMutex，可以极大提高读取的速度，因为是并发进行的。如果使用Mutex, 性能不会想RWMutex那么好，因为多个reader并发读的时候，使用互斥锁导致reader要排队读的情况。    

**在Go标准库中的RWMutex设计是采用写优先的方案。一个正在阻塞的Lock调用会排除新的reader请求到锁**    


## RWMutex的3个踩坑点

### 1. 不可复制

不可复制的原因其实和Mutex一样。一旦读写锁被使用，它的字段就会记录它当前的一些状态。


### 2. 重入导致死锁

### 3. 释放未加锁的RWMutex
和互斥锁一样，Lock和Unlock的调用总是成对出现的，RLock和RUnLock的调用也必须成对出现的。Lock和RLock多余的调用会导致锁没有被释放，可能会出现死锁的情况，而Unlock和RUnlock多余的调用会导致panic

