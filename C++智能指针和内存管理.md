---
title: C++智能指针和内存管理
categories:
  - point
  - memory
tags:
  - RAII
  - shared_ptr
excerpt: 本文介绍了现在C++中的智能指针和内存管理的一些思想，包括RAII，智能指针的原理介绍等等
---
<!-- more -->

#### RAII

RAII即Resource Acquisition is Initialization（资源获取即初始化）

在智能指针中添加了引用计数的概念，引用计数这种计数是为了防止内存泄露而产生的。 基本想法是对于动态分配的对象，进行引用计数，每当增加一次对同一个对象的引用，那么引用对象的引用计数就会增加一次， 每删除一次引用，引用计数就会减一，当一个对象的引用计数减为零时，就自动删除指向的堆内存。



#### std::unique_ptr

`std::unique_ptr` 是一种智能指针，它通过指针持有并管理另一对象（对其负责），并在 `unique_ptr` 离开作用域时释放该对象。

在发生下列两者之一时，用关联的删除器释放对象：（总结就是这个`unique_ptr`不在指向这个对象）

- 管理它的 `unique_ptr` 对象被销毁。
- 通过 [operator=](https://zh.cppreference.com/w/cpp/memory/unique_ptr/operator%3D) 或 [reset()](https://zh.cppreference.com/w/cpp/memory/unique_ptr/reset) 赋值另一指针给管理它的 `unique_ptr` 对象。

```cpp
#include <cassert>
#include <cstdio>
#include <fstream>
#include <iostream>
#include <memory>
#include <stdexcept>

// 用于下面运行时多态演示的辅助类
struct B
{
    virtual ~B() = default;

    virtual void bar() { std::cout << "B::bar\n"; }
};

struct D : B
{
    D() { std::cout << "D::D\n"; }
    ~D() override{ std::cout << "D::~D\n"; }

    void bar() override { std::cout << "D::bar\n"; }
};

// 消费 unique_ptr 的函数能以值或以右值引用接收它
std::unique_ptr<D> pass_through(std::unique_ptr<D> p)
{
    p->bar();
    return p;
}

// 用于下面自定义删除器演示的辅助函数
void close_file(std::FILE* fp)
{
    std::fclose(fp);
}

// 基于 unique_ptr 的链表演示
struct List
{
    struct Node
    {
        int data;
        std::unique_ptr<Node> next;
    };

    std::unique_ptr<Node> head;

    ~List()
    {
        // 循环按顺序销毁各列表节点，默认析构函数将会递归调用其 “next” 指针的析构函数，
        // 这在足够大的链表上可能造成栈溢出。
        while (head)
        {
            auto next = std::move(head->next);
            head = std::move(next);
        }
    }

    void push(int data)
    {
        head = std::make_unique<Node>(data, std::move(head->next));
    }
};

int main()
{
    std::cout << "1) 独占所有权语义演示\n";
    {
        // 创建一个（独占）资源
        std::unique_ptr<D> p = std::make_unique<D>();

        // 转移所有权给 “pass_through”，而它再通过返回值将所有权转移回来
        std::unique_ptr<D> q = pass_through(std::move(p));

        // “p” 现在是已被移动的“空”状态，等于 nullptr
        assert(!p);
    }

    std::cout << "\n" "2) 运行时多态演示\n";
    {
        // 创建派生类资源并通过基类指向它
        std::unique_ptr<B> p = std::make_unique<D>();

        // 动态派发如期工作
        p->bar();
    }

    std::cout << "\n" "3) 自定义删除器演示\n";
    std::ofstream("demo.txt") << 'x'; // 准备要读取的文件
    {
        using unique_file_t = std::unique_ptr<std::FILE, decltype(&close_file)>;
        unique_file_t fp(std::fopen("demo.txt", "r"), &close_file);
        if (fp)
            std::cout << static_cast<char>(std::fgetc(fp.get())) << '\n';
    } // 在此调用 “close_file()”（如果 “fp” 为空）

    std::cout << "\n" "4) 自定义 lambda 表达式删除器和异常安全性演示\n";
    try
    {
        std::unique_ptr<D, void(*)(D*)> p(new D, [](D* ptr)
        {
            std::cout << "由自定义删除器销毁...\n";
            delete ptr;
        });

        throw std::runtime_error(""); // “p” 是普通指针的情况下此处就会泄漏
    }
    catch (const std::exception&)
    {
        std::cout << "捕获到异常\n";
    }

    std::cout << "\n" "5) 数组形式的 unique_ptr 演示\n";
    {
        std::unique_ptr<D[]> p(new D[3]);
    } // “D::~D()” 被调用 3 次

}
```

#### std::shared_ptr

`std::shared_ptr` 是一种智能指针，它能够记录多少个 `shared_ptr` 共同指向一个对象，从而消除显式的调用 `delete`，当引用计数变为零的时候就会将对象自动删除。



使用`make_shared<T>(T&&... args)`,通过传入一个对象（make_shared内使用了完美转发）来获取这个对象类型的`shared_ptr`指针。

`std::shared_ptr` 可以通过 `get()` 方法来获取原始指针，通过 `reset()` 来减少一个引用计数， 并通过`use_count()`来查看一个对象的引用计数。



`std::shared_ptr` 是一种通过指针保持对象共享所有权的智能指针。多个 `shared_ptr` 对象可持有同一对象。下列情况之一出现时销毁对象并解分配其内存：

- 最后剩下的持有对象的 `shared_ptr` 被销毁；
- 最后剩下的持有对象的 `shared_ptr` 被通过 [operator=](https://zh.cppreference.com/w/cpp/memory/shared_ptr/operator%3D) 或 [reset()](https://zh.cppreference.com/w/cpp/memory/shared_ptr/reset) 赋值为另一指针。



但是在下面代码，存在着内存泄漏的问题：

```cpp
#include <memory>
#include <iostream>
struct A;
struct B;

struct A {
    std::shared_ptr<B> pointer;
    ~A() {
        std::cout << "A 被销毁" << std::endl;
    }
};
struct B {
    std::shared_ptr<A> pointer;
    ~B() {
        std::cout << "B 被销毁" << std::endl;
    }
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->pointer = b;
    b->pointer = a;
}//离开作用域的时候，std::shared_ptr都会被析构，但是Control block中的引用计数不为0，导致对象没有被析构从而引发内存泄漏
```



#### std::weak_ptr

`std::weak_ptr` 是一种智能指针，它持有被 [std::shared_ptr](https://zh.cppreference.com/w/cpp/memory/shared_ptr) 管理的对象的非拥有性“弱”引用。在访问引用的对象前必须先转换为 [std::shared_ptr](https://zh.cppreference.com/w/cpp/memory/shared_ptr)。（使用lock()）

- weak_ptr不持有对象的生命周期，不算入引用计数
- 但是可以用weak_ptr判断对象是否过期

`std::weak_ptr` 实现临时所有权：当某个对象只有存在时才需要被访问，且随时可能被他人删除时，可以使用 `std::weak_ptr` 来跟踪该对象，需要获得临时所有权时，将其转换为 [std::shared_ptr](https://zh.cppreference.com/w/cpp/memory/shared_ptr)。如果此时销毁了原始 [std::shared_ptr](https://zh.cppreference.com/w/cpp/memory/shared_ptr)，则对象的生命周期将被延长，直到临时 [std::shared_ptr](https://zh.cppreference.com/w/cpp/memory/shared_ptr) 也被销毁为止。

```cpp
#include <memory>
#include <iostream>
struct A;
struct B;

struct A {
    std::shared_ptr<B> pointer;
    void print() {
        std::cout << "A" <<std::endl;
    }
    ~A() {
        std::cout << "A 被销毁" << std::endl;
    }
};
struct B {
    std::weak_ptr<A> pointer;
    void print() {
        std::cout << "B" <<std::endl;
    }
    ~B() {
        std::cout << "B 被销毁" << std::endl;
    }
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->pointer->print();
    //b->pointer->print(); error: base operand of '->' has non-pointer type 'std::weak_ptr<A>'
    b->pointer.lock()->print();
}
```



#### 引用计数的实现

在cpp中，使用`shared_ptr`的时候，编译器会创建一个Control Block，控制块的实现思想主要是通过封装 **引用计数** 和 **删除器** 来确保对象的生命周期由多个智能指针共享时，能够安全、有效地管理资源的分配和释放。

**控制块的概念**

控制块是一个管理 `shared_ptr` 引用计数和其他元数据的结构。它包含了：

- **强引用计数（use_count）**：追踪有多少个 `shared_ptr` 实例引用了这个对象。
- **弱引用计数（weak_count）**：追踪有多少个 `weak_ptr` 引用了该对象，确保只有所有强引用计数为零时，控制块才会被销毁。
- **删除器（deleter）**：负责销毁被管理的对象。当引用计数归零时，删除器会被调用来销毁对象。
- **分配器（allocator）**：如果 `shared_ptr` 使用了自定义分配器，控制块也会包含该分配器的状态。

 **控制块的结构设计**

控制块通常是一个独立的结构体或类，它和被管理对象的内存分开存储。这样设计的好处是能确保内存和对象的生命周期是独立管理的，避免了直接将对象和引用计数存在同一块内存中可能导致的效率问题。

下面是一个简单的control block的实现

```cpp
#ifndef SHARED_PTR_H
#define SHARED_PTR_H
#pragma once
#include <iostream>
#include <functional>
#include <memory>
#include <stdexcept>
// Forward declaration
template <typename T>
class shared_ptr;

struct ControlBlock {
    size_t ref_count;
    void* ptr;
    std::function<void(void*)> deleter;

    ControlBlock() : ref_count(0), ptr(nullptr),
                    deleter([](void*){}) {}

    ControlBlock(size_t ref_count, void* ptr)
        : ref_count(ref_count), ptr(ptr),
          deleter([](void* p) {}) {}

    ControlBlock(size_t ref_count, void* ptr, std::function<void(void*)> d)
        : ref_count(ref_count), ptr(ptr), deleter(std::move(d)) {}

    ~ControlBlock() = default;
};

template <typename T>
class shared_ptr {
    ControlBlock* control_block;
    T* ptr;
    void release();
public:
    // Constructors
    shared_ptr() noexcept : control_block(nullptr), ptr(nullptr) {}
    explicit shared_ptr(T* p);
    shared_ptr(const shared_ptr& other) noexcept;
    shared_ptr(shared_ptr&& other) noexcept;

    template <typename... Args>
    static shared_ptr<T> make_shared(Args&&... args);
    // Destructor
    ~shared_ptr();

    // Assignment operators
    shared_ptr& operator=(const shared_ptr& other) noexcept;
    shared_ptr& operator=(shared_ptr&& other) noexcept;

    // Pointer operations
    T& operator*() const;
    T* operator->() const;
    T* get() const noexcept;
    void reset(T* p = nullptr);
    [[nodiscard]] size_t use_count() const noexcept;

    // Type conversion
    explicit operator bool() const noexcept;

    // Swap two smart pointers
    void swap(shared_ptr& other) noexcept;

    // Comparison operations
    bool operator==(const shared_ptr& other) const noexcept;
    bool operator!=(const shared_ptr& other) const noexcept;

};

template <typename T>
shared_ptr<T>::shared_ptr(T* p) : control_block(new ControlBlock(1, p)), ptr(p) {}

template <typename T>
shared_ptr<T>::shared_ptr(const shared_ptr& other) noexcept : control_block(other.control_block), ptr(other.ptr) {
    if (control_block) {
        control_block->ref_count++;
    }
}
template <typename T>
shared_ptr<T>::shared_ptr(shared_ptr&& other) noexcept : control_block(other.control_block), ptr(other.ptr) {
    other.control_block = nullptr;
    other.ptr = nullptr;
}

template <typename T>
template <typename... Args>
shared_ptr<T> shared_ptr<T>::make_shared(Args&&... args) {
    return shared_ptr<T>(new T(std::forward<Args>(args)...));
}

template <typename T>
shared_ptr<T>::~shared_ptr() {
    release();
}

template <typename T>
void shared_ptr<T>::release() {
    if (control_block) {
        control_block->ref_count--;
        if (control_block->ref_count == 0) {
            if (ptr) {
                control_block->deleter(ptr);
            }
            delete control_block;
        }
    }
}


template <typename T>
shared_ptr<T>& shared_ptr<T>::operator=(const shared_ptr& other) noexcept {
    if (this != &other) {
        release();
        control_block = other.control_block;
        ptr = other.ptr;
        if (control_block) {
            control_block->ref_count++;
        }
    }
    return *this;
}

template <typename T>
shared_ptr<T>& shared_ptr<T>::operator=(shared_ptr&& other) noexcept {
    if (this != &other) {
        release();
        control_block = other.control_block;
        ptr = other.ptr;
        other.control_block = nullptr;
        other.ptr = nullptr;
    }
    return *this;
}

template <typename T>
T& shared_ptr<T>::operator*() const { // Dereference operator
    if (ptr == nullptr) {
        throw std::runtime_error("Attempting to dereference a null pointer");
    }
    return *ptr;
}

template <typename T>
T* shared_ptr<T>::operator->() const { // Arrow operator for accessing members
    return ptr;
}

template <typename T>
T* shared_ptr<T>::get() const noexcept {
    return ptr;
}

template <typename T>
void shared_ptr<T>::reset(T* p) {
    release();
    ptr = p;
    control_block = new ControlBlock(1, p,
        [](void* p) { delete static_cast<T*>(p); });
}

template <typename T>
size_t shared_ptr<T>::use_count() const noexcept {
    if(control_block) {
        return control_block->ref_count;
    }
    return 0;
}

template <typename T>
bool shared_ptr<T>::operator==(const shared_ptr& other) const noexcept {
    return ptr == other.ptr;
}

template <typename T>
bool shared_ptr<T>::operator!=(const shared_ptr& other) const noexcept {
    return ptr != other.ptr;
}

template <typename T>
shared_ptr<T>::operator bool() const noexcept {
    return ptr != nullptr;
}

template <typename T>
void shared_ptr<T>::swap(shared_ptr& other) noexcept {
    std::swap(control_block, other.control_block);
    std::swap(ptr, other.ptr);
}

#endif //SHARED_PTR_H
```

