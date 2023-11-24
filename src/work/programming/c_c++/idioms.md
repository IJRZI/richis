# Idioms

## 1. CRTP

我想实现如下伪代码所示的编程模式, 其中 `Host` 代表 C/S 模型中的任何一个端点, `Server` 和 `Client` 都需要处理收到的包, 但对于每个类型的包, 它们有不同的处理方式. 收包的逻辑是相同的, 因此我希望复用 `handle_packet`. 最简单的方法就是用虚函数
```c++
class Host {
    void handle_packet(...) {
        // do receive packet
        switch (packet_type) {
        case (a):
            handle_packet_a();
            break;
        case (b):
            handle_packet_b();
            break;
        };
    };
    virtual void handle_packet_a();
    virtual void handle_packet_b();
};

class Server: public Host {
    virtual void handle_packet_a() override;
    virtual void handle_packet_b() override;
};

class Client: public Host {
    virtual void handle_packet_a() override;
    virtual void handle_packet_b() override;
};
```

但这样设计只是为了复用 `handle_packet` 的逻辑, 而没有动态绑定的需求, 因此引入虚函数增加了没有必要的开销

这种编译期多态可以通过 CRTP 实现
```c++
template<typename Derived>
class Host {
    void handle_packet() {
        switch (packet_type) {
        case (a):
            static_cast<Derived&>(*this).handle_packet_a();
            break;
        case (b):
            static_cast<Derived&>(*this).handle_packet_b();
            break;
        };
    }
};

class Server: public Host<Server> {
    void handle_packet_a();
    void handle_packet_b();
};
// ...
```

这样可以完全去除虚函数

## 2. SFINAE
> **S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror

起因是想在用 CRTP 时实现编译时检测子类是否具有某个方法, 仅当有方法时才调用的需求, 例如:
<!-- 如果不同的子类需要一些不同的逻辑, 比如如下伪码, 则可以通过 SFINAE 来实现, 见 2. SFINAE -->
```c++
template<typename D>
class Animal {
    void live() {
        while (true) {
            static_cast<D&>(*this).eat();
            static_cast<D&>(*this).sleep();
            static_cast<D&>(*this).bark(); // compile error! Cat will never bark
            // Want: if constexpr (D has member `bark`) then call `bark`
        }
    }
};

class Cat: public Animal<Cat> {
    void eat();
    void sleep();
};

class Dog: public Animal<Dog> {
    void eat();
    void sleep();
    void bark();
};
```

这种在编译期模板实例化时确定模板参数是否有某种性质的行为被称为[内省](https://zh.m.wikipedia.org/zh-hans/%E5%86%85%E7%9C%81_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))(introspection)

包含模板的重载函数的候选集中的某些(或者全部)候选函数来自 模板实参替换模板形参 的模板实例化结果, 在这个过程中, 某些模板的实参替换可能会失败, 这种替换失败(Substitution Failure)并不会立即被当作编译错误(Error)抛出, 这个替换失败的模板会被从候选集中删除, 只要到最后存在成功的替换, 即重载函数候选集不为空, 则这个重载函数的解析就是成功的, 编译也能通过

见来自 [替换失败并非错误](https://zh.m.wikipedia.org/zh-hans/%E6%9B%BF%E6%8D%A2%E5%A4%B1%E8%B4%A5%E5%B9%B6%E9%9D%9E%E9%94%99%E8%AF%AF) 的例子:
```c++
struct Test {
  typedef int foo;
};

template <typename T>
void f(typename T::foo) {}  // Definition #1

template <typename T>
void f(T) {}  // Definition #2

int main() {
  f<Test>(10);  // Call #1.
  f<int>(10);   // Call #2. 并无编译错误(即使没有 int::foo)
                // thanks to SFINAE.
}
```
在编译时, `f<Test>(10)` 会针对 `f` 的两个定义做两次 Substitution
1. 第一次替换得到的函数定义是 `void f(typename Test::foo) {}`
2. 第二次替换得到的函数定义是 `void f(Test) {}`
因此这个调用拥有两个可能的候选, 而根据实参 `10` 的类型可以推导得到只有 1. 符合要求, 因此最终会调用 1., 编译通过

`f<int>(10)` 也会针对 `f` 的两个定义做两次 Substitution
1. 得到 `void f(typename int::foo) {}`, 但 `int::foo` 并不存在, 因此这个替换失败了, 这个函数并不会进入候选集
2. 得到 `void f(int) {}`
因此这个调用拥有一个可能的候选, 而实参 `10` 的类型可以匹配这个唯一的候选, 因此最终会调用 2., 编译通过

上面的例子里, SFINAE 恰好干了在开头时我想干的事: 在编译时判断一个成员是否存在于 `struct/class` 中. 

经过修改可以得到以下真正实现了这个需求的代码, 检查类型 `T` 上是否有拥有 `bark` 成员函数:
```c++
template<typename T>
struct has_member_bark {
    private:
        template<typename U> static auto check(int) 
            -> decltype(std::declval<U>().bark(), std::true_type());
        template<typename U> static std::false_type check(...);
    public:
        enum {value = std::is_same<decltype(check<T>(0)), std::true_type>::value};
};

// 接上面 Animal 的例子...
if constexpr (has_member_bark<D>::value) {
    static_cast<D&>(*this).bark();
}
```

上述例子工作的原理是
1. 编译时要对 `has_member_bark<T>::value` 求值, 其值等于 `std::is_same<decltype(check<T>(0)), std::true_type>::value`
2. 需要推导 `decltype(check<T>(0))`
3. `check<T>(0)` 有两个可选的模板
    - `template<typename U> static auto check(int) -> decltype(std::declval<U>().bark(), std::true_type())`
    - `template<typename U> static std::false_type check(...)`
    
    其中后者由于指定了 `...` 作为参数, 因此拥有最低的匹配优先级, 所以编译器会优先尝试第一个定义
    
    第一个定义的返回值需要被推导, 其类型为 
    
    ```c++
    decltype(std::declval<U>().bark(), std::true_type())
    ```
    
    `decltype` 内是一个 comma expr, 其值等于最后一个表达式的值, 但是求值是从前到后进行的, 因此必须先推导 `std::declval<U>().bark()` 的类型, 这时
    - 如果 `U::bark` 不存在, 则这个替换就会失败, 因此 `check` 的第一个模板就会被删除, 只留下第二个, 则 `decltype(check<T>(0))` 为 `false_type`
    - 如果 `U::bark` 存在, 则这个替换成功, `check` 的第一个模板成为最终的选择, `decltype(check<T>(0))` 为 `true_type`
4. 无论 `T::bark` 是否存在, 由于 SFINAE, 最终总有一个 `check` 被匹配, 如果 `T::bark` 存在, 则 `value` 最终为 `true_type`, 否则是 `false_type`

一个完整的可编译的例子:
```c++
#include <iostream>

template<typename T>
struct has_member_bark {
    private:
        template<typename U> static auto check(int) 
            -> decltype(std::declval<U>().bark(), std::true_type());
        template<typename U> static std::false_type check(...);
    public:
        enum {value = std::is_same<decltype(check<T>(0)), std::true_type>::value};
};

template<typename D>
class Animal {
public:
    void live() {
        // while (true) {
            static_cast<D&>(*this).eat();
            static_cast<D&>(*this).sleep();
            if constexpr (has_member_bark<D>::value) {
                static_cast<D&>(*this).bark();   
            }
        // }
    }
};

class Cat: public Animal<Cat> {
public:
    void eat() {
        std::cout << "Cat eat" << std::endl;
    }
    void sleep() {
        std::cout << "Cat sleep" << std::endl;
    }
};

class Dog: public Animal<Dog> {
public:
    void eat() {
        std::cout << "Dog eat" << std::endl;
    }
    void sleep() {
        std::cout << "Dog sleep" << std::endl;
    }
    void bark() {
        std::cout << "Woof!" << std::endl;
    }
};

int main() {
    Cat().live();
    Dog().live();
    return 0;
}
```

输出
```
Cat eat
Cat sleep
Dog eat
Dog sleep
Woof!
```

## 3. PIMPL
> **P**ointer to **IMPL**ementation

写了一个库, 起初供用户 include 的头文件里有这样的声明:
```c++
class Server {
    public:
        Server();
        void start();
    private:
        void receive_packets();
        void handle_packet();
        void handle_packet_data();
        void handle_packet_ping();
        void handle_packet_pong();
    private:
        int socket;
        // more members ...
};
```

但首先, 暴露给用户的接口只有 `start`, 用户在看头文件时只需要看到他能使用的接口即可, 看到一堆其他的东西会干扰阅读, 其次暴露太多实现细节也不好

可以用 pimpl 来实现 implementation 的隐藏, 原理比较简单, 只贴代码了:
```c++
// server.h
class Server {
    public:
        Server();
        void start();
    private:
        class ServerImpl;
        std::unique_ptr<ServerImpl> impl;
};

// server.cpp
Server::Server(): impl(std::make_unique<ServerImpl>()) {}
void Server::start() {
    while (true) {
        impl->receive_packets();
    }
}

class Server::ServerImpl {
    public:
        void receive_packets() {
            // ...
        }
    private:
        void handle_packet();
        void handle_packet_data();
        void handle_packet_ping();
        void handle_packet_pong();
    private:
        int socket;
        // more members ...
}
```

这样所有的实现细节都被隐藏到 source file 里了