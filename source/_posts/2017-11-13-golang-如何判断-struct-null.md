---
title: golang 如何判断 struct==null?
date: 2017-11-13 10:01:22
tags:
- golang
categories:
 - 学习
 - 技术
---

Golang struct 是个非常常见的数据类型，和Java类似, struct可以用于“包容”其他类型的数据。例如，我们声明一个新的Struct Person用于表示一个人，并给它添加两个属性：身份证号(id)和姓名(name)：

<!-- more -->

```
type Person struct {
    id   int64
    name string
}
```
我们声明了一个Person类型的对象:
```
var person Person    // in Golang
Person person;       // in Java
```
如何判断它是否为空呢？在Java中我们可以用`person==null`来判断Person对象是否为空，不幸的是，Golang如果照葫芦画瓢，即`person==nil`，不出意外的会得到一个编译错误
> cannot convert nil to type Person  

nil在Golang中相当Java中的null，它表示某一个变量为空。nil只能赋值给指针、channel、func、interface、map或slice类型的变量，将nil赋值给其他类型的变量会引发panic。  
如果想要判断一个struct对象是否为未初始化的对象，按照如下方式进行：

```
package main

import "fmt"

type Person struct {
   id   int64
   name string
}

func main() {
   var foo Person
   if (Person{}) == foo {
      fmt.Println("foo is empty")
   }

   bar := Person{
      id:   110,
      name: "bar",
   }
   if (Person{}) != bar {
      fmt.Println("bar is not empty")
   }
}  

运行结果:
foo is empty
bar is not empty
```
但是，假如Person中保存了其他类型的字段，例如：
```
type Person struct {
    id   int 64
    name string
    addr []byte
}
```
假如在Person中用[]byte类型保存了另外一个字段：地址(addr)，那么按照上述方法就会GG了，不出意外，得到如下错误：
`invalid operation: Person literal == foo (struct containing []byte cannot be compared)`

其实不止上述应用场景，包含切片, map 等类型的struct也是无法相互比较的。例如，我们无法进行`foo==bar`的比较操作。这个限制在Golang的相关文档中也有体现：
> Struct values are comparable if all their fields are comparable. Two struct values are equal if their corresponding non-blank fields are equal...

所以，上述这种判断空struct对象的方法有个大前提：struct中的字段类型必须是可以“比较”的。这种情况下，有两种解决办法：

* 利用反射
```
package main

import "fmt"
import "reflect"

type Person struct {
	id   int64
	name string
	addr []byte
}

func main() {
	var foo Person
	if reflect.DeepEqual(foo, Person{}) {
		fmt.Println("foo is empty")
	}
	bar := Person{
		id:   110,
		name: "bar",
	}
	if !reflect.DeepEqual(bar, Person{}) {
		fmt.Println("bar is not empty")
	}
}

输出结果：
foo is empty
bar is not empty
```
* 额外增加一个字段，用于表示struct是否被初始化
```
package main

import "fmt"

type Person struct {
	ready bool
	id    int64
	name  string
	addr  []byte
}

func main() {
	var foo Person
	if !foo.ready {
		fmt.Println("foo is empty")
	}
	bar := Person{
		ready: true,
		id:    110,
		name:  "bar",
	}
	if bar.ready {
		fmt.Println("bar is not empty")
	}
}

运行结果：
foo is empty
bar is not empty
```
两种方法各有优劣，利用反射无需额外增加字段，但是利用发射往往意味着牺牲点性能，但是，若非经常性的操作，感觉还可以接受。额外增加字段的方法，恼人的是每次初始化struct的时候都需要记得把这个字段赋值，要不然就很尴尬了。
