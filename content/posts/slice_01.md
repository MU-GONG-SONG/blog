---
title: "slice 和 数组"
date: 2025-09-02
categories: ["slice"]
tags: ["数组", "切片"]
draft: false
---

>Go 语言里 slice 和 map 是非常有用的两个内置数据结构，
线上的工程代码几乎不可能绕开它们。

### 1.数组与切片

因为切片（slice）比数组更好用，也更安全，Go 推荐使用 slice 而不是数组。本节内容比较
了 slice 和数组的区别，也研究了 slice 的一些特有的性质。
#### 1.1数组和切片有何异同

Go 语言中的切片（slice）结构的本质是对数组的封装，它描述一个数组的片段。无论是数组
还是切片，都可以通过下标来访问单个元素。
数组是定长的，长度定义好之后，不能再更改。在 Go 语言中，数组是不常见的，因为其长度
是类型的一部分，限制了它的表达能力，比如 [3]int 和 [4]int 就是不同的类型。而切片则非常灵
活，它可以动态地扩容，且切片的类型和长度无关。

    func main() { 
        arr1 := [1]int{1}
        arr2 := [2]int{1, 2}
        if arr1 == arr2 {
        fmt.Println("equal type")
        }
    }

尝试运行，报编译错误：

    ./test.go:16:10: invalid operation: arr1 == arr2 (mismatched types [1]int and [2]int)

因为两个数组的长度不同，根本就不是同一类型，因此不能进行比较。
数组是一片连续的内存，切片实际上是一个结构体，包含三个字段：长度、容量、底层数组。

    // src/runtime/slice.go
        type slice struct {
        array unsafe.Pointer // 元素指针
        len int // 长度
        cap int // 容量
    }

注意，底层数组可以被多个切片同时指向，因此对一个切
片的元素进行操作有可能会影响到其他切片。


#### 1.2切片如何被截取
截取也是一种比较常见的创建 slice 的方法，可以从数组或者 slice 直接截取，需要指定起、
止索引位置。
基于已有 slice 创建新 slice 对象，被称为 reslice。新 slice 和老 slice 共用底层数组，新老
slice 对底层数组的更改都会影响到彼此。基于数组创建的新 slice 也是同样的效果：对数组或
slice 元素做的更改都会影响到彼此。
值得注意的是，新老 slice 或者新 slice 老数组互相影响的前提是两者共用底层数组，如果因
为执行 append 操作使得新 slice 或老 slice 底层数组扩容，移动到了新的位置，两者就不会相互影
响了。所以，问题的关键在于两者是否会共用底层数组。
    
    data := [...]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
    slice := data[2:4:6] // data[low, high, max

对 data 使用 3 个索引值，截取出新的 slice。这里 data 可以是数组或者 slice。low 是最低索引
值，这里是闭区间，也就是说第一个元素是 data 位于 low 索引处的元素；而 high 和 max 则是开区
间，表示最后一个元素只能是索引 high-1 处的元素，而最大容量则只能是索引 max-1 处的元素。
要求：max >= high >= low
当 high == low 时，新 slice 为空。