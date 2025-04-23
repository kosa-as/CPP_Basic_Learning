---
title: C++中std::functional的实现
categories:
  - STL
  - 类型擦除
  - 模板匹配
tags:
  - SFINAE
  - 构造顺序
excerpt: 本文介绍了现在C++中的std::functional的一些思想，包括类型擦出，SFINAE的原理介绍等等
---
<!-- more -->

### `std::functional`的基本原理：

- **将任意可调用对象包装为一个类型安全、可复制、可赋值的对象**

- **不需要知道其底层类型（即擦除原始类型）**

- **只要它们满足函数签名即可传参执行**

### `std：：functional`原理分析：

- 可调用对象：

  | 类型                 | 举例                              | 可传给 `std::function` | 说明             |
  | -------------------- | --------------------------------- | ---------------------- | ---------------- |
  | 普通函数指针         | `int(*)(int,int)`                 | ✅                      | 最基础           |
  | Lambda               | `[=](int x){ return x+1; }`       | ✅                      | 最常用           |
  | 函数对象             | `struct F { int operator()(); };` | ✅                      | 可定制行为       |
  | 静态成员函数         | `static int f()`                  | ✅                      | 和普通函数等价   |
  | 成员函数指针         | `&Class::method`                  | ❌（需要包装）          | 必须配合对象使用 |
  | `std::bind` 结果     | `bind(f, 1, _1)`                  | ✅                      | 预设参数         |
  | `std::mem_fn` 结果   | `mem_fn(&C::f)`                   | ✅                      | 成员函数包装     |
  | `std::function` 本身 | `function<void()>`                | ✅                      | 高阶封装         |

-  类型擦除：

  `std::decay_t<T>` 是 C++ 类型萃取中非常常用的工具，它会把一个类型 **退化（decay）** 成适用于函数参数传递的形式，**其行为类似于函数参数的默认类型转换规则**。可以理解为：`std::decay_t<T>` 会去掉引用、cv 限定符、数组/函数类型退化为指针等。

### Function的实现

```cpp
#pragma once
#include <memory>
#include <utility>
#include <type_traits>

template <typename>
class Function; 

//模板特化，R(Args...)表示一个可调用对象，R是返回类型，Args...是参数类型
template <typename R, typename... Args>
class Function<R(Args...)> {
    //ICallable是一个抽象类，定义了一个纯虚函数invoke，用于包裹可调用对象
    class ICallable {
    public:
        virtual R invoke(Args&&... args) = 0;
        virtual std::unique_ptr<ICallable> clone() const = 0;
        virtual ~ICallable() = default;
    };

    //CallableImpl是一个具体类，继承自ICallable，用于包裹可调用对象
    template <typename F>
    class CallableImpl : public ICallable {
        F func;
    public:
        explicit CallableImpl(F&& f) : func(std::forward<F>(f)) {}
        
        R invoke(Args&&... args) override {
            return func(std::forward<Args>(args)...);
        }
        
        std::unique_ptr<ICallable> clone() const override {
            return std::make_unique<CallableImpl<F>>(F(func));
        }
    };

    std::unique_ptr<ICallable> callable;

public:
    // 默认构造函数
    Function() noexcept = default;
    
    template <typename F, typename = std::enable_if_t<!std::is_same<std::decay_t<F>, Function>::value>>
    //构造函数，接受一个可调用对象F，并将其包裹在CallableImpl中
    Function(F&& f)
        : callable(std::make_unique<CallableImpl<std::decay_t<F>>>(std::forward<F>(f))) {}

    // 移动构造和赋值
    Function(Function&& other) noexcept = default;
    Function& operator=(Function&& other) noexcept = default;

    // 拷贝构造函数
    Function(const Function& other) 
        : callable(other.callable ? other.callable->clone() : nullptr) {}
    
    // 拷贝赋值运算符
    Function& operator=(const Function& other) {
        if (this != &other) {
            callable = other.callable ? other.callable->clone() : nullptr;
        }
        return *this;
    }

    R operator()(Args... args) {
        if (!callable) {
            throw std::bad_function_call();
        }
        return callable->invoke(std::forward<Args>(args)...);
    }

    explicit operator bool() const {
        return static_cast<bool>(callable);
    }
};
```

### 代码说明

`ICallable`是所有可调用对象的抽象包裹`wrapper`类，是一个基类。在Function中，通过`*ICallable`指针（或者智能指针）来调用派生类中的`invoke`函数，来实现最后的执行

```cpp
template <typename R, typename... Args> 
class ICallable {
    public:
        virtual R invoke(Args&&... args) = 0;//执行函数
        virtual std::unique_ptr<ICallable> clone() const = 0;//基类并不知道子类是什么样子的，添加一个辅助克隆函数
        virtual ~ICallable() = default;//析构函数，必须要申明为虚函数
};
```

`CallableImpl`是基于`ICallable`的派生类，里面实现了具体的方法。

```cpp
template <typename R, typename... Args>     
template <typename F>
class CallableImpl : public ICallable {
    F func;
    public:
        explicit CallableImpl(F&& f) : func(std::forward<F>(f)) {}//单参构造函数，使用完美转发
        
        R invoke(Args&&... args) override {
            return func(std::forward<Args>(args)...);//使用func(args...)
        }
        
        std::unique_ptr<ICallable> clone() const override {
            return std::make_unique<CallableImpl<F>>(F(func));//深拷贝函数，返回本派生类的一个深拷贝对象
        }
    };
```

然后是Function内容的实现：

首先私有变量采用了`unique_ptr`的设计，来管理类型为基类指针但是指向子类的指针。

- 单参构造函数：`explicit Function(F&& f)`，在上面使用了`SFINAE（substitute failure is not an error）`，即在进行模板匹配的时候，如果发生了模板不匹配，则不认为是一种错误，转而匹配其他的构造函数。（在其他的设计中也有体现，比如`std：：lock_guard`和`std::unique_lock`,两者都要求传入的类实现`lock`和`unlock`方法。）

- 讲到构造函数匹配的问题：下面是构造函数的匹配规则

  | 匹配类别                   | 优先级                     |
  | -------------------------- | -------------------------- |
  | 非模板函数                 | 最高                       |
  | 模板函数（精确匹配）       | 高                         |
  | 模板函数（万能引用 `T&&`） | 较低（仅在无其他更优时选） |
  | 可变参数模板 (`...`)       | 最低                       |

- 为什么要实现辅助的`clone`函数呢？

  在这里可见，在赋值构造和赋值拷贝中，调用的是callable实际指向对象的clone函数，如果不这样，默认的赋值构造与拷贝实现的是浅拷贝，那么会引发一系列的问题

- 为什么要重载`R operator()(Args... args)`呢？

  通过重载这个运算符，来包裹invoke函数，那么就可以直接使用f()的方式调用函数，实现可调用对象的完美还原（不改变使用方式）

```cpp
template <typename>
//模板特化，R(Args...)表示一个可调用对象，R是返回类型，Args...是参数类型
template <typename R, typename... Args>
class Function<R(Args...)> {
    //ICallable是一个抽象类，定义了一个纯虚函数invoke，用于包裹可调用对象
    class ICallable;

    //CallableImpl是一个具体类，继承自ICallable，用于包裹可调用对象
    template <typename F>
    class CallableImpl : public ICallable;
    std::unique_ptr<ICallable> callable;
public:
    // 默认构造函数
    Function() noexcept = default;
    template <typename F, typename = std::enable_if_t<!std::is_same<std::decay_t<F>, Function>::value>>
    //构造函数，接受一个可调用对象F，并将其包裹在CallableImpl中,同时使用类型擦除，来进行包裹
    Function(F&& f)
        : callable(std::make_unique<CallableImpl<std::decay_t<F>>>(std::forward<F>(f))) {}

    // 移动构造和赋值
    Function(Function&& other) noexcept = default;
    Function& operator=(Function&& other) noexcept = default;

    // 拷贝构造函数
    Function(const Function& other) 
        : callable(other.callable ? other.callable->clone() : nullptr) {}
    
    // 拷贝赋值运算符
    Function& operator=(const Function& other) {
        if (this != &other) {
            callable = other.callable ? other.callable->clone() : nullptr;
        }
        return *this;
    }

    R operator()(Args... args) {
        if (!callable) {
            throw std::bad_function_call();
        }
        return callable->invoke(std::forward<Args>(args)...);
    }

    explicit operator bool() const {
        return static_cast<bool>(callable);
    }
};
```

