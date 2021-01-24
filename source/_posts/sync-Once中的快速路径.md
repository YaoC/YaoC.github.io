---
title: sync.Once中的快速路径
date: 2021-01-25 00:23:27
tags: [golang, 源码, 编程范式]
categories: golang
---

`sync.Once` 的 [`Do`](https://github.com/golang/go/blob/b634f5d97a6e65f19057c00ed2095a1a872c7fa8/src/sync/once.go#L42) 方法中有这样一个技巧：先用原子操作 **快速** 判断是否执行过, 如果没有执行过（`done == 0`）再加锁执行 `f` 方法。

``` Go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

其实这里完全可以直接执行 `doSlow` 方法，因为它内部已经加锁了，即便有并发执行也不会有任何问题。

``` Go
func (o *Once) Do(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

之所以要多此一举是因为 **原子操作的性能要优于加锁** ，对于 `Do` 方法来说，大部分调用 `done` 都是不等于 0 的，因此这样做避免了对大部分的调用进行加锁判断（只有最初的 **一次或几次并发调用** 会进入到 `doSlow` 方法），是一种非常经典的编程范式。
