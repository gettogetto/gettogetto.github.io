
要知道详细的原因和动机可以看提案文档：

[http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1674.pdf​www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1674.pdf](https://link.zhihu.com/?target=http%3A//www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1674.pdf)

总结来说就是用什么方法保证迭代器内容的常量化。以往我们写迭代器的[for循环](https://www.zhihu.com/search?q=for%E5%BE%AA%E7%8E%AF&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2289608033%7D)是什么样的？

```cpp
vector<MyType> v;
typedef vector<MyType>::iterator iter;
    for( iter it = v.begin(); it != v.end(); ++it ) {
}
```

如果想让迭代器是一个常量该怎么办？

```cpp
for( const iter it = v.begin(); it != v.end(); ++it ) {
}
```

错！const iter it会导致++it编译失败，因为it是常量无法++。所以这时候需要一个可以++，但是保证不能操作it内容的类型，于是container::const_iterator出现了。所以上面的代码可以写为：

```cpp
vector<MyType> v;
typedef vector<MyType>::const_iterator iter;
    for( iter it = v.begin(); it != v.end(); ++it ) {
}
```

别高兴的太早，解决上面问题后，问题又来了！那就是auto，大家已经很熟悉auto了吧。它可以去推导初始化的变量类型，并且常常用于迭代器上，因为迭代器的写法确实很冗长，完全可以用auto来代替。但是，如果使用：

```cpp
for( auto it = v.begin(); it != v.end(); ++it ) {
}
```

显然无法获得一个const_iterator。

这个时候我们就需要一个返回const_iterator类型的函数，所以[cbegin](https://www.zhihu.com/search?q=cbegin&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2289608033%7D)和cend就孕育而生。

故事大概就是这样的。