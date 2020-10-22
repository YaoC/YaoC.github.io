---
title: Kubernetes 源码阅读 -- Indexer
date: 2020-10-22 23:36:37
tags: [k8s, 源码, client-go]
categories: kubernetes
---

## 概述

[`Indexer`](https://github.com/kubernetes/client-go/blob/d1a4fe5f2d96df815903781843870155cb4f5f40/tools/cache/index.go#L35) 是 kubernetes 中用于在内存中缓存资源对象的接口，支持通过 Key 对存储的资源对象进行增、删、查、改操作， 同时也可以对存储的对象进行倒排索引。

`cache` 是 `Indexer` 的实现方式，其定义如下：

``` Go
type cache struct {
    cacheStorage    ThreadSafeStore
    keyFunc         KeyFunc
}
```

`ThreadSafeStore` 定义了一个线程安全的存储接口，由 `threadSafeMap` 实现。

ThreadSafeStore 与 Indexer 非常相似，前者可以看作是后者的一种特例。ThreadSafeStore 操作资源对象时需要指定对象的 key，而 Indexer 则不需要，是在 Indexer 的实现结构 cache 中通过 `keyFunc` 动态计算资源对象的 key。因此可以认为 `Indexer = ThreadSafeStore + KeyFunc`。

## Indexer

Indexer 与其它数据结构的关系如下：
![Indexer UML](http://lc-C7OxrU7U.cn-n1.lcfile.com/c34bc3865ff4230a648a.png/diagram-16025179975213868166.png)

从图中可以看出没有索引的 cache 就是一个 Store，源码中 Indexer 和 Store 的创建方式分别如下：

```GO
// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
    return &cache{
        cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
        keyFunc:      keyFunc,
    }
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
    return &cache{
        cacheStorage: NewThreadSafeStore(indexers, Indices{}),
        keyFunc:      keyFunc,
    }

type KeyFunc func(obj interface{}) (string, error)
```

`ThreadSafeStore` 通过 `indexers` 创建索引，indexers 是一个如下的 map：

```GO
type Indexers map[string]IndexFunc

type IndexFunc func(obj interface{}) ([]string, error)
```

Indexers 的 Key 是索引名称，值是 _根据资源对象创建计算该索引的索引值的函数_

`Indices` 是存储倒排索引的数据结构：

```Go
type Indices map[string]Index

type Index map[string]sets.String
```

Indices 的 Key 是索引名, 值是对应的索引 Index
Index 的 Key 是索引值，值是具有该索引值的资源对象的 Key

## KeyFunc 与 IndexFunc

KeyFunc 与 IndexFunc 对比如下：

- KeyFunc: f(obj) -> key
- IndexFunc: f(obj) -> indexValue1, indexValue2 ...

由上对比可知 KeyFunc 是 IndexFunc 的一个特例，当 IndexFunc 只返回一个索引值时它就退化为 KeyFunc，client-go 中也提供了由 IndexFunc 到 KeyFunc 的适配函数：

```Go
func IndexFuncToKeyFuncAdapter(indexFunc IndexFunc) KeyFunc {
    return func(obj interface{}) (string, error) {
        indexKeys, err := indexFunc(obj)
        if err != nil {
            return "", err
        }
        if len(indexKeys) > 1 {
            return "", fmt.Errorf("too many keys: %v", indexKeys)
        }
        if len(indexKeys) == 0 {
            return "", fmt.Errorf("unexpected empty indexKeys")
        }
        return indexKeys[0], nil
    }
}
```

常用的 KeyFunc 有 `MetaNamespaceKeyFunc`, 即 f(obj) -> namespace/name
常用的 IndexFunc 则有 `MetaNamespaceIndexFunc`, 即 f(obj) -> [namespace]

## threadSafeMap

`threadSafeMap` 是 `ThreadSafeStore` 的具体实现，通过 `sync.RWMutex` 实现线程安全。threadSafeMap 只能保证其定义的操作是线程安全的，直接对 Get/List 等操作返回的结果进行修改是无法保证的。

threadSafeMap 的数据结构如下，它的增删查改操作主要就是操作 `items`，同时通过 indexers 计算资源对象在各个索引中的值，通过  `updateIndices` 和 `deleteFromIndices` 对 `indices` 进行更新。

```Go
type threadSafeMap struct {
    lock  sync.RWMutex
    items map[string]interface{}

    // indexers maps a name to an IndexFunc
    indexers Indexers
    // indices maps a name to an Index
    indices Indices
}
```
