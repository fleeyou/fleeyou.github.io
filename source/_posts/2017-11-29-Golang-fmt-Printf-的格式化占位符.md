---
title: Golang fmt.Printf 的格式化占位符
date: 2017-11-29 09:30:03
tags:
- Golang
categories:
- 技术
- 学习
copyright: true
---
&emsp; &emsp;今天看Golang fmt[官方文档][1]，看到了占位符的相关内容，记录下。  

<!-- more -->

&emsp; &emsp; Golang的占位符与C语言的占位符很类似，但是要精简的多。下面按照数据类型作为分类标准，介绍下各个占位符的使用。  

-  通用占位符  
   &emsp; &emsp;Golang使用“%v”作为一个“万能”的占位符，编译器可以根据数据类型自动推断最终的数据展示格式。但是，这个通用占位符还是有定制的空间的。

   ```golang
   # 示例数据
   type animal struct {
     name string
     age  int
   }
   cat := animal{name: "kitty", age: 3}
   ```

|占位符|说明                    |示例                     |输出                            |
|:--:  |:--:                    |:--:                     |:--:                            |
|%v    |输出默认格式            |`fmt.Printf("%v", cat)`  |{kitty 3}                       |
|%+v   |为struct类型添加字段名称|`fmt.Printf("%+v", cat)` |{name:kitty age:3}              |
|%#v   |输出完整语法表示        |`fmt.Printf("%#v", cat)` |main.animal{name:"kitty", age:3}|
|%T    |打印数据类型            |`fmt.Printf("%T", cat)`  |main.animal                     |
|%%    |打印百分号%             |`fmt.Printf("%%")`       |%                               |

   &emsp;&emsp;除了上述通用的占位符，针对各个不同的数据类型，Golang有相应的专用占位符。

-  布尔型  
   &emsp;&emsp;针对布尔型的数据，如果想要打印"true"或"false", 可以使用%t作为占位符。  

|占位符|说明           |示例                   |输出|
|:--:  |:--:           |:--:                   |:--:|
|%t    |输出true或false|`fmt.Printf("%t",ture)`|true|

-  整型
 
|占位符|说明                      |示例                      |输出    |
|:--:  |:--:                      |:--:                      |:--:    |
|%b    |二进制表示                |`fmt.Printf("%b", 233)`   |11101001|
|%o    |八进制表示                |`fmt.Printf("%o", 233)`   |351     |
|%d    |十进制表示                |`fmt.Printf("%d", 0351)`  |233     |
|%x    |十六进制表示(小写字母)    |`fmt.Printf("%x", 233)`   |e9      |
|%X    |十六进制表示(大写字母)    |`fmt.Printf("%X", 233)`   |E9      |
|%c    |Unicode码点对于的字符     |`fmt.Printf("%c", 0x75D2)`|痒      |
|%U    |输出Unicode,等价于"U+%04X"|`fmt.Printf("%U", '痒')`  |0x75D2  |

-  浮点型和复数

|占位符|说明                                  |示例                       |输出                |
|:--: |:--:                                   |:--:                       |:--:                |
|%b   |无小数部分，指数基数为二的科学计数法   |`fmt.Printf("%b", 65536.0)`|4503606499318170p-36|
|%e   |科学计数法                             |`fmt.Printf("%e", 65536.0)`|6.553610e+04        |
|%E   |科学计数法                             |`fmt.Printf("%e", 65536.0)`|6.553610E+04        |
|%f   |没有指数的小数                         |`fmt.Printf("%f", 65536.1)`|65536.1             |
|%F   |没有指数的小数                         |`fmt.Printf("%F", 65536.1)`|65536.1             |
|%g   |以更紧凑的方式自动选择%e或%f(末尾无0)  |`fmt.Printf("%g", 65536.1)`|65536.1             |
|%G   |以更紧凑的方式自动选择%E或%f(末尾无0)  |`fmt.Printf("%G", 65536.1)`|65536.1             |
|%2.1g|输出结果宽度2(左侧补空格),结果共1位数字|`fmt.Printf("%2.1g", 1.21)`| 1(左侧有一空格)    |
|%2.1f|输出结果共2位(左侧补空格),小数1位      |`fmt.Printf("%2.1f", 1.21)`|1.2                 |

-  字符串及字节切片

|占位符|说明                              |示例                              |输出                    |
|:--:  |:--:                              |:--:                              |:--:                    |
|%s    |字符串表示                        |`fmt.Printf("%s", "hello, world")`|hello, world            |
|%q    |带双引号的字符串表示              |`fmt.Printf("%q", "hello, world")`|"hello, world"          |
|%x    |对字节序列的十六进制表示(小写字母)|`fmt.Printf("%x", "hello, world")`|68656c6c6f2c20776f726c64|
|%X    |对字节序列的十六进制表示(大写字母)|`fmt.Printf("X%", "hello, world")`|68656C6C6F2C20776F726C64|

-  指针

|占位符|说明                      |示例                    |输出        |
|:--:  |:--:                      |:--:                    |:--:        |
|%p    |十六进制表示指针值, 0x打头|`fmt.Printf("%p", &var)`|0xc4200160c8|

-  再探通用占位符
   对于，各数据类型，通用占位符对应的特定占位符为：

|数据类型                    |对应的特定占位符             |
|:---                        |:--:                         |
|bool                        |%t                           |
|int, int8等                 |%d                           |
|uint, uint8等               |%d 或%#v(%#v)                |
|float32, complex64等        |%g                           |
|string                      |%s                           |
|chan                        |%p                           |
|指针                        |%p                           |
|struct                      |{field0 field1...}           |
|array, slice                |[elem0 elem1...]             |
|maps                        |map[key1:valule1 key2:value2]|
|struct,array,slice,maps 指针|&{}, &[], &map[]             |

-  占位符辅助标识
   * +: 打印数值正负号  
   * -: 在右侧填充空格  
   * \#: 为八进制/十六进制添加前缀; %#p将去掉0x前缀;%#q将打印原始字符串(若有单引号则输出)  
   * 0: 用0而非空格填充  
   * 空格: % d将用空格代替原本的正负号; % x或% X打印字符串或字节切片时，字节直接将以空格分隔  

[1]:	https://golang.org/pkg/fmt/
