> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/379566824)

> Copy elision 在 [[C++]]11 以前是省略 copy constructor，但是 C++11 之后，更多地是指省略 move constructor（只有 copy constructor 而没有 move constructor 的时候就是省略 copy constructor)。C++ 17 与 11 省略的方式还不一样，因为 value categories 有了调整。更多内容请看本文分析。  
> 如果希望复习一下 copy constructor 和 move constructor，可以阅读 [C++：Rule of five/zero 以及 Rust：Rule of two](https://zhuanlan.zhihu.com/p/369349887) 和[什么是 move？理解 C++ Value categories 和 Rust 的 move](https://zhuanlan.zhihu.com/p/374392832)

文章经过修改整理后，发于公众号 [Copy/move elision: C++ 17 vs C++ 11](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s%3F__biz%3DMzI4ODE4NTkxMw%3D%3D%26mid%3D2649441370%26idx%3D1%26sn%3Db8f4d13501cb1a4bf376c72779f716fb%26chksm%3Df3ddaf0cc4aa261a76a50b15745573e148962b17444d99ff1a7162534aefcf0ce67c547ac4fd%26token%3D2061556422%26lang%3Dzh_CN%23rd)

什么是 copy elision?
-----------------

Copy elision 是指编译器为了优化，将不需要的 copy/move 操作（含析构函数，为了行文简洁，本文忽略析构函数的省略）直接去掉了。要理解 copy elision，让我们先看看函数是如何返回值的。比如下面的函数返回一个 Verbose 对象

```
Verbose createWithPrvalue()
{
  return Verbose();
}
Verbose v = createWithPrvalue();
```

在没有优化发生的时候，编译器在返回 createWithPrvalue() 的时候，做了下面三件事

1.  创建临时的 Verbose 对象，
2.  将临时的 Verbose 对象复制到函数的返回值，
3.  将返回值复制到变量 v。

copy elision 就是将第 1 和第 2 步的对象优化掉了，直接创建 Verbose 对象并绑定到变量 v。这样只用创建一次对象。

下面让我们用可以实际编译和运行的例子来近距离观察 copy elision。(推荐试着运行代码，更好地理解)

下面的代码包含了 Verbose 类和 create() 函数。

```
#include <iostream>
class Verbose {
    int i;
public:
    Verbose() { std::cout << "Constructed\n"; }
    Verbose(const Verbose&) { std::cout << "Copied\n"; }
    Verbose(Verbose&&) { 
        std::cout << "Moved\n"; }
};

Verbose create() {
    Verbose tv;
    return tv;
}

int main() {
    Verbose v = create();
    return 0;
}
```

当没有 copy elision 的时候，也就是使用 - fno-elide-constructors 编译 flag 的时候：

C++ 11 编译代码并运行

```
g++ -std=c++11 -fno-elide-constructors main.cpp -o main
./main
```

输出是

```
Constructed
Moved
Moved
```

而如果用 C++ 17 编译代码和运行，输出是

```
Constructed
Moved
```

从输出，我们可以看到对象被创建后，又被 move 了（C++11 move 两次，C++17move 了一次，下文再分析为什么次数不一样）。如果不理解 move，可以先阅读[什么是 move？理解 C++ Value categories 和 Rust 的 move](https://zhuanlan.zhihu.com/p/374392832)

**让我们看看有 copy elision 的时候输出是什么。C++ 11 和 C++17 这时候输出是一致的**

```
Constructed
```

所以 copy elision 将 move constructor 的调用给省略了（没有 move constructor 的时候，省略 copy constructor）。而在什么时候以及怎么省略的呢?

move constructor 的省略
--------------------

**C++ 11**

如果不发生 copy elision，在 C++ 11`Verbose v = create()`发生下面的 5 件事

1.  未调用函数`create()`前，选择一个空间 v-space 用来存储 v。
2.  准备进入函数`create()`时，选择一个临时的空间 temp-space 来存储函数的返回值。
3.  进入函数后，选择一个地址创建 tv。
4.  函数返回时，将 tv 移动到 temp-space。
5.  赋值的时候，从 temp-space 将 tv 移动到 v-space。

可以看到，发生了两次 move(第四和第五步)，所以输出了两个 "moved"。我们可以从下面的汇编代码进行确定。（如果看不懂汇编并且想要看懂的话，请阅读[如何阅读简单的汇编（持续更新)](https://zhuanlan.zhihu.com/p/368962727)）

![[assets/img/2023-8-18-Copy move elision C++ 17 vs C++ 11/81a954685c6c9c664cd423e4b7b3a0ce_MD5.jpg]]

而当发生 copy elison 的时候，这两个 moved 都被省略了，因为上面的第三步创建 tv, 直接在 v-space 创建了，所以就不用发生第四步和第五步的 move。请看下面的汇编，

![[assets/img/2023-8-18-Copy move elision C++ 17 vs C++ 11/6fde9bb3d5f7d966a410b70db4a4d9f5_MD5.jpg]]

**C++ 17**

当没有 copy elision 的时候，为什么 C++17 只被 move 了一次。因为 C++17 将 temp-space 等同于 v-space，第五步不存在了。对应的汇编代码是

![[assets/img/2023-8-18-Copy move elision C++ 17 vs C++ 11/b2086f5cbccdf66be43ee4fd08ce9a0e_MD5.jpg]]

很神奇吧！C++ 17 跟 C++11 的不同，源于 C++17 的 value categories 有了更容易理解和一致的定义——将 prvalue 定义为用于初始化对象的值。create() 返回的是 prvalue，而 prvalue 用来创建 v 的时候，是直接构造。所以没有了 temp-space 到 v-space 的 move。更多关于 value categories 请看[什么是 move？理解 C++ Value categories 和 Rust 的 move](https://zhuanlan.zhihu.com/p/374392832)

RVO 和 NRVO
----------

请问下面的三个写法有什么区别，哪一个是推荐的写法，哪一个是不推荐的？

```
// 写法一
Verbose create() {
  return Verbose();
}
//写法二
Verbose create() {
  Verbose v;
  return v;
}
//写法三
Verbose create() {
  Verbose v;
  return std::move(v);
}
```

RVO 是返回值优化，就是将返回值的创建省略了。NRVO 是函数内具有名字的局部变量充当返回值的时候，它的创建也被省略了。它们的省略结果都是在最终的变量里创建对象。

在 C++ 没有 move 的时候，很多编译器实现就支持 RVO(return value optimization) 和 NRVO(named return value optimization)。这两种优化用于在返回值的时候避免不必要的 copy。在 C++11 以后，引入了 move，move 的存在与 copy elision 交织在一起。在这里直接给出答案，写法一是推荐的，写法二场景需要就推荐，而写法三是不推荐的。大家可以通过运行代码查看对应的输出从而得到答案。

C++17 以后，RVO 是保证会发生的，因为标准这么定义了。而 NRVO 并没有保证会发生，但是大部分编译器都会实现。所以在 C++17 里面，没有 copy elision 的时候只有一个 moved，因为 RVO 也存在，而这个 moved，是由于 NRVO 没有开启生成的。如果开启优化，那么一个 moved 也没有，因为 NRVO 把这个 moved 给优化掉了。当用 prvalue 创建对象的时候，直接会构造对象，而不用调用那几个构造函数。

所以 C++17 以后就不用再提 RVO 了，因为已经是语言标准，而不是优化了。

之前有人请教，函数调用者怎么知道是不是发生了 copy elision，如果不知道发生了 copy elision，那么怎么正确地调用函数？

实际上，函数调用者不需要知道是不是发生了 copy elision，因为 non-trivial 对象返回都是 Caller 直接传递存储对象的地址。区别是发生了 copy elision 的时候，直接在传递的地址构造了对象；而没有发生 copy elision 的时候，需要先创建对象，然后将对象 move 到 temp-space，最后 move 到 v-space。更多内容请看上面的讲解和汇编。

参考文献
----

[Copy elision - cppreference.com](https://link.zhihu.com/?target=https%3A//en.cppreference.com/w/cpp/language/copy_elision)

[Interaction between Move Semantic and Copy Elision in C++](https://link.zhihu.com/?target=https%3A//source.coveo.com/2018/11/07/interaction-between-move-semantic-and-copy-elision-in-c%2B%2B/)  
[Wording for guaranteed copy elision through simplified value categories](https://link.zhihu.com/?target=http%3A//www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0135r1.html)