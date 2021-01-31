---
title: go 标准库的 heap
date: 2021-01-31 22:05:38
tags: [golang, 源码, 堆]
categories: golang
---

go 标准库提供的 [堆](https://en.wikipedia.org/wiki/Heap_(data_structure)) 虽然在 [`container`](https://github.com/golang/go/tree/master/src/container/heap) 包里，但它其实并没有像 `list` 和 `ring` 那样提供一个实实在在的容器，事实上它定义了一个如下的接口和一系列堆的基本操作，只要是实现了该接口的结构就可以被认为是一个堆，从而把堆的操作方法用到该结构上。

```go
type Interface interface {
    // 排序接口
    sort.Interface
    // 追加元素
    Push(x interface{})
    // 移除首个元素
    Pop() interface{}
}
```

**注意** `Interface` 中的 `Push` 和 `Pop` 方法是针对要实现该接口的数据接口本身的追加和移除操作，并非我们通常说的堆的 `Push` 和 `Pop` 方法，在使用堆时切记使用的是 `heap.Push(h, x)` 和 `heap.Pop(h)` 。

## 堆的初始化

```go
// 把给定的结构初始化成一个 最小堆
func Init(h Interface) {
    n := h.Len()
    // i = n / 2 - 1
    // 2i + 1 = n - 1 => i 是 非叶子节点
    // 对所有非叶子节点进行下沉操作
    for i := n/2 - 1; i >= 0; i-- {
        down(h, i, n)
    }
}

// 节点下沉操作
// n: 下沉操作的界限
func down(h Interface, i0, n int) bool {
    i := i0
    for {
        j1 := 2*i + 1
        // 下沉是考虑的节点不超过 n
        // 可以观察下 Init/Fix 和 Pop/Remove 时 n 的值
        if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
            break
        }
        j := j1 // left child
        // 选出左、右子节点中较小的
        if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
            j = j2 // = 2*i + 2  // right child
        }
        if !h.Less(j, i) {
            // 子节点不小于父节点 (已经是有序的)
            break
        }
        // 父节点与左、右子节点中较小的进行交换
        h.Swap(i, j)
        // 原来的父节点已经变为子节点，考虑是否还要继续下沉
        i = j
    }
    // 返回 i0 节点是否下沉了
    return i > i0
}
```

## 插入元素

```go
func Push(h Interface, x interface{}) {
    h.Push(x)
    // 添加的元素是堆对应二叉树的最右下的节点
    // 考虑该元素是否需要上浮
    up(h, h.Len()-1)
}

// 元素上浮操作
func up(h Interface, j int) {
    for {
        i := (j - 1) / 2 // parent
        if i == j || !h.Less(j, i) {
            // 该元素不小于父节点
            break
        }
        // 与父节点交换
        h.Swap(i, j)
        // 考虑是否需要继续上浮
        j = i
    }
}
```

## 弹出元素与移除某个元素

```go
// 移除堆的根节点
func Pop(h Interface) interface{} {
    n := h.Len() - 1
    // 根节点与最后一个（最右下）元素交换
    h.Swap(0, n)
    // 对交换后的根节点进行下沉操作
    // 注意下沉时的界限为 倒数第二个 元素 （不包括最后一个元素，否则就白交换了）
    down(h, 0, n)
    // 移除最后一个元素，即原来的根节点
    return h.Pop()
}

// 移除堆的第 n 个节点
func Remove(h Interface, i int) interface{} {
    n := h.Len() - 1
    // 如果是最后一个节点的话直接移除
    if n != i {
        h.Swap(i, n)
        if !down(h, i, n) {
            // 交换、下沉与 Pop 类似
            // 因为 i 不一定是根节点，如果有下沉操作的话还要考虑是否要进行上浮
            up(h, i)
        }
    }
    return h.Pop()
}
```

## 修复

```go
// 节点 i 的值发生变化后需要对堆进行修复
func Fix(h Interface, i int) {
    // 先下沉
    if !down(h, i, h.Len()) {
        // 如果有下沉操作则考虑上浮
        up(h, i)
    }
}
```

## 使用示例

使用堆实现 [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) 来解决 [Leetcode 1631. 最小体力消耗路径](https://leetcode-cn.com/problems/path-with-minimum-effort/) 问题：

```go
type node struct {
    x, y, distance int
}

// 定义堆
type nodeHeap []*node

func (h nodeHeap) Len() int {
    return len(h)
}

func (h nodeHeap) Less(i, j int) bool {
    return h[i].distance < h[j].distance
}

func (h nodeHeap) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
}

func (h *nodeHeap) Pop() interface{} {
    tail := len(*h) - 1
    x := (*h)[tail]
    *h = (*h)[0:tail]
    return x
}

func (h *nodeHeap) Push(x interface{}) {
    *h = append(*h, x.(*node))
}

// 4 个方向
var direct = [4][2]int{
    {0, 1},
    {0, -1},
    {1, 0},
    {-1, 0},
}

func max(x, y int) int {
    if x > y {
        return x
    }
    return y
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}

func minimumEffortPath(heights [][]int) int {
    const infinity = 10000000
    m, n := len(heights), len(heights[0])

    maxDis := make([][]int, m)
    for i:=0;i<m;i++{
        maxDis[i] = make([]int, n)
        for j:=0;j<n;j++{
            maxDis[i][j] = infinity
        }
    }

    nh := &nodeHeap{{
        x: 0,
        y: 0,
    }}
    heap.Init(nh)
    for {
        p := heap.Pop(nh).(*node)
        if p.x == m-1 && p.y == n-1 {
            return p.distance
        }
        for _, d := range direct {
            x, y := p.x + d[0], p.y + d[1]
            if x >= 0 && x < m && y >= 0 && y < n {
                d := max(p.distance, abs(heights[p.x][p.y] - heights[x][y]))
                if d < maxDis[x][y] {
                    maxDis[x][y] = d
                    heap.Push(nh, &node{
                        x: x,
                        y: y,
                        distance: d,
                    })
                }
            }
        }
    }

    return 0
}
```
