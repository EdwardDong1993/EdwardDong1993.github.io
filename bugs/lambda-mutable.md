## 1 问题简述

代码简化如下：
```cpp
class test_t {
public:
    void print_cnt(std::shared_ptr<int> ptr) {
        std::cout << ptr.use_count() << std::endl;
    }

    int a{0};
};

int main() {
    auto int_ptr = std::make_shared<int>(0);
    auto l = [ptr = std::move(int_ptr)] {
        test_t t;
        t.print_cnt(std::move(ptr));
    };
    l();

    return 0;
}
```

期望计数值为1，实际计数值为2。

## 2 修改

想要在lambda表达式的函数体中修改非静态局部变量的值，需要引用捕获，如果非要修改值捕获的变量，需要加上关键字mutable，否则代码中的 move 是无效的。

代码修改如下：
```cpp
    auto int_ptr = std::make_shared<int>(0);
    auto l = [ptr = std::move(int_ptr)] mutable {
        test_t t;
        t.print_cnt(std::move(ptr));
    };
    l();
```

自己在 vscode 中编码并没有错误提示，同事用 visual studio 直接出现操作无效的提示，我哭晕。

lambda 的用法相当灵活，后续找机会进行总结。
