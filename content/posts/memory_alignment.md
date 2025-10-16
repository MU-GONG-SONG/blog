---
title: "内存对齐（memory alignment）"
date: 2025-10-14
categories: ["内存"]
tags: ["memory"]
draft: false
---


> 看起来很简单，但它其实是 为了内存对齐而存在的。


`````
type itab struct {
    inter *interfacetype // 接口类型信息
    _type *_type         // 实现接口的具体类型信息
    hash  uint32         // 类型 hash 值
    _     [4]byte
    fun   [1]uintptr     // 实现接口方法的函数地址
}
`````
## 一、Go 的结构体内存布局规则
Go 里每个字段在内存中都有一个偏移量（offset），而编译器会自动插入 padding（填充字节），以保证每个字段都按其类型对齐（alignment）。

规则大致是：
每个字段的起始地址必须是该字段类型的对齐倍数。
比如：uint32 对齐要求 4 字节，uintptr（在 64 位机上）对齐要求 8 字节。
整个结构体的大小必须是其内部最大对齐单位的整数倍。
编译器自动插入 padding 字节，但有时源码里会显式加 _ [N]byte 来占位或兼容 ABI。

二、itab 的字段分析（以 64 位架构为例）
我们来计算每个字段的内存偏移：


| 左对齐 | 居中 | 右对齐 |
|:-------|:----:|-------:|
| 左     | 中   | 右     |
