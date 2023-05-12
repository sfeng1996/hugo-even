---
layout:     post
title:      "go for-range 的坑"
description: "go for-range 的坑"
date:     2021-06-21
author: sfeng
tags: ["Golang"]
URL: "/2021/06/21/go-forrange"
---

## 简介

go 语言的 for - range 作为 for 的语法糖，用起来确实舒服，但是却有不少坑在当中。

## 问题：

### 1、使用指针获取元素

如下代码想从数组遍历获取一个指针元素切片集合

```go
arr := [2]int{1, 2}
res := []*int{}
for _, v := range arr {
    res = append(res, &v)
}
for _, v := range res {
		fmt.println(*v)
}
// output: 2 2
```

结果是2 2 , 不符合预期。通过阅读[go 编译器源码](https://github.com/golang/gofrontend/blob/e387439bfd24d5e142874b8e68e7039f74c744d7/go/statements.cc#L5501)，发现会生成一个全局变量，在循环过程中，这个每次遍历的值都会赋值给这个全局变量，所以上例中使用指针获取值的地址追加到数组中，始终都是这个全局变量的地址，所以地址是一样的。这个地址遍历完最后指向了数组最后一个元素，所以打印的是数组最后一个元素。

```go
// len_temp := len(range)
// range_temp := range
// for index_temp = 0; index_temp < len_temp; index_temp++ {
       // 这个 value_temp 就是上面说的全局变量
//     value_temp = range_temp[index_temp]
//     index = index_temp
//     value = value_temp
//     original body
//   }
```

出现这种情况如何修改呢？

方法一：使用局部变量拷贝 v

```go
for _, v := range arr {
    //局部变量v替换了v，也可用别的局部变量名, 这样 &v 就不是for - range 创建的全局变量地址了
    //保证每个地址不同
    v := v 
    res = append(res, &v)
}
```

方法二：使用索引获取值

```go
//这种其实退化为for循环的简写
for k := range arr {
    res = append(res, &arr[k])
}
```

### 2、遍历过程中扩容数组

```go
v := []int{1, 2, 3}
for i := range v {
    v = append(v, i)
}
```

虽然在遍历过程中，不断给切片增加元素，但是根据go编译器源码，可以发现在遍历之前，数组长度已经固定，所以这个循环会正常结束。

```go
// 切片长度已经在遍历前固定
// len_temp := len(range)
// range_temp := range
// for index_temp = 0; index_temp < len_temp; index_temp++ {
       // 这个 value_temp 就是上面说的全局变量
//     value_temp = range_temp[index_temp]
//     index = index_temp
//     value = value_temp
//     original body
//   }
```

### 3、对大数组遍历

```go
var arr = [102400]int{1, 1, 1} 
for i, n := range arr {
   pass
}
```

对一个大数组使用 for range 遍历的话，会浪费内存资源，因为在遍历前，会拷贝一次数组。

```go
// len_temp := len(range)
// 拷贝切片
// range_temp := range
// for index_temp = 0; index_temp < len_temp; index_temp++ {
//     value_temp = range_temp[index_temp]
//     index = index_temp
//     value = value_temp
//     original body
//   }
```

可以使用遍历数组的指针，或者对数组转为切片

```go
for i, n := range &arr
for i, n := range arr[:]
```

### 4、goroutine 加 闭包

```go
var m = []int{1, 2, 3}
for i := range m {
    go func() {
        fmt.Print(i)
    }()
}
//block main 1ms to wait goroutine finished
time.Sleep(time.Millisecond)
```

预期输出0，1，2的某个组合，如012，210.. 结果是222. 也是拷贝的问题，因为闭包 = 函数 + 环境

这里闭包函数引用了 i，是指针

使用参数传入闭包内，这样 i 进行了值拷贝

```go
for i := range m {
    go func(i int) {
        fmt.Print(i)
    }(i)
}
```

使用局部变量拷贝

```go
for i := range m {
    i := i
    go func() {
        fmt.Print(i)
    }()
}
```