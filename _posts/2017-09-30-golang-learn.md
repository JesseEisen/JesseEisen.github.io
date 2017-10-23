---
layout: post
title: Go语言初学
date: 2017-09-30 05:57:00
description: golang learn and think
---

很早就听说了Go这门语言，大概的接触了下。也就写了一些很浅显的代码，并没有太多深入的尝试。最近
购了一本《GO程序设计语言》影印版。 我的偶像 Kerninghan出品，虽然是英文的，但是看着并没有特别的不适

练过go tour后，对一些点还是觉得不太清晰。虽然能用go做了测试题，但是总觉得不踏实。所以在看这本书的时候，了解了很多细节。很多东西都有了更加清晰的认识。

<hr>
上面说了一些废话，本文主要是想谈下interface的使用。这个对我而言算是比较新的一个概念，因为我一直写的C／shell这些
。所以一些高级语言的概念了解的不太多。所以在理解上有一些阻碍。但是细细思考后，觉得interface很类似于C的void \*，
但是我个人觉得interface比void \* 更加灵活。

很多地方都提到了duck模型，说白了就是只要实现了所有interface定义的方法就表示这个类型实现了这个interface。
即函数的参数是这个interface的类型的话，任何实现了这个interface的类型都可以传入。一段代码来

```
type I interface {
  M() string
}

type T struct {
  name string
}

func (t T) M() string {
  return t.name
}

func Hello(i I) {
  fmt.Printf("My name is %s", i.M())
}

func main() {
  Hello(T{name: "Stephen"})
}
```

## 简单示例

<script src="https://gist.github.com/JesseEisen/aec959e6ccf7da486f45fbd19b2bc9e2.js"></script>
