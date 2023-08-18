> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzI4ODE4NTkxMw==&mid=2649441370&idx=1&sn=b8f4d13501cb1a4bf376c72779f716fb&chksm=f3ddaf0cc4aa261a76a50b15745573e148962b17444d99ff1a7162534aefcf0ce67c547ac4fd&token=2061556422&lang=zh_CN#rd)

> Copy elision 在 C++11 以前是省略 copy constructor，但是 C++11 之后，更多地是指省略 move constructor。
> 
> C++ 17 与 11 省略的方式还不一样，因为 value categories 有了调整。详情请看本文分解。

什么是 copy elision?


---------------------

Copy elision 是指编译器为了优化，将不需要的 copy/move  操作（含析构函数）直接去掉了。它由 RVO 和 NRVO 组成。

  
要理解 copy elision，让我们先看看函数是如何返回值的。比如下面的函数返回一个 Verbose 对象

```
Verbose create() {
    Verbose tv;
    return tv;

}
```

在没有优化发生的时候，编译器 (C++11）调用 createWithPrvalue() 的时候，做了下面三件事

1.  创建临时的 Verbose 对象，
    
2.  创建返回值对象——即将第 1 步的临时 Verbose 对象复制到函数的返回值存储区域 (如果是小对象，这块区域一般是寄存器，大对象则是栈上的某块空间），
    
3.  将返回值复制到变量 v。
    

注：这里的复制是泛指，指代 copy 或者 move 操作。

copy elision 是指将第 1 和第 2 步的创建 Verbose 对象优化掉了，只在第三步直接创建 Verbose 对象并绑定到变量 v。这样只创建了一次对象。

下面让我们用可以实际编译和运行的例子来近距离观察 copy elision。(推荐运行代码，更好地理解)

下面的代码包含了 Verbose 类和 create() 函数。

```
#include <iostream>

class Verbose {

    int i;

public:

    Verbose() { std::cout << "Constructed\n"; }
    Verbose(const Verbose&) { std::cout << "Copied\n"; }
    Verbose(Verbose&&) { 


        std::cout << "Moved\n"; 

     }

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

从以上输出，我们可以看到对象被创建后，又被 move 了。（C++11 move 两次，C++17 move 了一次，你应该会好奇为什么次数不一样，下文会分析为什么）。

注：如果不理解 move，可以先阅读[理解 C++ 右值左值等名词的关键——move](http://mp.weixin.qq.com/s?__biz=MzI4ODE4NTkxMw==&mid=2649441284&idx=1&sn=008f68f7f16c4b852a368b7f6d3eadfc&chksm=f3ddaf52c4aa26443f3179f9092f6a30f85b07deffd77cd8be06f9a5f4e95de1906099f55da4&scene=21#wechat_redirect)

让我们先看看有 copy elision 优化的时候输出是什么。

这时候，C++ 11 和 C++17 的输出是一致的

```
Constructed
```

所以 copy elision 将 move constructor 的调用给省略了（没有 move constructor 的时候，省略 copy constructor）。

而在什么时候以及怎么省略的呢?

move constructor 省略

* * *


------------------------------

C++ 11

如果不发生 copy elision，在 C++ 11，Verbose v = create()

```
Verbose create() {
    Verbose tv;
    return tv;
}
```

发生下面的 5 件事

1.  未调用函数 create() 前，选择一个空间 v-space 用来存储 v。
    
2.  即将进入函数 create() 时，选择一个临时的空间 return-space 来存储函数的返回值。
    
3.  进入函数后，选择一个地方创建变量 tv。
    
4.  函数返回时，将 tv 移动到 return-space。
    
5.  赋值的时候，从 return-space 将 tv 移动到 v-space。
    
6.  至此，变量 v 被创建成功。
    

   （有个动图就好了~ 但暂时没有用得顺手的画动图软件）  

可以看到，发生了两次 move(第四和第五步)，所以输出了两个 "moved"。我们可以从下面的汇编代码进行确定。

注：如果看不懂汇编并且想要看懂的话，请阅读如何阅读简单的汇编（文章随后会发在公众号上）

![[assets/img/未命名/247a89425ad74e2dfbaa4def8bfb83ef_MD5.png]]

而当发生 copy elison 的时候，这两个 moved 都被省略了。因为上面的第三步创建 tv，直接在 v-space 创建了，省略了第四步和第五步的 move。

如果不信，请看下面的汇编，

![[assets/img/未命名/9099945f790302cb82d164ab79fad2a8_MD5.png]]

C++ 17

当没有 copy elision 的时候，为什么 C++17 只被 move 了一次，不像 C++11 那样被 move 了两次呢？。

  
因为 C++17 将 return-space 等同于 v-space，第四步直接将 tv 移动到了 v-space，第五步不存在了。对应的汇编代码（我应该注解一下汇编，以后吧：））是

![[assets/img/未命名/1dea3f73feb2cadd678682002f04a104_MD5.png]]

很神奇吧！C++ 17 跟 C++11 的不同，源于 C++17 的 value categories 有了更容易理解和一致的定义——将 prvalue 定义为用于初始化对象的值。

  
create() 函数的返回值是 prvalue，prvalue 不会调用构造函数 (不论是移动构造，还是拷贝构造，还是直接构造）。

只有当创建左值对象的时候才会调用构造函数。也就是当变量 v 要被创建的时候，对象才被构造出来。而此时源头是变量 tv，所以要调用移动构造函数。

  
所以没有了 return-space，也就没有了变量 tv 到 return-space 的 move，只有 tv 到 v-space 的 move。

更多关于 value categories 请看[理解 C++ 右值左值等名词的关键——move](http://mp.weixin.qq.com/s?__biz=MzI4ODE4NTkxMw==&mid=2649441284&idx=1&sn=008f68f7f16c4b852a368b7f6d3eadfc&chksm=f3ddaf52c4aa26443f3179f9092f6a30f85b07deffd77cd8be06f9a5f4e95de1906099f55da4&scene=21#wechat_redirect)

而当有 copy elision 的时候，实际上是发生了 RVO 和 NRVO。下面让我们看看 RVO 和 NRVO 具体指什么。

RVO 和 NRVO


--------------

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

RVO（return value optimization）是返回值优化，就是将返回值的创建省略了。

  
NRVO（named return value optimization）是当具有名字的局部变量充当返回值的时候，局部变量的创建也被省略了。

它们的省略结果都是在最终的变量里创建对象。在一开始的例子，就是变量 v。

C++ 没有 move 的时候，很多编译器实现就支持 RVO(return value optimization) 和 NRVO(named return value optimization)。这两种优化可以避免不必要的 copy。

C++11 以后，引入了 move，move 的存在与 copy elision 交织在一起。

写法一是推荐的，写法二场景需要就推荐，而写法三是不推荐的。大家可以通过运行代码查看对应的输出来验证其中的区别。

C++17 以后，RVO 是保证会发生的，因为标准这么定义了：返回值是 prvalue，我们不用构造 prvalue 对象，当对象真正存在的时候，再去构造对象。什么时候是真正存在呢？当左值对象被需要的时候。

即使在 C++17，NRVO 并没有保证会发生，但是大部分编译器都会实现。

所以在 C++17 里面，没有 copy elision 的时候只有一个 moved 而不是两个，是因为 RVO 也存在。而这一个 moved，是由于 NRVO 没有开启生成的。

如果开启 copy elision 优化，那么一个 moved 也没有，因为 NRVO 把这个 moved 给优化掉了。

**caller 怎么知道是否发生了 copy elision**

有人可能会好奇，函数调用者 (caller) 怎么知道是不是发生了 copy elision，如果不知道发生了 copy elision，那么怎么正确地调用函数？  

实际上，函数调用者不需要知道是不是发生了 copy elision，因为 non-trivial 对象的返回都是 Caller 直接传递存储对象的地址。也就是你会把 v-space 传递给函数。

  
区别是发生了 copy elision 的时候，直接在传递的地址构造了对象；而没有发生 copy elision 的时候，需要先创建对象，然后将对象 move 到 temp-space，最后 move 到 v-space。（没有 move 函数，就是调用拷贝函数）

**总结**

C++17 以后，prvalue（纯右值）诞生了，我们不用再提 RVO 优化了。因为已经是语言标准，而不是优化了。

[什么是纯右值，请看理解 C++ 右值左值等名词的关键——move](http://mp.weixin.qq.com/s?__biz=MzI4ODE4NTkxMw==&mid=2649441284&idx=1&sn=008f68f7f16c4b852a368b7f6d3eadfc&chksm=f3ddaf52c4aa26443f3179f9092f6a30f85b07deffd77cd8be06f9a5f4e95de1906099f55da4&scene=21#wechat_redirect)  

参考文献


--------

Copy elision - cppreference.com

Interaction between Move Semantic and Copy Elision in C++Wording for guaranteed copy elision through simplified value categories