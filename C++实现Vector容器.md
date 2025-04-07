---
title: 在C++20的标准下实现自己的vector容器
categories:
  - STL
tags:
  - Template
excerpt: 本文详细介绍了在C++20的标准下，仿照`std::vector`实现了一个自己的vector
---
<!-- more -->

首先介绍一下`vector`的基本功能：`vector`是一个线性容器，他是一种**动态数组**，其内存管理是动态的，但元素的访问方式类似于静态数组（即通过索引直接访问）

#### 相关知识介绍：

- `std::allocator`: 在所有的标准stl容器中，都是使用`allocator`来管理内存，这种管理方式只会申请**原始内存**，即没有初始化。（不同于new，申请内存的同时会在堆上构建 ==不使用new的原因是默认的new效率更低，同时使用allocator可以更好的控制对象的生命周期，允许在已分配的内存中按需构造析构对象==）。同时使用`std::allocator_traits<std::allocator<T>>::construct`和`std::allocator_traits<std::allocator<T>>::destory`来显示的构造或者析构对象。
- 类的基本知识：在C++ 11之后，新建一个类，哪怕没有声明任何一个函数，编译器会默认实现六个函数，分别是：默认构造函数，默认析构函数，拷贝构造函数，移动构造函数，拷贝赋值函数，移动赋值函数。
- 迭代器：在类中，再去定义一个迭代器类，来封装一个容器内对象类型的指针，同时重载一些运算符的实现
- 动态内存：vector在堆上新申请一片内存，当这个内存被用满之后，那么vector就要重新去申请一片内存，然后把之前的元素全部放入。

```cpp
#include <memory>
#include <stdexcept>
#include <initializer_list>
#include <algorithm>
#include <iostream>
template <typename T>
class Vector {
private:
    T* data;
    size_t vec_size;
    size_t vec_capacity;
    std::allocator<T> allocator;
    // 扩容函数声明
    void reserve(size_t new_capacity);
    // 快速排序函数声明
    template<typename Compare>
    void quick_sort(typename Vector<T>::iterator first, typename Vector<T>::iterator last, Compare cmp);
public:
    class iterator {
        T* ptr;
        public:
        explicit iterator(T* ptr) : ptr(ptr) {}
        T& operator*() const { return *ptr; }//解引用操作符
        T* operator->() const { return ptr; }//成员访问操作符
        iterator& operator++() { ++ptr; return *this; }//前缀加
        iterator operator++(int) { iterator tmp = *this; ++(*this); return tmp; }//后缀加
        iterator& operator--() { --ptr; return *this; }//前缀减
        iterator operator--(int) { iterator tmp = *this; --(*this); return tmp; }//后缀减
        bool operator==(const iterator& other) const { return ptr == other.ptr; }
        bool operator!=(const iterator& other) const { return ptr != other.ptr; }
        iterator operator+(size_t n) const { return iterator(ptr + n); }
        iterator operator-(size_t n) const { return iterator(ptr - n); }

    };
    class const_iterator : public iterator {
        public:
        explicit const_iterator(const T* ptr) : iterator(ptr) {}
        const T& operator*() const { return iterator::operator*(); }
        const T* operator->() const { return iterator::operator->(); }
    };
    // 构造函数声明
    Vector() noexcept;
    explicit Vector(size_t n);
    Vector(size_t n, const T& val);
    Vector(std::initializer_list<T> init);
    Vector(iterator begin, iterator end);
    Vector(const Vector<T>& other);
    Vector(Vector<T>&& other) noexcept;
    // 赋值运算符声明
    Vector& operator=(const Vector<T>& other);
    Vector& operator=(Vector<T>&& other) noexcept;
    // 析构函数声明
    ~Vector();
    // 迭代器函数声明
    iterator begin();
    const_iterator const_begin() const;
    iterator end();
    const_iterator const_end() const;
    // 访问元素函数声明
    T& operator[](size_t index);
    T& at(size_t index);
    // 容量和大小函数声明
    [[nodiscard]] size_t capacity() const;
    void set_capacity(size_t new_capacity);
    [[nodiscard]] bool empty() const;
    [[nodiscard]] size_t size() const;
    // 清空函数声明
    void clear();
    // 添加元素函数声明
    void push_back(const T& value);
    // 添加元素函数声明,移动添加
    void push_back(T&& value);
    // 添加元素函数声明，直接构造
    template<typename... Args>
    void emplace_back(Args&&... args);
    // 查找元素函数声明
    const_iterator find(const T& value) const;
    // 插入元素函数声明
    void insert(iterator pos, const T& value);
    // 删除元素函数声明
    void erase(iterator pos);
    void pop_back();
    // 访问首尾元素函数声明
    T& front() const;
    T& back() const;
    // 交换函数声明
    void swap(Vector& other);
    // 友元函数声明
    template <typename U>
    friend std::ostream& operator<<(std::ostream& os, const Vector<U>& vec);
    // 快速排序函数声明
    template<typename Compare = std::less<T>>
    void sort(iterator first, iterator last, Compare cmp = Compare());
    // 函数重写
    template<typename Compare = std::less<T>>
    void sort(Compare cmp = Compare());
};
```

#### vector的私有属性

```cpp
    T* data;//指向管理的数据的指针
    size_t vec_size;//容器内数据的数量
    size_t vec_capacity;//分配内容可容纳的对象的数量
    std::allocator<T> allocator;//容器内存管理器
```

#### vector的迭代器

```cpp
    class iterator {
        T* ptr;
        public:
        explicit iterator(T* ptr) : ptr(ptr) {}
        T& operator*() const { return *ptr; }
        T* operator->() const { return ptr; }
        iterator& operator++() { ++ptr; return *this; }//前缀加
        iterator operator++(int) { iterator tmp = *this; ++(*this); return tmp; }//后缀加
        iterator& operator--() { --ptr; return *this; }//前缀减
        iterator operator--(int) { iterator tmp = *this; --(*this); return tmp; }//后缀减
        bool operator==(const iterator& other) const { return ptr == other.ptr; }
        bool operator!=(const iterator& other) const { return ptr != other.ptr; }
        iterator operator+(size_t n) const { return iterator(ptr + n); }
        iterator operator-(size_t n) const { return iterator(ptr - n); }

    };
    class const_iterator : public iterator {
        public:
        explicit const_iterator(const T* ptr) : iterator(ptr) {}
        const T& operator*() const { return iterator::operator*(); }
        const T* operator->() const { return iterator::operator->(); }
    };
```

在C++11之后，除了`iterator`之后，还实现了`const_iterator`。迭代器内实现了各种常见指针操作的重载

#### vector的扩容操作

在执行push_back，emplace_back, insert等操作后，vector可能需要扩容，此时需要调用reserve函数去堆上申请一块新的内存空间。在vector扩容的时候，通常扩充为之前的1.5~2倍大小

```cpp
template <typename T>
void Vector<T>::set_capacity(size_t new_capacity) {
    if (new_capacity <= vec_capacity) return;  // 只在需要扩容时执行
    reserve(new_capacity);
}
// 扩容函数实现
template <typename T>
void Vector<T>::reserve(size_t new_capacity) {
    if (new_capacity <= vec_capacity) return;  // 只在需要扩容时执行

    T* new_data = std::allocator_traits<std::allocator<T>>::allocate(allocator, new_capacity);
    // 移动现有元素
    for (size_t i = 0; i < vec_size; i++) {
        std::allocator_traits<std::allocator<T>>::construct(allocator, &new_data[i], std::move(data[i]));
        std::allocator_traits<std::allocator<T>>::destroy(allocator, &data[i]);
    }
    // 释放旧内存
    if (data) {
        std::allocator_traits<std::allocator<T>>::deallocate(allocator, data, vec_capacity);
    }
    data = new_data;
    vec_capacity = new_capacity;
}
```

#### vector的构造函数

```cpp
Vector() noexcept;
explicit Vector(size_t n);
Vector(size_t n, const T& val);
Vector(std::initializer_list<T> init);
Vector(iterator begin, iterator end);
Vector(const Vector<T>& other);
Vector(Vector<T>&& other) noexcept;
```

- 首先是默认构造`Vector() noexcept`, noexcept表示不会抛出异常
- 然后是单参构造函数`explicit Vector(size_t n)`,在C++11之前，对于单参构造函数，都要用`explicit`来避免隐式转化。在C++11之后，也可以堆多参构造函数使用

- 拷贝构造函数`Vector(const Vector<T>& other);`

  ```cpp
  template <typename T>
  Vector<T>::Vector(const Vector<T>& other) : data(nullptr), vec_size(0), vec_capacity(0) {
      reserve(other.vec_size);
      for (size_t i = 0; i < other.vec_size; ++i) {
          std::allocator_traits<std::allocator<T>>::construct(allocator, &data[i], other.data[i]);
      }
      vec_size = other.vec_size;
  }
  ```

- 移动构造函数`Vector<T>::Vector(Vector<T>&& other) noexcept :`

  ```cpp
  // 移动构造函数实现
  template <typename T>
  Vector<T>::Vector(Vector<T>&& other) noexcept : 
      data(other.data),
      vec_size(other.vec_size),
      vec_capacity(other.vec_capacity),
      allocator(std::move(other.allocator)) {
      // 防止 other 析构时释放我们刚"偷"来的内存
      other.data = nullptr;
      other.vec_size = 0;
      other.vec_capacity = 0;
  }
  ```

- 初始化列表构造 `Vector(std::initializer_list<T> init);`:

  ```cpp
  // 初始化列表构造函数实现
  template <typename T>
  Vector<T>::Vector(std::initializer_list<T> init) : data(nullptr), vec_size(0), vec_capacity(0) {
      if (init.size() > 0) {
          reserve(init.size());
          for (size_t i = 0; i < init.size(); ++i) {
              std::allocator_traits<std::allocator<T>>::construct(allocator, &data[i], *(init.begin() + i));
          }
          vec_size = init.size();
      }
  }
  ```

- 迭代器构造函数`Vector(iterator begin, iterator end);`

  ```cpp
  // 迭代器构造函数实现
  template <typename T>
  Vector<T>::Vector(iterator begin, iterator end) : data(nullptr), vec_size(0), vec_capacity(0) {
      size_t count = end - begin;
      reserve(count);
      for (size_t i = 0; i < count; ++i) {
          std::allocator_traits<std::allocator<T>>::construct(allocator, &data[i], *(begin + i));
      }
      vec_size = count;
  }
  ```

#### vector的析构函数

调用`clear()`函数，清空`allocator`分配的内存

```cpp
// 析构函数实现
template <typename T>
Vector<T>::~Vector() {
    clear();
}
// 清空函数实现
template <typename T>
void Vector<T>::clear() {
    for (size_t i = 0; i < vec_size; i++) {
        std::allocator_traits<std::allocator<T>>::destroy(allocator, &data[i]);
    }
    if (data) {
        std::allocator_traits<std::allocator<T>>::deallocate(allocator, data, vec_capacity);
    }
    data = nullptr;
    vec_size = 0;
    vec_capacity = 0;
}
```

#### vector添加元素的操作

- `push_back`:有两种方法，一种是拷贝构造，一种是移动构造

  ```cpp
  template <typename T>
  void Vector<T>::push_back(const T& value) {//拷贝原对象
      if (vec_size == vec_capacity) {
          size_t new_capacity = (vec_capacity == 0) ? 1 : 2 * vec_capacity;
          reserve(new_capacity);
      }
      std::allocator_traits<std::allocator<T>>::construct(allocator, data+vec_size, value);
      ++vec_size;
  }
  
  template <typename T>
  void Vector<T>::push_back(T&& value) {//利用右值直接构造
      if (vec_size == vec_capacity) {
          size_t new_capacity = (vec_capacity == 0) ? 1 : 2 * vec_capacity;
          reserve(new_capacity);
      }
      std::allocator_traits<std::allocator<T>>::construct(allocator, data+vec_size, std::move(value));
      ++vec_size;
  }
  ```

- `emplace_back`

  ```cpp
  // 添加元素函数实现，直接构造
  template <typename T>
  template<typename... Args>
  void Vector<T>::emplace_back(Args&&... args) {
      if (vec_size == vec_capacity) {
          size_t new_capacity = (vec_capacity == 0) ? 1 : 2 * vec_capacity;
          reserve(new_capacity);
      }
      
      // 使用完美转发构造新元素
      std::allocator_traits<std::allocator<T>>::construct(
          allocator, 
          data + vec_size, 
          std::forward<Args>(args)...
      );
      
      ++vec_size;
  }
  ```

- `insert`：利用迭代器实现了元素的定点插入

  ```cpp
  // 插入元素函数实现
  template <typename T>
  void Vector<T>::insert(iterator pos, const T& value) {
      size_t index = pos - begin();
      if (vec_size == vec_capacity) {
          size_t new_capacity = (vec_capacity == 0) ? 1 : 2 * vec_capacity;
          reserve(new_capacity);
      }
      for (size_t i = vec_size; i > index; --i) {
          std::allocator_traits<std::allocator<T>>::construct(allocator, &data[i], std::move(data[i-1]));
      }
      std::allocator_traits<std::allocator<T>>::construct(allocator, &data[index], value);
      ++vec_size;
  }
  ```

#### vector中排序的使用

这里使用了比较器模板，在 C++ 中，**比较器模板**允许你为算法（如排序、查找、优先级队列等）提供自定义的比较规则，同时保持代码的泛用性和高性能。以下是实现比较器模板的详细方法及示例：

比较器可以是 **函数指针**、**函数对象（仿函数）** 或 **Lambda 表达式**。它们需要满足以下条件：

- 接受两个相同类型的参数。
- 返回 `bool` 类型，表示两个元素的顺序关系。

这里在类中比较器模板的申明中，指定了`Compare = std::less<T>`

```cpp

// 快速排序函数实现
template <typename T>
template <typename Compare>
void Vector<T>::quick_sort(typename Vector<T>::iterator first, typename Vector<T>::iterator last, Compare cmp) {
    if (first >= last) return;
    // 选择基准元素（此处选择中间元素）
    T pivot = *(first + (last - first) / 2);
    iterator left = first;
    iterator right = last - 1;

    // 分区操作
    while (left <= right) {
        while (cmp(*left, pivot)) ++left;
        while (cmp(pivot, *right)) --right;
        if (left <= right) {
            std::swap(*left, *right);
            ++left;
            --right;
        }
    }
    // 递归排序
    quick_sort(first, right + 1, cmp);
    quick_sort(left, last, cmp);
}
// 快速排序函数实现
template <typename T>
template <typename Compare>
void Vector<T>::sort(iterator first, iterator last, Compare cmp) {
    quick_sort(first, last, cmp);
}

template <typename T>
template <typename Compare>
void Vector<T>::sort(Compare cmp) {
    quick_sort(begin(), end(), cmp);
}
```



​	
