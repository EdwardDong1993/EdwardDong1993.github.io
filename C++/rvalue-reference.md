## 0 关于值类别

**值类别（value category）** 的概念，即平时所说的 **左值** 和 **右值**，C++ 的表达式要么是左值，要么是右值。

- 当对象是左值时，用的是对象的身份（内存中的位置），例如普通变量；
- 当对象是右值时，用的是对象的值（内容），例如字面常量、表达式求值过程中创建的临时对象；

判断一个值是左值还是右值，最有效的办法之一就是看 **能否对这个值进行取址操作**，如果可以，则该值为左值，否则为右值；另外，也可以通过名称来判断，有名称的变量为左值，没有名称的变量为右值。

值类别更详细的说明可参考：https://edwarddong1993.github.io/C++/value-categories.html

## 1 左值引用

左值引用可以绑定左值，但不能直接绑定右值：
```cpp
int a = 42;
int &r1 = a;    // ok
int &r2 = 42;   // error
```

const 的左值引用可以绑定右值：
```cpp
const int &r2 = 42; // ok
```

## 2 右值引用

右值引用可以绑定右值，但不能绑定左值：
```cpp
int a = 42;
int &&rr1 = a;      // error
int &&rr2 = 42;     // ok
```
一个右值引用绑定临时对象的例子如下：
```cpp
class test {
public:
    test() = default;
    test(const test &other) {
      std::cout << "copy constructor" << std::endl;
    }
    test(test &&other) {
        std::cout << "move constructor" << std::endl;
    }
};

test get_test_obj() {
    return test();
}

int main () {
    auto &&t = get_test_obj();
    return 0;
}
```

如果想要使右值引用绑定某个左值，可以使用 `std::move` 将左值强制变为右值引用（`std::move` 的原理在下文进行介绍）：
```cpp
int a = 42;
int &&rr = std::move(a);    // ok
```

由于右值绑定的对象都是临时的，在使用右值引用的时候，所引用的对象状态如下：
- 所引用的对象将要被销毁；
- 该对象没有其他用户；

所以，**使用右值引用的代码可以接管所引用的对象的资源**。

> **[info] For info**
>
> 右值引用指向将要被销毁的对象，一次，可以从绑定到右值引用的对象中窃取资源。

由此，右值引用引申出了如下两个功能：
- **移动语义（Move Semantics）**
- **完美转发（Perfect Forward）**

## 3 引用折叠规则

在介绍移动语义和完美转发之前，需要先介绍一下 **引用折叠**。

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

上述两个规则导致了两个重要结果：
- 如果一个函数参数是一个指向模板参数的右值引用（T&&），则它可以被绑定到一个左值；
- 如果实参是一个左值，则推断出的模板实参将会是一个左值引用，且函数参数将被实例化为一个普通的左值引用参数（T&）；

> **[info] For info**
>
> 引用折叠只能用于间接创建的引用的引用，如 **类型别名** 或者 **模板参数**。

## 4 移动语义

在这里继续看一下 `std::move` 的实现，事实上 `std::move` 唯一的功能是就是把输入的参数强制转化为右值引用，具体实现如下：
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

如果输入的参数是左值，经过引用折叠后会变成左值引用，而 `std::remove_reference<_Tp>::type&&>` 操作会将其类型变成一个右值引用并返回。

移动语义还可以为类定义移动构造函数（move constructor）和移动赋值（move assignment）运算符：
```cpp
X(X&&);
X& operator=(X&&);
```

> **[info] For info**
>
> 上述中入参均为右值引用，表示该入参中的资源可以被“窃取”走，实现资源的移动，取代复制操作，避免了资源消耗，而由于输入参数中包含的值可能已经被移动走了，后续不应当再使用该变量。

## 5 完美转发

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

使用 `std::forward`，能保持原始参数类型以及相应的属性，其源码如下：
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
`std::forward` 提供左值和右值两个接口，分别返回其相应的折叠引用，保证转发的性质不变，另外，`std::forward` 必须通过显示模板实参来调用，最终转发函数可修改如下：
```cpp
template <typename F, tyename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2) {
    f(std::forward<T2>t2, std::forward<T1>(t1));
}
```

## 参考
- https://blog.csdn.net/xhtchina/article/details/120619910
- https://zhuanlan.zhihu.com/p/335994370
- 《C++ primer》
