---
title: Kubernetes 源码阅读 -- Expiring
date: 2021-01-02 21:54:45
tags: [k8s, 源码, cache, apimachinery]
categories: kubernetes
---

## 概述

[`Expiring`](https://github.com/kubernetes/apimachinery/blob/v0.20.1/pkg/util/cache/expiring.go) 是 Kubernets 中的一个 **带有过期时间的键值对缓存**，位于 `k8s.io/apimachinery` 模块的 `pkg/util/cache` 包中。

`Expiring` 具有以下功能与特性：

- 可以存取任意类型的键值对
- 键值对在过期后会被自动清除
- 线程安全

``` Go
// 添加带过期时间的键值对
func (c *Expiring) Set(key interface{}, val interface{}, ttl time.Duration)

// 根据 key 获取 value
func (c *Expiring) Get(key interface{}) (val interface{}, ok bool)

// 根据 key 删除键值对
func (c *Expiring) Delete(key interface{})

// 获取 Expiring 中存储的键值对数量
func (c *Expiring) Len()
```

## Expiring 的数据结构

基于上述的功能与特性，`Expiring` 提供了以下方法：

`Expiring` 的数据结构设计如下：

``` Go
type Expiring struct {
    // 时钟接口，使用该接口主要是为了便于测试
    // 在 Expiring 中用到其中的 Now 方法，该方法可以直接看作 time.Now
    clock utilclock.Clock
    // 读写锁，用于保证线程安全
    mu sync.RWMutex
    // 实际存储键值对的数据结构
    cache map[interface{}]entry
    // 全局版本号，用来实现键值对过期时间的更新
    generation uint64
    // 存储过期时间的最小堆
    heap expiringHeap
}
```

其中 `entry` 的数据结构如下：

``` Go
type entry struct {
    // 用户输入的 value
    val        interface{}
    // 用户指定的过期时间
    expiry     time.Time
    // 存储当前键值对时 Expiring 的版本号
    generation uint64
}
```

### expiringHeap

[`expiringHeap`](https://github.com/kubernetes/apimachinery/blob/v0.20.1/pkg/util/cache/expiring.go#L168) 是一个存储键值对过期时间的最小堆，其数据结构如下：

``` Go
type expiringHeap []*expiringHeapEntry

type expiringHeapEntry struct {
    // 用户输入的 key
    key        interface{}
    // 过期时间
    expiry     time.Time
    // 存储当前键值对时 Expiring 的版本号
    generation uint64
}

// expiringHeap 实现了 heap.Interface 接口
// 即可以当作堆来使用
var _ heap.Interface = &expiringHeap{}

// 比较的是过期时间
// expiringHeapEntry 中的元素 i 小于 元素 j 当且仅当 i 的过期时间小于 j 的过期时间
func (cq expiringHeap) Less(i, j int) bool {
    return cq[i].expiry.Before(cq[j].expiry)
}

// heap.Interface 的其它方法的实现较为常见，在此忽略
```

## Expiring 的实现

### Set

``` Go
func (c *Expiring) Set(key interface{}, val interface{}, ttl time.Duration) {
    // 计算超时时间
    now := c.clock.Now()
    expiry := now.Add(ttl)

    // 加写锁保证线程安全
    c.mu.Lock()
    defer c.mu.Unlock()

    // 全局版本号自增
    c.generation++

    // 存储 value
    c.cache[key] = entry{
        val:        val,
        expiry:     expiry,
        generation: c.generation,
    }

    // 清理已过期的键值对
    // Run GC inline before pushing the new entry.
    c.gc(now)

    // 过期时间存入最小堆中
    heap.Push(&c.heap, &expiringHeapEntry{
        key:        key,
        expiry:     expiry,
        generation: c.generation,
    })
}
```

#### Get

``` Go
func (c *Expiring) Get(key interface{}) (val interface{}, ok bool) {
    // 加读锁保证线程安全
    c.mu.RLock()
    defer c.mu.RUnlock()
    e, ok := c.cache[key]
    if !ok || !c.clock.Now().Before(e.expiry) {
        // 如果键值对存在但已过期则当作不存在返回
        return nil, false
    }
    return e.val, true
}
```

### 清理过期的键值对

``` Go
// 将 Expiring 所有过期的键值对清除
func (c *Expiring) gc(now time.Time) {
    for {
        if len(c.heap) == 0 || now.Before(c.heap[0].expiry) {
            // 因为过期时间的 entry 以最小堆的形式组织
            // 所以如果第一个元素没有过期的话其余的元素也肯定没有过期
            return
        }
        cleanup := heap.Pop(&c.heap).(*expiringHeapEntry)
        // 执行清理操作
        c.del(cleanup.key, cleanup.generation)
    }
}

func (c *Expiring) del(key interface{}, generation uint64) {
    e, ok := c.cache[key]
    if !ok {
        return
    }
    // generation == 0 标识该方法是由 Delete 调用的，键值对一定需要删除
    // generation != 0 则说明该方法由 gc 调用
    //      当 gc 调用该方法时如果该键值对在 cache 中存储的过期时间与在最小堆中存储的版本号不一致则说明
    //      该键值对被更新过，当前的过期时间 entry 已经无效了，忽略就行（堆中还存在着该键值对的其它过期时间 entry）
    if generation != 0 && generation != e.generation {
        return
    }
    delete(c.cache, key)
}

func (c *Expiring) Delete(key interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.del(key, 0)
}
```

### utilclock.Clock

在测试 `Expiring` 时，添加键值对需要指定过期时间，这可能导致测试时间变得很长，
`Expiring` 中引入 [`utilclock.Clock`](https://github.com/kubernetes/apimachinery/blob/v0.20.1/pkg/util/clock/clock.go) 能够让测试变得更加简单迅速。

在实际使用时，`Expiring` 使用的是 [`RealClock`](https://github.com/kubernetes/apimachinery/blob/v0.20.1/pkg/util/clock/clock.go#L43)，它使用的是真实的时间。

而在测试时则可以使用 [`FakeClock`](https://github.com/kubernetes/apimachinery/blob/v0.20.1/pkg/util/clock/clock.go#L85) 来 “快进” 到任意时间。

下一篇文章将对 `Clock` 进行详细的介绍。
