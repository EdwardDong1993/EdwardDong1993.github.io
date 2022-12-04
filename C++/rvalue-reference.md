---
title: 右值引用
category: C++
order: 1
---

## 1 左值，右值
如下例所示：
```cpp
class test {
public:
    test() {};
    int foo{0};
    int bar{0};
};
 
test t = test();
```

其中，`t` 可以通过 `&` 取地址，位于等号左边，所以 `t` 是左值；而 `test()` 是个临时值，没法通过 `&` 取地址，位于等号右边，所以 `test()` 是个右值。

总结来说：**有地址的变量就是左值，没有地址的字面值、临时值就是右值。**

## 2 左值引用
左值引用可以指向左值，但不能指向右值（引用是变量的别名，由于右值没有地址，没法被修改，所以普通左值引用无法指向右值）：
```cpp
int a = 42;
int &r1 = a;    // ok
int &r2 = 42;   // error
```
但 const 左值引用可以指向右值，因为不会修改指向值：
```cpp
const int &r2 = 42; // ok
```

## 3 右值引用
右值引用可以指向右值，但不能指向左值：
```cpp
int a = 42;
int &&rr1 = a;      // error
int &&rr2 = 42;     // ok
```

如果想要使右值引用指向某个左值，可以先使用 `std::move` 将左值变为右值，再将右值引用指向这个右值：
```cpp
int a = 42;
int &&rr = std::move(a);    // ok
```

事实上 `std::move` 唯一的功能是把输入的参数强制转化为右值，等同于 `static_cast<T&&>(lvalue)` 类型转换，后续将进行介绍。

## 4 特点

- 被 **声明** 出来的左值引用和右值引用都是左值，因为被声明出的引用都有地址；
- 作为函数形参时，右值引用更灵活，因为右值引用可以直接指向右值，也可以通过 `std::move` 指向左值；虽然 const 左值引用也可以做到左右值都接受，但它无法进行修改，有一定局限性；

## 5 引用折叠

**规则 1**：将一个左值传递给函数的右值引用参数，且此右值引用指向模板类型参数（如 T&&），编译器推断模板类型参数为实参的左值引用类型：
```cpp
template <typename T> void f(T&&);
int i = 42;
f(i);
```
当调用 `f(i)` 时，编译器推断 T 的类型为 `int&` 而非 `int`。

**规则 2**：T 被推断为 `int&` 意味着函数 `f` 的参数应该是 `int&` 的右值引用，但是不能直接定义一个引用的引用（因为引用并不是对象），所以这里给出规则2：如果间接的创建了引用的引用，引用会折叠成一个普通的引用：
- `X& &`，`X& &&`，`X&& &` 将折叠成左值引用 `X&`;
- `X&& &&` 将折叠成为右值引用 `X&&`;

引用折叠只能用于间接创建的引用的引用，如 **类型别名** 或者 **模板参数**；

上述两个规则导致了两个重要结果：
- 如果一个函数参数是一个指向模板参数的右值引用（T&&），则它可以被绑定到一个左值；
- 如果实参是一个左值，则推断出的模板实参将会是一个左值引用，且函数参数将被实例化为一个普通的左值引用参数（T&）；

## 6 std::move

其功能为将输入的参数转换成右值，源代码如下：
```cpp
  /**
   *  @brief  Convert a value to an rvalue.
   *  @param  __t  A thing of arbitrary type.
   *  @return The parameter cast to an rvalue-reference to allow moving it.
  */
  template<typename _Tp>
    constexpr typename std::remove_reference<_Tp>::type&&
    move(_Tp&& __t) noexcept
    { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }

```

## 6 完美转发

某些函数需要将实参连同类型不变的转发给其它函数，需要保持被转发的实参的所有性质，包括是否为 const 以及左值还是右值。

**step 1**
```cpp
template <typename F, tyename T1, typename T2>
void flip(F f, T1 t1, T2 t2) {
    f(t2, t1);
}
```

如果函数 `f` 的参数为引用，则会出现问题：
```cpp
void f(int v1, int & v2) {
    //...
}
```
如果通过 `flip` 调用，就不会改变实参的值。

**step 2**
```cpp
template <typename F, tyename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2) {
    f(t2, t1);
}
```
通过引用折叠，可以保持实参的左值右值属性，但不能传递右值引用参数：
```cpp
void f(int &&i, int &j) {
    //...
}
```
例如调用 `flip(f, i, 42)` 时，在 `flip` 函数内部 `t2` 为变量（左值表达式），不能从左值实例化为右值。

**step 3**

使用 std::forward，能保持原始参数类型以及相应的属性，其源码如下：
```cpp
  /**
   *  @brief  Forward an lvalue.
   *  @return The parameter cast to the specified type.
   *
   *  This function is used to implement "perfect forwarding".
   */
  template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type& __t) noexcept
    { return static_cast<_Tp&&>(__t); }

  /**
   *  @brief  Forward an rvalue.
   *  @return The parameter cast to the specified type.
   *
   *  This function is used to implement "perfect forwarding".
   */
  template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
    {
      static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
		    " substituting _Tp is an lvalue reference type");
      return static_cast<_Tp&&>(__t);
    }
```
std::forward 提供左值和右值两个接口，分别返回其相应的折叠引用，保证转发的性质不变，另外，std::forward 必须通过显示模板实参来调用，最终转发函数可修改如下：
```cpp
template <typename F, tyename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2) {
    f(std::forward<T2>t2, std::forward<T1>(t1));
}
```




## 参考
https://zhuanlan.zhihu.com/p/335994370
