#### lambda表达式

- 基础：lambda表达式的基本语法如下：

  ```c
  [捕获列表](参数列表) mutable(可选) 异常属性 -> 返回类型 {
  // 函数体
  }
  ```

- 在其中，捕获列表起到了传递外部数据的作用，根据传递的行为，捕获列表可以分为以下几种：

  - 值捕获：与参数传值类似，值捕获的前提是变量可以拷贝，不同之处则在于，被捕获的变量在 Lambda 表达式被创建时拷贝， 而非调用时才拷贝。（**在Lambda表达式被传建的时候，会对捕获的变量保存一个内部的拷贝，在修改捕获变量的值不会影响结果**）

  ```cpp
  void lambda_value_capture() {
      int value = 1;
      auto copy_value = [value] {
          return value;
      };
      value = 100;
      auto stored_value = copy_value();
      std::cout << "stored_value = " << stored_value << std::endl;
      // 这时, stored_value == 1, 而 value == 100.
      // 因为 copy_value 在创建时就保存了一份 value 的拷贝
  }
  ```

  - 引用捕获：引用捕获保存的是引用，获取最新的捕目标的值

  ```cpp
  void lambda_reference_capture() {
      int value = 1;
      auto copy_value = [&value] {
          return value;
      };
      value = 100;
      auto stored_value = copy_value();
      std::cout << "stored_value = " << stored_value << std::endl;
      // 这时, stored_value == 100, value == 100.
      // 因为 copy_value 保存的是引用
  }
  ```

  - 隐式捕获：让编译器自己去猜测要捕获的变量
    - [] 空捕获列表
    - [name1, name2, ...] 捕获一系列变量
    - [&] 引用捕获, 从函数体内的使用确定引用捕获列表
    - [=] 值捕获, 从函数体内的使用确定值捕获列表



#### 函数对象包装器

- `std::function`：`std::function` 是一种通用、多态的函数封装， 它的实例可以对任何可以调用的目标实体进行存储、复制和调用操作， 它也是对 C++ 中现有的可调用实体的一种类型安全的包裹（相对来说，函数指针的调用不是类型安全的）， 换句话说，就是函数的容器。

- `std::bind`和`std::placeholder`,`std::bind` 则是用来绑定函数调用的参数的， 它解决的需求是我们有时候可能并不一定能够一次性获得调用某个函数的全部参数，通过这个函数， 我们可以将部分调用参数提前绑定到函数身上成为一个新的对象，然后在参数齐全后，完成调用。而`std::placeholder`则是占位函数，用于表明调用时用户来提供函数

```cpp
#include <iostream>
#include <functional>

class MyClass {
public:
    void print(int x) {
        std::cout << "Value: " << x << std::endl;
    }
};

int main() {
    MyClass obj;
    
    // 绑定成员函数，需要提供实例对象
    auto boundFunc = std::bind(&MyClass::print, &obj, std::placeholders::_1);

    // 调用绑定的函数，相当于 obj.print(42)
    boundFunc(42);

    return 0;
}
```

#### 右值引用

- 左值：左值是表达式（不一定是赋值表达式）后依然存在的持久对象
- 右值：右值指表达式结束后就不再存在的临时对象
- 纯右值：纯粹的右值，要么是纯粹的字面量，例如 `10`, `true`； 要么是求值结果相当于字面量或匿名临时对象，例如 `1+2`。非引用返回的临时变量、运算表达式产生的临时变量、 原始字面量、Lambda 表达式都属于纯右值
- 将亡值：即将被销毁、却能够被移动的值

**右值引用和左值引用**:

要拿到一个将亡值，就需要用到右值引用：`T &&`，其中 `T` 是类型。 右值引用的声明让这个临时值的生命周期得以延长、只要变量还活着，那么将亡值将继续存活

```cpp
#include <iostream>
#include <string>

void reference(std::string& str) {
    std::cout << "左值" << std::endl;
}
void reference(std::string&& str) {
    std::cout << "右值" << std::endl;
}

int main()
{
    std::string lv1 = "string,"; // lv1 是一个左值
    // std::string&& r1 = lv1; // 非法, 右值引用不能引用左值
    std::string&& rv1 = std::move(lv1); // 合法, std::move可以将左值转移为右值
    std::cout << rv1 << std::endl; // string,

    const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
    // lv2 += "Test"; // 非法, 常量引用无法被修改
    std::cout << lv2 << std::endl; // string,string,

    std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
    rv2 += "Test"; // 合法, 非常量引用能够修改临时变量
    std::cout << rv2 << std::endl; // string,string,string,Test

    reference(rv2); // 输出左值

    return 0;
}
```



#### 完美转发

在作参数转发的时候，保证左值和右值可以被正确的传递

```cpp
#include <iostream>
#include <utility>
void reference(int& v) {
    std::cout << "左值引用" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值引用" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "              普通传参: ";
    reference(v);
    std::cout << "       std::move 传参: ";
    reference(std::move(v));
    std::cout << "    std::forward 传参: ";
    reference(std::forward<T>(v));
    std::cout << "static_cast<T&&> 传参: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1);

    std::cout << "传递左值:" << std::endl;
    int v = 1;
    pass(v);

    return 0;
}
```

无论传递参数为左值还是右值，普通传参都会将参数作为左值进行转发； 由于类似的原因，`std::move` 总会接受到一个左值，从而转发调用了`reference(int&&)` 输出右值引用。

唯独 `std::forward` 即没有造成任何多余的拷贝，同时**完美转发**(传递)了函数的实参给了内部调用的其他函数