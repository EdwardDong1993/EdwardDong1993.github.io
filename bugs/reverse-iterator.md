## 1 问题描述

简化了代码如下，循环出不来了：
```cpp
std::map<int, int> m{{0,0}};
for (auto it = m.rbegin(); it != m.rend();) {
	if (it->first == 0) {
		m.erase((++it).base());
	} else {
		it++;
	}
}
```

## 2 reverse_iterator

reverse_iterator 反向迭代器继承自普通迭代器，是一个迭代器适配器，它反转了给定迭代器的方向。
```cpp
  template<typename _Iterator>
    class reverse_iterator
    : public iterator<typename iterator_traits<_Iterator>::iterator_category,
		      typename iterator_traits<_Iterator>::value_type,
		      typename iterator_traits<_Iterator>::difference_type,
		      typename iterator_traits<_Iterator>::pointer,
                      typename iterator_traits<_Iterator>::reference>
```

![](<../.github/images/bugs1.png>)

- **物理位置**
对应迭代器在内存中的实际位置，反向迭代器与普通迭代器在保持了一一对应，即 `rbegin()` 对应 `end()` 的位置，`rend()` 对应 `begin()` 的位置
- **逻辑位置**
对应迭代器对应容器中元素的位置，`rbegin()` 对应的元素为 `end()` 的前一个元素，`rend()` 对应的元素为 `begin()` 的前一个元素，反向迭代器索引到的均为物理位置上的下一个元素。

有些容器的成员函数包括只支持普通迭代器作为参数，反向迭代器可通过 `base()` 方法得到逻辑位置上对应的普通迭代器。下图展示了迭代器间的物理位置、逻辑位置、base 间的关系：

![](<../.github/images/bugs2.png>)

由图可知，反向迭代器 `r_it` 实际指向的元素在 `&*r_it` 位置上，如果需要获取普通迭代器处理该位置元素，则需要获取 `(++it).base()` 的普通迭代器从而指向该元素。

## 3 map 删除元素

有两种方法可以通过迭代器删除元素：
1. `erase` 方法返回指向了删除元素的下一个元素：
```
it = c.erase(it);
```
2. 手动将 it 指向为下一个元素，删除当前元素；
```cpp
c.erase(it++);
```

对于反向迭代器删除，可以使用上述的第一种方式，先删除，再将返回的普通迭代器转换成反向迭代器：
```cpp
it = std::reverse_iterator<std::map<int, std::string>::iterator>(c.erase((++it).base()));
```

为什么不能使用第二种方式直接 `m.erase((++it).base())` ？ `(++it).base()` 被删除，则 `it` 已经失效，不同于普通迭代器的处理，再下一次进入循环操作 `it` 已属于是 UB。