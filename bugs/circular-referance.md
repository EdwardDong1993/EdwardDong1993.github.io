## 1 问题简述

代码简化如下：
```cpp
class circular_referance {
public:
    circular_referance() {
        std::cout << "circular_referance constructed" << std::endl;
    }
    ~circular_referance() {
        std::cout << "circular_referance destructed" << std::endl;
    }
    void init(std::function<void()> f) {
        callback = f;
    }
    void add_count() {
        count++;
    }

private:
    std::function<void()> callback{nullptr};
    int count{0};
};

int main() {
    auto ptr = std::make_shared<circular_referance>();
    auto func = [=] {
        ptr->add_count();
    };
    ptr->init(func);

    return 0;
}
```

终端显示：
```
circular_referance constructed
```

## 2 修改

将 lambda 传入的 shared_ptr 改为 weak_ptr:
```cpp
int main() {
    auto ptr = std::make_shared<circular_referance>();
    auto func = [weak_ptr = std::weak_ptr<circular_referance>(ptr)] {
        if (weak_ptr == nullptr) {
            return;
        }
        ptr->add_count();
    };
    ptr->init(func);

    return 0;
}
```

## 3 总结

这个业务层面的实现是根据 go 的代码转成 C++ 的，go 的部分特性没法实现的时候，靠 C++ 强行拼凑，一个不仔细就造成了这种情况。

1. 智能指针优先使用 unique_ptr；
2. 如果设计层面决定了这个对象确实需要多线程访问（首先确定设计层面对不对），那么使用 shared_ptr；
3. 使用 shared_ptr 一定要梳理清楚是否存在循环引用的情况，确保存在相互引用的情况下，使用 weak_ptr 打破循环引用；如果两者之间有逻辑上的父子关系，父持有子的 shared_ptr，子持有父的 weak_ptr；
4. lambda 优先使用引用捕获;

