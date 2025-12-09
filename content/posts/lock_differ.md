---
title: "Go 中的 Lock使用?"
date: 2025-12-09
categories: ["Lock"]
tags: ["lock"]
draft: false
---
>锁（Lock）是并发编程中用于保护共享资源、防止数据竞争（data race）的关键同步原语。
### 1.sync.Mutex (互斥锁)
* 特点
~~~
同一时间只允许一个 goroutine 持有锁。
不可重入（reentrant）：同一个 goroutine 再次调用 Lock() 会导致死锁。

var mu sync.Mutex
var count int
func increment() {
    mu.Lock()
    defer mu.Unlock()
    count++
}
~~~
 * 注意
~~~
必须配对使用：每个 Lock() 必须对应一个 Unlock()。
避免在持有锁时进行阻塞操作（如 I/O、channel 发送/接收），否则会降低并发性能。
不要复制已使用的 Mutex（因为其内部状态不可复制）。
~~~
### 2.sync.RWMutex 读写锁

* 特点
~~~
支持 多个 reader 或 一个 writer。
读锁（RLock）：允许多个 goroutine 同时读。
写锁（Lock）：独占，阻塞所有读和其他写。
适用于“读多写少”的场景。

var rwmu sync.RWMutex
var cache map[string]string
// 读操作
func get(key string) string {
    rwmu.RLock()
    defer rwmu.RUnlock()
    return cache[key]
}
// 写操作
func set(key, value string) {
    rwmu.Lock()
    defer rwmu.Unlock()
    cache[key] = value
}
~~~
* 注意
~~~
写锁优先级问题：某些实现中，持续的读请求可能“饿死”写请求（Go 的 RWMutex 在写等待时会阻止新读者进入，缓解此问题）。
不要在读锁内修改共享数据！仅用于读取。
和 Mutex 一样，不可重入、不可复制。
~~~

### 3.sync.Map 并发安全的map
* 特点
~~~
内置并发安全的 map，内部使用分段锁或原子操作优化。
适用于读远多于写、key 集合不固定（频繁增删）的场景。
不是通用替代品，仅在特定场景比 map + RWMutex 更高效。

var m sync.Map

m.Store("key", "value")
if v, ok := m.Load("key"); ok {
    fmt.Println(v)
}
~~~
* 注意
~~~
接口类型：key 和 value 都是 interface{}，有类型转换开销。
不支持遍历快照：Range 回调期间 map 可能被修改。
不适合复杂操作（如“读-改-写”原子操作），此时仍需配合锁。
~~~

### 4.原子操作（sync/atomic）—— 无锁
* 特点
~~~
对简单类型（int32/int64/pointer 等）提供原子读写、CAS（Compare-And-Swap）。
性能极高，无 goroutine 阻塞。
适用于计数器、标志位等简单状态。

var counter int64
// 原子自增
atomic.AddInt64(&counter, 1)
// 原子加载
val := atomic.LoadInt64(&counter)
~~~
* 注意
~~~
仅适用于对齐的、简单内存操作。
不能用于保护复杂数据结构（如 slice、struct 字段组合）。
~~~

### 5.使用注意
* 锁粒度控制

`尽量缩小临界区，只保护真正需要同步的代码。`
* 及时释放锁资源

`建议使用defer，避免异常路径漏释放。`
* 避免锁复制,

`保证必要使用指针传递，或确保 struct 不被复制。`
~~~
type Counter struct {
    mu sync.Mutex
    n  int
}
c1 := Counter{}
c2 := c1 // 错误！Mutex 被复制，行为未定义
~~~
* 过度使用 RWMutex

`如果写操作频繁，RWMutex 可能比 Mutex 更慢（因管理 reader 队列开销）。`

``````
简单共享变量 → 优先考虑 atomic。
保护复杂状态（struct/slice/map） → 用 Mutex。
高频读、低频写（如配置缓存） → 用 RWMutex。
动态 key 的 map 且读为主 → 考虑 sync.Map。
``````