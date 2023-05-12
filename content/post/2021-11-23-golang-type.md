---
layout:     post
title:      "golang类型系统"
description: "本文讲述golang类型系统及使用"
except: "本文讲述golang类型系统及使用"
date:     2021-11-23
author: "sfeng"
tags: ["Golang"]
URL: "/2021/11/23/golang-type/"
---

## 命名类型和未命名类型

### 命名类型

类型可以通过标识符，这种类型称为命名类型（ `Named Type`）。 `Go`语言的基本类型中有 20个预声明简单类型都是命名类型， `Go` 语言还有一种命名类型一一用户自定义类型，使用 `type` 字段定义的类型都是命名类型。在`go` 语言中，20个内置类型都是`type`字段定义的，以及在平时开发使用 `type` 的自定义类型。

### 未命名类型

一个类型由预声明类型、关键字和操作符组合而成，这个类型称为未命名类型（ `Unamed Type` ）。未命名类型又称为类型字面量（ `Type Literal` ）。Go 语言的基本类型中的复合类型：数组（ `array` ）、切片（ `slice` ）、字典（ `map` ）、通道（ `channel` ）、指针（ `pointer` ） 、函数字面量（ `function` ）、结构（ `struct` ）和接口（ `interface` ）都属于类型字面量，也都是未命名类型。所以 *int , []int , [2]int , map[k]v 都是未命名类型。

注意：前面所说的结构和接口是未命名类型，这里的结构和接口没有使用 `type` 格式定义，具体见下方示例说明。

```go
package main

import "fmt"

// Person 使用 type 声明的是命名类型
type Person struct {
	name string
	age  int
}

func main() {
	// 使用 struct 字面量声明的是未命名类型
	a := struct {
		name string
		age  int
	}{"Jim", 20}

	fmt.Printf("%T\n", a) // struct { name string; age int }
	fmt.Printf("%v\n", a) // {Jim 20}

	b := Person{"Tom", 22}
	fmt.Printf("%T\n", b) // main.Person
	fmt.Printf("%v\n", b) // {Tom 22}

}
```

`Go`语言的命名类型和未命名类型总结如下：

![Untitled](/img/golang-type.png)

![Untitled](/img/golang-identifilers.png)

未命名类型和类型字面量是等价的，我们通常所说的 Go 语言基本类型中的复合类型就是类型字面量，所以未命名类型、类型字面量和 Go 语言基本类型中的复合类型三者等价。
通常所说的 Go 语言基本类型中的简单类型就是这 20 个预声明类型，它们都属于命名类型。
预声明类型是命名类型的一种，另一类命名类型是自定义类型。

## 底层类型

所有“类型”都有一个 `underlying type` （底层类型）。底层类型的规则如下：

- 简单类型和复合类型的底层类型是它们自身。
- 自定义类型 type newtype oldtype 中 newtype 的底层类型是逐层递归向下查找的，直到查到的 oldtype 是简单类型或复合类型为止。

例如：

```go
type T1 string
type T2 Tl
type T3 []string
type T4 T3
type T5 []T1
type T6 T5
```

`T1` 和 `T2` 的底层类型都是 `string` ， `T3`和 `T4` 的底层类型都是 `[]string` ， `T5` 和 `T6` 的底层类型都是 `[]T1` 。特别注意这里的 `T6` 、 `T5` 与 `T3`、 `T4` 的底层类型是不一样的， 一个是 `[]T1`
 ，另一个是 `[]string` 。

## 类型相同

`Go` 是强类型的语言，编译器在编译时会进行严格的类型校验。两个命名类型是否相同，参考如下：

- 两个命名类型相同的条件是两个类型声明的语句完全相同；
- 命名类型和未命名类型永远不相同；
- 两个未命名类型相同的条件是它们的类型声明字面量的结构相同，井且内部元素的类型相同；
- 通过类型别名语句声明的两个类型相同；

Go 1.9 引入了类型别名语法 `type T1 = T2` , T1 的类型完全和 T2 一样。

## 类型赋值

不同类型的变量之间一般是不能直接相互赋值的，除非满足一定的条件。类型为 `T1` 的变量 `a`可以赋值给类型为 `T2`的变量 `b` ， 称为类型 `T1` 可以赋值给类型 `T2` ，伪代码表述如下：

```go
// a 是类型为T1 的变量，或者a 本身就是一个字面常量或 nil
// 如果如下语句可以执行，则称之为类型 Tl 可以赋值给类型T2
var b T2 = a
```

a 可以赋值给变量 b 必须要满足如下条件中的一个：

- T1 和 T2 类型相同；
- T1 和 T2 具有相同的底层类型，并且 T1 和 T2 里面至少有一个是未命名类型；
- T2 是接口类型， T1 是具体类型， T1 的方法集是 T2 方法集的超集；
- T1 和 T2 都是通道类型，它们拥有相同的元素类型，并且 T1 和 T2 中至少有一个是未命名类型；
- T1是预声明标识符 nil ， T2 是 pointer 、 funcition 、 slice 、 map 、 channel 、 interface 类型中的一个；
- a 是一个字面常量值，可以用来表示类型 T 的值；

```go
package main

import "fmt"

type Map map[string]string

func (m Map) Print() {
	for _, v := range m {
		fmt.Println("v is ", v)
	}
}

type iMap Map

// 只要底层类型是slice 、map 等支持range 的类型字面量，新类型仍然可以使用range 迭代
func (m iMap) Print() {
	for _, v := range m {
		fmt.Println("v is ", v)
	}
}

type slice []int

func (s slice) Print() {
	for _, v := range s {
		fmt.Println("v is ", v)
	}
}

func main() {
	mp := make(map[string]string, 10)
	mp["hi"] = "hello"

	// mp 与ma 有相同的底层类型map[string]stirng ，并且mp 是未命名类型
	// 所以mp 可以直接赋值给ma
	var ma Map = mp

	/*
		im 与 ma 虽然有相同的底层类型map[string]stirng，但它们中没有一个是未命名类型
		不能赋值， 如下语句不能通过编译
	*/
	// var im iMap = ma

	ma.Print()
	// im.Print()

	var i interface {
		Print()
	} = ma

	i.Print()
	s1 := []int{1, 2, 3}
	var s2 slice
	s2 = s1
	s2.Print()
}

```

## 类型强制转换

由于Go是强类型的语言， 如果不满足自动转换的条件，则必须进行强制类型转换。任意两个不相干的类型如果进行强制转换，则必须符合一定的规则。强制类型的语法格式：

```go
var a T = (T) (b)
```

使用括号将类型和要转换的变量或表达式的值括起来。

非常量类型的变量 `x` 可以强制转化并传递给类型 `T` ， 需要满足如下任一条件：

- `x` 可以直接赋值给 `T` 类型变量；
- `x` 的类型和 `T` 具有相同的底层类型；

继续上一节使用的示例：

```go
/*
		im 与ma 虽然有相同的底层类型，但是二者中没有一个非命名类型，不能直接赋值，可以
		强制进行类型转换
	*/
	var im iMap = (iMap)(ma)
```

- x 的类型和 T 都是未命名的指针类型，并且指针指向的类型具有相同的底层类型；
- x 的类型和 T 都是整型，或者都是浮点型；
- x 的类型和 T 都是复数类型；
- x 是整数值或 []byte 类型的值， T 是 string 类型；
- x 是一个字符串， T 是 []byte 或 []rune ；

字符串和字节切片之间的转换最常见，示例如下：

```go
func main() {
	s := "hello,你好"
	var a []byte
	a = []byte(s)

	var b string
	b = string(a)

	var c []rune
	c = []rune(s)

	fmt.Printf("%T\n", a) // []uint8 	byte 是 int8 的别名
	fmt.Printf("%T\n", b) // string
	fmt.Printf("%T\n", c) // []int32 	rune 是 int32 的别名
}
```

注意：

- 数值类型和 string 类型之间的相互转换可能造成值部分丢失； 其他的转换仅是类型的转换，不会造成值的改变。
- string 和数字之间的转换可使用标准库 strconv 。
- Go 语言没有语言机制支持指针和 interger 之间的直接转换，可以使用标准库中的 unsafe 包进行处理。

## 类型别名与新声明类型

使用类型别名需要在别名和原类型之间加上赋值符号（=）；使用类型别名定义的类型与原类型等价，而使用类型定义出来的类型是一种新的类型。

```go
package main

import (
    "fmt"
)

type a = string
type b string

func SayA(str a) {
    fmt.Println(str)
}

func SayB(str b) {
    fmt.Println(str)
}

func main() {
    var str = "test"
    SayA(str)

    //错误参数传递，str是字符串类型，不能赋值给b类型变量
    SayB(str)
}

```

这段代码在编译时会出现如下错误：

```go
cannot use str (type string) as type b in argument to SayB
```

从错误信息可知，`str` 为字符串类型，不能当做 `b` 类型参数传入 `SayB` 函数中。而 `str`却可以当做 `a` 类型参数传入到 `SayA` 函数中。由此可见，使用类型别名定义的类型与原类型一致，而类型定义定义出来的类型，是一种新的类型。

### 类型别名

示例：

```go
package main

import "fmt"

func main() {
	{
		type MyString = string
		str := "BCD"
		myStr1 := MyString(str)
		myStr2 := MyString("A" + str)
		fmt.Printf("%T(%q) == %T(%q): %v\n", str, str, myStr1, myStr1, str == myStr1)
		fmt.Printf("%T(%q) > %T(%q): %v\n", str, str, myStr2, myStr2, str > myStr2)
		fmt.Printf("Type %T is the same as type %T.\n", myStr1, str)
		fmt.Println()
		strs := []string{"E", "F", "G"}
		myStrs := []MyString(strs)
		fmt.Printf("A value of type []MyString: %T(%q)\n", myStrs, myStrs)
		fmt.Printf("Type %T is the same as type %T.\n", myStrs, strs)
		fmt.Println()
	}
}
```

输出结果：

```go
string("BCD") == string("BCD"): true
string("BCD") > string("ABCD"): true
Type string is the same as type string.

A value of type []MyString: []string(["E" "F" "G"])
Type []string is the same as type []string.

```

### 新类型声明

示例：

```go
package main

import "fmt"

func main() {
	{
		type MyString string
		str := "BCD"
		myStr1 := MyString(str)
		myStr2 := MyString("A" + str)
		_ = myStr2

		// 这里的判等不合法，会引发编译错误。
		// invalid operation: str == myStr1 (mismatched types string and MyString)
		// fmt.Printf("%T(%q) == %T(%q): %v\n", str, str, myStr1, myStr1, str == myStr1)

		// 这里的比较不合法，会引发编译错误。
		// fmt.Printf("%T(%q) > %T(%q): %v\n", str, str, myStr2, myStr2, str > myStr2)
		fmt.Printf("Type %T is different from type %T.\n", myStr1, str)

		strs := []string{"E", "F", "G"}
		var myStrs []MyString
		// 这里的类型转换不合法，会引发编译错误。
		// cannot convert strs (type []string) to type []MyString
		// myStrs = []MyString(strs)
		//fmt.Printf("A value of type []MyString: %T(%q)\n", myStrs, myStrs)
		fmt.Printf("Type %T is different from type %T.\n", myStrs, strs)
		fmt.Println()
	}

}

```

### ****类型别名和新声明类型相互赋值****

```go
package main

func main() {
	{
		type MyString1 = string
		type MyString2 string
		str := "BCD"
		myStr1 := MyString1(str)
		myStr2 := MyString2(str)
		myStr1 = MyString1(myStr2)
		myStr2 = MyString2(myStr1)

		myStr1 = str
		// 这里的赋值不合法，会引发编译错误。
		// cannot use str (type string) as type MyString2 in assignment
		// myStr2 = str

		//myStr1 = myStr2 // 这里的赋值不合法，会引发编译错误。
		//myStr2 = myStr1 // 这里的赋值不合法，会引发编译错误。
	}
}
```

### ****给类型别名新增方法，会添加到原类型方法集中****

给类型别名新增方法后，原类型也能使用这个方法。下边请看一段示例代码：

```go
package main

import (
    "fmt"
)

// 根据string类型，定义类型S
type S string
func (r *S) Hi() {
    fmt.Println("S hi")
}

// 定义S的类型别名为T
type T = S
func (r *T) Hello() {
    fmt.Println("T hello")
}

// 函数参数接收S类型的指针变量
func exec(obj *S) {
    obj.Hello()
    obj.Hi()
}

func main() {
    t := new(T)
    s := new(S)
    exec(s)
    // 将T类型指针变量传递给S类型指针变量
    exec(t)
}
```

输出结果：

```go
T hello
S hi
T hello
S hi
```

上边的示例中，S 是原类型，T 是 S 类型别名。在给 T 增加了 Hello 方法后，S 类型的变量也可以使用 Hello 方法。说明给类型别名新增方法后，原类型也能使用这个方法。从示例中可知，变量 t 可以赋值给 S 类型变量 s，所以类型别名是给原类型取了一个小名，本质上没有发生任何变化。

类型别名，只能对同一个包中的自定义类型产生作用。举个例子，Golang SDK 中有很多个包，是不是我们可以使用类型别名，给 SDK 包中的结构体类型新增方法呢？答案是：不行。请牢记一点：类型别名，只能对包内的类型产生作用，对包外的类型采用类型别名，在编译时将会提示如下信息：