---
title: Tips in Golang
date: 2017-11-15 17:16:37
tags:
  - Golang
categories:
  - 技术
  - 学习
---

&emsp;&emsp;新人Golang入门，记录下平时遇到的小姿势，供以后回顾。

<!-- more -->

1. slice & append  
	\`\`\`
	var foo, bar []T  
	var elem T
	\`\`\`
	* 将elem添加到foo中： `foo=append(foo, elem)`
	* 将bar添加到foo中： `foo=append(foo,bar...)`
	* 将bar复制到一个名为a的slice中：`a:=make([]T,len(bar));copy(a,bar)`
	* 从bar中删除下标为i的元素（假定不会越界）：`bar=append(bar[:i],bar[i+1]...)`
	* 从bar中删除下标i到j的元素（假定不会越界）：`bar=append(bar[:i],bar[j+1]...)`
	* 将bar扩展长度为j的容量：`bar=append(bar,make([]T,j)...)`
	* 在bar的下标为i处插入长度为j的slice：`bar=append(bar[:i],append(make([]T,j),a[i:]...)...)`
	* 在bar的下标为i处插入foo：`bar=append(a[:i],append(b,a[i:]...)...)`
	* 返回并删除bar中下标最高的元素：`x,foo=foo[len(foo)-1],a[:len(foo)-1]`
