---
title: "SQL_Index 索引"
date: 2025-10-20
categories: ["SQL"]
tags: ["命中", "失效"]
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