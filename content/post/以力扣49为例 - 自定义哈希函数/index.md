---
title: 以力扣49为例 - 自定义哈希函数
# description: Welcome to Hugo Theme Stack
slug: self_define_hash
date: 2025-02-13 00:00:00+0000
# image: filesystem.jpg
categories:
    - 疑难经验
tags:
    - C/C++
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
题目链接： https://leetcode.cn/problems/group-anagrams/solutions/520469/zi-mu-yi-wei-ci-fen-zu-by-leetcode-solut-gyoc/

题解中定义了`array<int, 26>`到 `vector<string>`的映射，其中前者为一个大小为26的int数组。C++中，除了vector这种变长数组，还提供了固定大小数组的std::array这个容器类模板。用意是更好的类型检查和防止越界错误，还可以使用标准库的很多算法函数。性能和原生数组相同。使用前需要`#include <array>`。

之所以要定义这个映射，是因为互为字母异位词的两个字符串，他们每个字母出现的频率是相同的。所以用`array<int, 26>`作为key，将字母异位词分组。而key相同的字符串需要插入映射的`vector<string>`中。

unordered_map和unordered_set等基于哈希表的容器默认只能处理内置类型（如整数、指针）和一些标准库类型（string等）。对于自定义类型或复杂类型，如`array<int, 26>`，需要提供一个合适的哈希函数才能作为key。

哈希函数的目的是区分不同的key。合适的哈希函数需要将相同的key映射到相同的slot，而哈希碰撞的概率很小。在实现哈希函数时，注意：
- 函数返回值为size_t类型，这是因为unordered_map是基于哈希表的数据结构，索引是基于哈希值的，而哈希值就是一个无符号整数。不用int是为了避免符号扩展的问题。具体需要参考unordered_map的实现。
- 一般的设计方法有
	- 利用`std::hash<int>`函数，需要`#include <functional>`
	- **位运算**：如位移、异或等操作可以帮助混合不同的比特位。
	- **乘法和加法**：通过乘以素数并加上其他数值，可以使结果更加随机化。
	- **组合多个哈希值**：如果输入数据由多个字段组成，可以分别对每个字段进行哈希计算，然后将这些哈希值组合起来。
总之，自定义哈希函数就是要保证输入相同的key时，返回值相同；输入不同的key时，返回值尽量不同。

本题中，key是一个整数数组。因此可以组合数组中每一位整数的哈希值。
```
size_t my_hash(const std::array<int, 26> &k) {
	int n;
	size_t res = 0;
	for (n : k) {
		hash = (hash << 1) ^ std::hash<int>{}(n);
	}
	return hash;
}
```

至于哈希函数`std::hash<int>{}()`中为什么有{}，这是因为std::hash是一个类模板，()是他定义的函数运算操作符。`std::hash<int>{}`的意思是临时创建一个`std::hash<int>`类型的匿名临时对象，{}是参数列表（不需要参数）。这个对象后面再用()，就是调用了该临时对象定义的操作符。

使用左移和异或就是想要哈希函数映射得更均匀，尽量保留每一位的特征。

题解使用了嵌套的Lambda表达式，太复杂。还可以使用函数对象（仿函数）实现。函数对象是一个具有operator()的类实例。不仅可以像函数一样调用，还可以保存状态。此处不介绍。