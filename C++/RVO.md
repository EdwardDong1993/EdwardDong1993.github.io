
## 1 简介
返回值优化（Return Value Optimization，RVO），是一种编译器优化技术，可以省略函数返回创建的临时对象，然后可以达到少调用拷贝构造的操作。包含在 C++11 标准中，被命名为 **copy elision**。

## 2 测试
首先定义如下结构：
```cpp
class object {
public:
    object() {
        std::cout << "constructor" << std::endl;
    }
    object(const object&) {
        std::cout << "copy constructor" << std::endl;
    }
    object &operator=(const object &obj) {
        std::cout << "copy assignment operator" << std::endl;
        return *this;
    }
    ~object() {
        std::cout << "destructor" << std::endl;
    }
};
```

另外分别定义使用 RVO 和 NRVO 的接口，其中 RVO 接口返回的是未命名对象，NRVO 接口返回的是命名对象，NRVO 是 RVO 的一种变体：
```cpp
object get_object_rvo() {
    return object();
}

object get_object_nrvo() {
    object obj;
    return obj;
}
```

使用 RVO 接口测试：
```cpp
int main(void) {
    auto obj = get_object_rvo();
    return 0;
}
```

首先测试禁用优化的场景，进行如下编译：
```
g++ -o rvo rvo.cpp -fno-elide-constructors
```

运行输出：
```
constructor
copy constructor
destructor
copy constructor
destructor
destructor
```
一共进行了两次拷贝，第一次是 `get_object_rvo` 内部的对象拷贝到 main 函数中的临时对象，第二次是临时对象拷贝到 `obj` 对象。

然后测试使用优化的场景，进行编译如下（正常编译默认使用优化）：
```
g++ -o rvo rvo.cpp
```

运行输出：
```
constructor
destructor
```

使用 RVO 优化时没有复制构造函数，当应对成本较高时，RVO 使我们能够更快地运行程序。使用 NRVO 接口运行情况也相同。

## 3 move
相比于拷贝构造函数，讲道理移动构造函数更能节省开销，这里在 RVO 中尝试使用移动，看看会产生什么效果。

在 object 结构中添加移动构造函数：
```cpp
    object(object&&) {
        std::cout << "move constructor" << std::endl;
    }
    object &operator=(object &&) {
        std::cout << "move assignment operator" << std::endl;
        return *this;
    }
```

在 RVO 和 NRVO 接口的基础上将返回值进行移动：
```cpp
object get_object_rvo_move() {
    return std::move(object());
}

object get_object_nrvo_move() {
    object obj;
    return std::move(obj);
}
```

使用 RVO 接口测试（这里省略返回值）：
```cpp
int main(void) {
    get_object_rvo_move();
    return 0;
}
```

运行输出：
```
constructor
move constructor
destructor
destructor
```

这里编译器并没有执行 RVO，而是调用移动构造函数。这是就需要看一下C++标准中的 copy elision 的细节：
> in a return statement in a function with a class return type, when the expression is the name of a non-volatile automatic object (other than a function or catch-clause parameter) with the same cv-unqualified type as the function return type, the copy/move operation can be omitted by constructing the automatic object directly into the function’s return value

也就是说返回值类型需要与函数类型一致，但 `get_object_rvo_move` 的返回值类型是 object，而函数内部实际返回的类型是 object&&，没有匹配。修改函数如下：
```cpp
object&& get_object_rvo_move2() {
    return std::move(object());
}

object&& get_object_nrvo_move2() {
    object obj;
    return std::move(obj);
}
```

使用 RVO 接口测试（这里省略返回值）：
```cpp
int main(void) {
    get_object_rvo_move2();
    return 0;
}
```

再次执行运行输出：
```
constructor
destructor
```

省略返回值为了避免测试中出现移动构造函数干扰，因为 get_object_rvo_move2 返回的是右值，使用右值创建对象也会调用移动构造函数，会出现混淆，无法分辨结果。

## 参考
- https://community.ibm.com/community/user/power/blogs/archive-user/2020/05/28/rvo-vs-stdmove
