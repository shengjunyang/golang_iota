原文连接：https://blog.wolfogre.com/posts/golang-iota/
一
先看一段代码吧：

const (
    a = iota
    b
    c
)
相信你能脱口答出来，常量 a 等于 0，此后定义的常量依次递增，b = 1，c = 2。没毛病，这有何难？

是的，我此前也和你一样，觉得自己已经搞懂了 golang 里的 iota 的用法了，即使它有什么鲜有人知鬼畜的神奇花招，我也没兴趣细究，毕竟我并不喜欢研究奇技淫巧。

直到不久前的一天，我在一次代码研读会上，看到了这样一段代码：

const (
	mutexLocked = 1 << iota
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota
)
这可不是什么野狐禅，它摘自 golang 标准库里 sync.Mutex 的实现，你可以在 golang 源码里看到它。先不用管这些常量是做什么用的，只回答一个简单的问题，这些常量的值是多少？如果你能快读回答出来，那这篇文章对你来说应该没有什么阅读价值了，不过如果你想继续挑战一下自己的话，不妨再看看这段代码：

const a = iota

const (
	b = iota
)

const (
	c = 10
	d = iota
	e
	f = "hello"
	// nothing
	g
	h = iota
	i
	j = 0
	k
	l, m = iota, iota
	n, o

	p = iota + 1
	q
	_
	r = iota * iota
	s
	t = r
	u
	v = 1 << iota
	w
	x = iota * 0.01
	y float32 = iota * 0.01
	z
)
还是那个问题，这些常量的值是多少？

没错，这确实是我故意写出来刁难人的，不过不是刁难你，而是刁难我自己，如标题所言，我试图彻底搞懂 golang 的 iota 的用法，不管代码如何变态，我至少能看懂常量的值是多少，这样当我转过身去揍写出这样类似代码的童鞋的时候，心里还能有点底气。

那就彻底搞懂它吧。

二
要理清楚这些常量，我们需要一些步骤，这些步骤没什么技巧可言，纯粹是类似口诀的东西，记下来即可。

第一步：不同 const 定义块互不干扰
这一点好理解，就是我们在推算这些常量值的时候只需要看当前的 const ( ) 之内的内容，不用关心之前或之后是否也用常量定义语句。

上文中那段代码的开头部分：

const a = iota

const (
	b = iota
)
由于 a 和 b 在不同的定义块里，互不影响，所以 a 等于 0 且 b 也等于 0，也不会影响后面的常量定义。所以下面我们重点看后面的常量 c 到 z 。

第二步：所有注释行和空行全部忽略
没错，你应该注意到我在代码里安插了一行毫无意义的注释和一行莫名其妙的空行，这是我故意为之，但不用多想，这完全不会影响常量的定义，直接忽略即可。

但需要注意的是，代码 _ 并不是一个空行，它是一个省略了标识符也省略了表达式的常量定义，这一点你需要清楚，不要大意。

所以现在你脑中的代码应该是这样：

const (
	c = 10
	d = iota
	e
	f = "hello"
	g
	h = iota
	i
	j = 0
	k
	l, m = iota, iota
	n, o
	p = iota + 1
	q
	_
	r = iota * iota
	s
	t = r
	u
	v = 1 << iota
	w
	x = iota * 0.01
	y float32 = iota * 0.01
	z
)
第三步：没有表达式的常量定义复用上一行的表达式
这一步比较关键，golang 在常量定义时是可以省略表达式的，编译时会自动复用上一行的表示式。你问如果上一行也省略了表达式怎么办，继续往上找嘛，由此可见，一个常量定义代码块的第一行定义是不可以省略，否则就不明所以了。

要注意这个特性跟 iota 是没有关系的，即使定义时没有用到 iota，这个特性也仍然有效。

到这里，思路就开始清晰了：

const (
	c = 10
	d = iota
	e = iota
	f = "hello"
	g = "hello"
	h = iota
	i = iota
	j = 0
	k = 0
	l, m = iota, iota
	n, o = iota, iota
	p = iota + 1
	q = iota + 1
	_ = iota + 1
	r = iota * iota
	s = iota * iota
	t = r
	u = r
	v = 1 << iota
	w = 1 << iota
	x = iota * 0.01
	y float32 = iota * 0.01
	z float32 = iota * 0.01
)
第四步：从第一行开始，iota 从 0 逐行加一
这是一个比较容易混淆人的点，就是赋值表达式里无论是否引用了 iota，也无论引用了多少次，iota 的都会从常量定义块的第一行（注意这里不计空行和注释）开始计数，从 0 开始，逐行加一。

所以在这一步里我们先不用管常量定义的表达式是什么，先把 iota 在当前行的位置的值先写出来，这有助于防止被混淆视听。

形如：

const (
	c = 10 // iota = 0
	d = iota // iota = 1
	e = iota // iota = 2
	f = "hello" // iota = 3
	g = "hello" // iota = 4
	h = iota // iota = 5
	i = iota // iota = 6
	j = 0 // iota = 7
	k = 0 // iota = 8
	l, m = iota, iota // iota = 9
	n, o = iota, iota // iota = 10
	p = iota + 1 // iota = 11
	q = iota + 1 // iota = 12
	_ = iota + 1 // iota = 13
	r = iota * iota // iota = 14
	s = iota * iota // iota = 15
	t = r // iota = 16
	u = r // iota = 17
	v = 1 << iota // iota = 18
	w = 1 << iota // iota = 19
	x = iota * 0.01 // iota = 20
	y float32 = iota * 0.01 // iota = 21
	z float32 = iota * 0.01 // iota = 22
)
第五步：替换所有 iota
最后一步就比较无脑了，逐行替换出现的 iota 为真实值即可：

const (
	c = 10 // iota = 0
	d = 1 // iota = 1
	e = 2 // iota = 2
	f = "hello" // iota = 3
	g = "hello" // iota = 4
	h = 5 // iota = 5
	i = 6 // iota = 6
	j = 0 // iota = 7
	k = 0 // iota = 8
	l, m = 9, 9 // iota = 9
	n, o = 10, 10 // iota = 10
	p = 11 + 1 // iota = 11
	q = 12 + 1 // iota = 12
	_ = 13 + 1 // iota = 13
	r = 14 * 14 // iota = 14
	s = 15 * 15 // iota = 15
	t = r // iota = 16
	u = r // iota = 17
	v = 1 << 18 // iota = 18
	w = 1 << 19 // iota = 19
	x = 20 * 0.01 // iota = 20
	y float32 = 21 * 0.01 // iota = 21
	z float32 = 22 * 0.01 // iota = 22
)
到这里，事情已经水落石出。

无它。

三
总结一下：

不同 const 定义块互不干扰；
所有注释行和空行全部忽略；
没有表达式的常量定义复用上一行的表达式；
从第一行开始，iota 从 0 逐行加一；
替换所有 iota。
如何你想记住些什么，记住这 5 条即可，足够帮你处理各种鬼畜情况了。

四
行文至此，本该结束了。但我还想多说一句：“劝你善良。”

我相信你或多或少开始动歪心思了：“这段鬼畜的代码，可以做成一道面试题了，来考考哪些自称‘精通 golang 开发’的求职者，打击打击他们的自信心。”

不不不，我劝你不要这样做，你提出这样一道不负责任的面试题，即是对求职者的不尊重，也会让你的公司错失优秀的人才，我有理由相信一位优秀的 golang 开发工程师完全不清楚本文所述的内容，但他一样会拥有高超的技术水平和优秀的工程能力，但你用这样一道尖酸刻薄的面试题否定他，当他被拒之后，在回家的路上会满腹牢骚，也会满腹狐疑，不自主得拿出手机搜问题答案，便很有可能寻至此处，定睛一看，对我破口大骂。那我就得很负责任地说一句：“此事与我无关。”

确实与我无关，我只是偶然发现了一个可能无关痛痒问题，分享了出来，避免大家踩坑，从未有过“你不知道这个就说明你不够厉害”的龌龊想法。在我看来，有些（注意，只是说有一些）被人奉为圭臬的“知识点”，都是人造产品，甚至是人造副产品，不知道它们也不见得是一件坏事。

附
参考文章：

golang wiki - Iota
iota: Golang 中优雅的常量
The Go Programming Language Specification - Iota
本次实验的代码全文：

package main

import (
	"fmt"
)

const a = iota

const (
	b = iota
)

const (
	c = 0
	d = iota
	e
	f = "hello"
	// nothing
	g
	h = iota
	i
	j = 0
	k
	l, m = iota, iota
	n, o

	p = iota + 1
	q
	_
	r = iota * iota
	s
	t = r
	u
	v = 1 << iota
	w
	x = iota * 0.01
	y float32 = iota * 0.01
	z
)

func main() {
	fmt.Printf("a : %T = %v\n", a, a)
	fmt.Printf("b : %T = %v\n", b, b)
	fmt.Printf("c : %T = %v\n", c, c)
	fmt.Printf("d : %T = %v\n", d, d)
	fmt.Printf("e : %T = %v\n", e, e)
	fmt.Printf("f : %T = %v\n", f, f)
	fmt.Printf("g : %T = %v\n", g, g)
	fmt.Printf("h : %T = %v\n", h, h)
	fmt.Printf("i : %T = %v\n", i, i)
	fmt.Printf("j : %T = %v\n", j, j)
	fmt.Printf("k : %T = %v\n", k, k)
	fmt.Printf("l : %T = %v\n", l, l)
	fmt.Printf("m : %T = %v\n", m, m)
	fmt.Printf("n : %T = %v\n", n, n)
	fmt.Printf("o : %T = %v\n", o, o)
	fmt.Printf("p : %T = %v\n", p, p)
	fmt.Printf("q : %T = %v\n", q, q)
	fmt.Printf("r : %T = %v\n", r, r)
	fmt.Printf("s : %T = %v\n", s, s)
	fmt.Printf("t : %T = %v\n", t, t)
	fmt.Printf("u : %T = %v\n", u, u)
	fmt.Printf("v : %T = %v\n", v, v)
	fmt.Printf("w : %T = %v\n", w, w)
	fmt.Printf("x : %T = %v\n", x, x)
	fmt.Printf("y : %T = %v\n", y, y)
	fmt.Printf("z : %T = %v\n", z, z)
}
