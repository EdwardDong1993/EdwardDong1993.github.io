**资源申请即初始化（Resource Acquisition Is Initialization，RAII）**，是 C++ 的一种 **管理资源**、**避免泄漏** 的方法，其做法是：
1. 使用一个对象，在其构造时获取对应的资源；
2. 在对象生命期内控制对资源的访问，使之始终保持有效；
3. 最后在对象析构的时候，释放构造时获取的资源；

因此 RAII 也可以说是一种利用 **对象生命周期** 来控制程序资源的技术，资源的有效期与持有资源的对象的生命期严格绑定。

系统的资源不具有自动释放的功能，但由于 C++ 中的类具有自动调用析构函数的功能，所以如果把资源用类进行封装起来，对资源操作都封装在类的内部，在析构函数中进行释放资源，当定义的局部变量的生命结束时，它的析构函数就会自动的被调用，其生命周期是由操作系统来管理，无需人工介入。

以 `lock_guard` 为例，只需要加锁操作资源，而不需要手动解锁，锁自动释放：
```cpp
std::mutex m;
void fn() {
    std::lock_guard<std::mutex> guard(m);
    do_something();
    // 执行完成后，lock_guard 会被析构，从而 mutex 也会被释放
}
```

`lock_guard` 的源码如下：
```cpp
  /** @brief A simple scoped lock type.
   *
   * A lock_guard controls mutex ownership within a scope, releasing
   * ownership in the destructor.
   */
  template<typename _Mutex>
    class lock_guard
    {
    public:
      typedef _Mutex mutex_type;

      explicit lock_guard(mutex_type& __m) : _M_device(__m)
      { _M_device.lock(); }

      lock_guard(mutex_type& __m, adopt_lock_t) noexcept : _M_device(__m)
      { } // calling thread owns mutex

      ~lock_guard()
      { _M_device.unlock(); }

      lock_guard(const lock_guard&) = delete;
      lock_guard& operator=(const lock_guard&) = delete;

    private:
      mutex_type&  _M_device;
    };
```
`lock_guard` 构造时，会初始化内部字段执行加锁操作；析构时，会指向解锁操作，所以借助这个类，可以不需要关注解锁的操作。
