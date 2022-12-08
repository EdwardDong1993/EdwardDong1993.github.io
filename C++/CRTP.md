## 1 简介

奇异递归模板（Curiously Recurring Template Pattern，CRTP），本质为 **静态多态**，发生在 **编译期**，基类接收一个派生类作为模板参数，派生类在实现的时候将自身传入，基类利用此模板参数将自身静态转换为派生类类的指针或者引用后调用子类的函数，示例如下：

```cpp
template <typename Child> 
struct Base {
    void interface() { static_cast<Child *>(this)->implementation(); }
};

struct Derived : Base<Derived> {
    void implementation() { std::cout << "Derived implementation\n"; }
};

int main() {
    // is-a
    auto d1 = std::make_unique<Base<Derived>>();
    d1->interface();

    // not only is-a
    Derived d2;
    d2.interface();

    return 0;
}
```

## 2 特点

1. 通过 CRTP 可以使得类具有类似于虚函数的效果，同时又没有虚函数调用时的开销，对象的体积相比使用虚函数也会减少（因为不需要存储虚函数指针），但因此也没有实现动态绑定；
2. 这里使用 static_cast 而非 dynamic_cast ，因为 dynamic_cast 一般是为了确保在运行期由上向下转换的正确性，CRTP 中派生类本身就是基类的模板参数，因此 static_cast 足够保证转换的正确性，并且省去了 dynamic_cast 虚函数调用的开销；
3. 在 `Derived : Base<Derived>` 中，`Base<Derived>` 先于 `Derived` 而存在，此时 `Derived` 是一个不完整类型，但由于 `Base<Derived>::interface()` 只有被调用时才会被实例化，而这时 `Derived` 已经是一个完整类型了；

CRTP 的性能测试可参考：https://zhuanlan.zhihu.com/p/419747485

## 3 析构

按照上述流程，析构函数本应写成如下的形式，但是提前声明，这种形式是错误的：
```cpp
template <typename Child>
class Base {
public:
    ~Base() { static_cast<D*>(this)->~Child(); }    // error
}
```
原因如下：
1. 在基类的析构函数中，实际对象不再是派生类型；
2. 派生类会执行完自己的析构后执行基类的析构，会造成了循环调用行为；

可以专门编写一个方法实现子类析构以解决此问题：
```cpp
template <typename Child>
void destroy(Base<Child>* b) {
    delete static_cast<Child*>(b);
}
```

## 4 委托模式

CRTP 相较于虚函数调用另外一个区别在于，虚函数通过基类指针或引用的调用（is-a的关系），而 CRTP 可以直接调用派生类的对象，可以添加更多的功能，此时基类不再是接口，派生类也不仅仅是接口的实现，派生类扩展了基类接口，同时基类委托了一些行为给派生类。

委托模式案例可参考：https://zhuanlan.zhihu.com/p/142679612

## 5 使用场景

大多数情况下，CRTP 的应用场景是套用某框架模板生成自己的类，并且这样的类可能整个生命周期只有确定的个数，对于动态分发场景或者有大量同一行为模式的类的场景并不适用；

## 6 std::enable_shared_from_this

假如想要从被 shared_ptr 管理的对象中，获的一个指向该对象本身的 shared_ptr，代码大概实现如下：

```cpp
struct Bad
{
    std::shared_ptr<Bad> get_shared_ptr() { return std::shared_ptr<Bad>(this); }
    ~Bad() { std::cout << "Bad::~Bad() called\n"; }
};

int main()
{
	{
		std::shared_ptr<Bad> b1 = std::make_shared<Bad>();
		std::shared_ptr<Bad> b2 = b1->get_shared_ptr();
	}

	return 0;
}
```

但是上面的写法是错误的，违反了智能指针的使用规则：不能使用 shared_ptr 的裸指针去初始化另外一个智能指针，最后将会重复释放，发生未定义的行为。正确的写法应该是继承 enable_shared_from_this，同时模板参数为该类的类型：

```cpp
struct Good: std::enable_shared_from_this<Good>
{
    std::shared_ptr<Good> get_shared_ptr() { return shared_from_this(); }
};

```

enable_shared_from_this 是一个模板类，遵循 CRTP 的使用方法，定义在头文件 `<memory>`。enable_shared_from_this 中存储了一个 weak_ptr，当这个派生类初始化一个 shared_ptr 对象时，如果类继承自 enable_shared_from_this，则会初始化 shared_from_this 中的 weak_ptr 指向的对象的所有权，这样 shared_from_this 时就可以通过此 weak_ptr 而非 this 指针构造返回新的 shared_ptr，保证最初的智能指针析构时，shared_from_this 返回的智能指针也能正常析构。

需要注意的是 enable_shared_from_this 中的 weak_ptr 是通过 shared_ptr 的构造函数初始化的，所以必须在 shared_ptr 构造函数调用之后才能调用 shared_from_this，而且不能对一个没有被 shared_ptr 接管的类调用shared_from_this，否则 C++17 前会产生未定义行为，C++17 后会抛出 `std::bad_weak_ptr` 异常。


最后贴一下 enable_shared_from_this 的源码：
```cpp
  /**
   *  @brief Base class allowing use of member function shared_from_this.
   */
  template<typename _Tp>
    class enable_shared_from_this
    {
    protected:
      constexpr enable_shared_from_this() noexcept { }

      enable_shared_from_this(const enable_shared_from_this&) noexcept { }

      enable_shared_from_this&

      operator=(const enable_shared_from_this&) noexcept
      { return *this; }

      ~enable_shared_from_this() { }

    public:
      shared_ptr<_Tp>
      shared_from_this()
      { return shared_ptr<_Tp>(this->_M_weak_this); }

      shared_ptr<const _Tp>
      shared_from_this() const
      { return shared_ptr<const _Tp>(this->_M_weak_this); }

#if __cplusplus > 201402L || !defined(__STRICT_ANSI__) // c++1z or gnu++11
#define __cpp_lib_enable_shared_from_this 201603
      weak_ptr<_Tp>
      weak_from_this() noexcept
      { return this->_M_weak_this; }

      weak_ptr<const _Tp>
      weak_from_this() const noexcept
      { return this->_M_weak_this; }
#endif

    private:
      template<typename _Tp1>
	void
	_M_weak_assign(_Tp1* __p, const __shared_count<>& __n) const noexcept
	{ _M_weak_this._M_assign(__p, __n); }

      // Found by ADL when this is an associated class.
      friend const enable_shared_from_this*
      __enable_shared_from_this_base(const __shared_count<>&,
				     const enable_shared_from_this* __p)
      { return __p; }

      template<typename, _Lock_policy>
	friend class __shared_ptr;

      mutable weak_ptr<_Tp>  _M_weak_this;
    };
```

## 参考
- https://zhuanlan.zhihu.com/p/137879448
- https://zhuanlan.zhihu.com/p/142407249
- https://zhuanlan.zhihu.com/p/142679612
- https://zhuanlan.zhihu.com/p/419747485
- https://www.zhihu.com/question/45169117/answer/98451509
- https://en.cppreference.com/w/cpp/memory/enable_shared_from_this
