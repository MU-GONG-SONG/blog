---
title: "什么是“重锁” Heavy Lock?"
date: 2025-10-14
categories: ["Lock"]
tags: ["lock"]
draft: false
---

### 1.什么是“重锁”（Heavy Lock）
>在 Go 性能调优或并发编程中，我们常说的 “重锁”（heavy lock）不是官方术语，而是一个工程上的概念，指的是：锁竞争严重、临界区较大、持锁时间较长的互斥锁（sync.Mutex）。

1.多个 goroutine 同时频繁地去争夺同一把锁；

2.加锁的代码块中做了比较“重”的操作（比如 I/O、JSON 编码、数据库操作）；

导致 goroutine 阻塞、上下文切换频繁，最终造成性能瓶颈。


 
### 2.为什么会出现“重锁”问题


1.临界区太大（锁保护的范围过广）；

2.频繁写操作导致锁争用；

3.使用全局变量或共享状态；

4.没有分片（sharding）或局部化锁机制；

5.锁中包含耗时操作（例如网络请求、磁盘 I/O）。
``````
    var mu sync.Mutex
    var cache = make(map[string]string)
    
    func Set(k, v string) {
    mu.Lock()
    defer mu.Unlock()
    cache[k] = v
    }
    #当高并发调用 Set() 时，所有 goroutine 都在争抢同一把 mu，这就形成“重锁”。
``````

### 3.优化思路与替代方案

#### 3.1 使用 sync.Map
适用于读多写少的场景：

    var m sync.Map
    m.Store("a", 1)
    v, _ := m.Load("a")
    #sync.Map 内部采用分片和原子操作，避免了全局锁竞争。

#### 3.2 使用原子操作（sync/atomic）
   适用于简单的计数、标志位等操作：

    var count int64
    atomic.AddInt64(&count, 1)
    #无锁化操作，性能更高，且不阻塞其他 goroutine。

#### 3.3 优化锁粒度（细化锁）
将一把全局锁拆分成多把局部锁：

    var locks [16]sync.Mutex
    func getLock(key string) *sync.Mutex {
        return &locks[hash(key)%16]
    }
    #减少锁争用，提高并发性能。


