---
title: 在C++20的标准下实现一个线程池
categories:
  - STL
tags:
  - 设计模式
  - 并行
excerpt: 本文详细介绍了在C++20的标准下，实现了一个采用单例模式的线程池
---
<!-- more -->

线程池是提前声明好的，是一种**预先创建线程并重复利用它们**来执行多个任务的并发编程技术。他可以对线程进行准确的资源控制，同时减少线程创建销毁的开销，并且对任务进行统一的管理。本文将讲一下如何实现一个简单的线程安全的线程池。首先是前置知识的介绍

#### 线程std::thread和std::jthread

- `std：：thread`是在`C++11`中引入，一个 `std::thread` 对象**代表了一个真实运行中的线程句柄（handle）**。因此在使用thread的时候，必须要使用join（等待线程结束）或者detach（放弃对这个线程的管理）来告诉操作系统，这个线程句柄接下来要怎么处理。
- `std：：jthread`是在`C++20`中引入，在`std：：thread`的基础上，实现了`stop_token`即取消支持。同时在离开作用域的时候，无需使用`join`，`std：：jthread`会自己实现`join`，来避免忘记使用join的情况
- `std：：thread`和`std：：jthread`的共同之处：
  - 均不支持拷贝赋值和拷贝构造，但是支持移动赋值和移动构造。原因也很简单，`std：：thread`所持有的句柄是单例的，只允许移动不准拷贝。
  - 在使用时，接受[可调用对象](https://kosa-as.github.io/2025/04/23/C++%E4%B8%ADfunctional%E7%9A%84%E5%AE%9E%E7%8E%B0/#more)来调用单参构造函数，创建一个新的线程

#### 互斥量std::mutex

- `std::mutex`的引入是用来保护共享数据免受从多个线程同时访问的同步原语。在`CPP`中，常见于使用`std::lock_guard`和`std::unique_lock`,来保证临界区的正常访问。
- `std::lock_guard`: 简单固定，在创建锁的时候即立即固定，在生命周期结束的时候立即自动解锁。不提供`unlock，lock`选项，也不允许移动。
- `std：：unique_lock`:提供`lock`和`unlock`的选项，同样的不允许拷贝赋值和拷贝构造，但是可以移动。它是独占式的拥有互斥量。

#### 条件变量`std::condition_variable`

条件变量`std::condition_variable`的出现是用于唤醒等待线程从而避免死锁。如果不采用条件变量，那么在等待进入临界区的时候，使用`while(true)`检查，不仅造成了CPU资源的浪费，同时还容易造成死锁。

**在开发中，`std::condition_variable` 是与 [std::mutex](https://zh.cppreference.com/w/cpp/thread/mutex) 一起使用的同步原语，它能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（*条件*）并通知 `std::condition_variable`**。

- `cv.notify_one()`和`cv.notify_all()`：通知一个或者所有在等待的线程，检查在`wait`中的条件。

- 条件变量的等待：[cv.wait()](https://zh.cppreference.com/w/cpp/thread/condition_variable/wait)
  - `void wait(std::unique_lock<std::mutex>& lock );`:没有实现避免**虚假唤醒**（这是一定存在的，它是一种内核的行为。及没有发生notify但是却被唤醒）的方法，必须在`while`循环中使用
  
  - ```cpp
     template< class Predicate >`
     void wait( std::unique_lock<std::mutex>& lock, Predicate pred );
     //实现了避免虚假唤醒的方法，接受一个谓词类型作为参数，来判断是否发生了虚假唤醒

#### 期物`std::future`

类模板 `std::future` 提供访问异步操作结果的机制，（通过 [std::async](https://zh.cppreference.com/w/cpp/thread/async)、[std::packaged_task](https://zh.cppreference.com/w/cpp/thread/packaged_task) 或 [std::promise](https://zh.cppreference.com/w/cpp/thread/promise) 创建的）异步操作能提供一个 `std::future` 对象给该异步操作的创建者

| 特性           | `std::async`                    | `std::packaged_task`                  | `std::promise`              |
| -------------- | ------------------------------- | ------------------------------------- | --------------------------- |
| 用途           | 自动运行异步任务并返回 `future` | 封装函数任务，手动启动，返回 `future` | 显式设置结果，获取 `future` |
| 控制执行时机   | 否（由实现决定是否新线程）      | 是（手动调用）                        | 是（你决定何时设置结果）    |
| 可传入函数     | ✅ 是                            | ✅ 是                                  | ❌ 否，只设置值              |
| 多用于哪种场景 | 一次性异步任务                  | 自定义调度策略的异步任务              | 跨线程通信，或异步中断返回  |

#### 原子量`std：：atomic`

`std::atomic<T>` 提供**原子性访问**，即不会被线程调度打断，也不会产生数据竞争。它支持的操作如 `.store()`、`.load()`、`.exchange()`、`.compare_exchange_weak()` 等都能保证线程安全。

在使用`std：：atomic`的时候，默认的内存顺序是 `mememory_order_seq_cst`最强顺序，全局顺序一致性，编译器和CPU都不能重排序

| 模式      | 类比关系         | 含义                       |
| --------- | ---------------- | -------------------------- |
| `relaxed` | 自扫门前雪       | 不管别人，自己原子执行即可 |
| `release` | 发出公告         | 把前面的写操作同步给别人   |
| `acquire` | 等待公告         | 拿到别人发布的写操作       |
| `seq_cst` | 一切都按顺序排队 | 所有线程看到同样的顺序     |

#### 单例模式

单例模式是指这个类只能有一个实例存在，并且通常在工具类中使用，比如数据库链接，线程池。要实现单例模式是不是要将构造和析构函数私有，然后所有的移动，拷贝赋值构造都被delete，提供一个getinstance方法来返回全局唯一的静态的类的实例

- 懒汉式：**在第一次调用时才构造实例**；节省资源，适用于**高启动性能要求或可能从不使用的单例类**

  ```cpp
  class LazySingleton {
  public:
      static LazySingleton& getInstance() {
          static LazySingleton instance; // 局部静态变量线程安全（C++11）
          return instance;
      }
  
      LazySingleton(const LazySingleton&) = delete;
      LazySingleton& operator=(const LazySingleton&) = delete;
  
  private:
      LazySingleton() = default;
  };
  ```

- 饿汉式：**类加载时就初始化实例**，不等用的时候才创建；实例在程序开始时就存在，**一定不会为 null**；

  ```cpp
  class EagerSingleton {
  public:
      static EagerSingleton& getInstance() {
          return instance;
      }
  
      EagerSingleton(const EagerSingleton&) = delete;
      EagerSingleton& operator=(const EagerSingleton&) = delete;
  
  private:
      EagerSingleton() = default;
      static EagerSingleton instance; // 饿汉式，程序启动时创建
  };
  EagerSingleton EagerSingleton::instance; // 定义并初始化
  ```

#### 具体实现的线程池代码

```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <atomic>
#include <memory>

class SingletonThreadPool {

    std::vector<std::thread> workers;//线程池
    std::queue<std::function<void()>> tasks;//任务队列
    std::mutex queue_mutex;//入任务队列的锁
    std::condition_variable condition;//条件变量，控制任务被执行
    std::atomic<bool> stop;//线程池是否结束
    explicit SingletonThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this] {
                while (true) {//线程要一直保持活跃状态
                    std::function<void()> task;
                    {//这是使用一个代码块来保证unique_lock生命周期的正常结束
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock, [this] {//谓词函数，当线程池被停止，或者有任务的时候去执行任务
                            return this->stop.load() || !this->tasks.empty();
                        });
                        if (this->stop.load() && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());//移动所有权，调用了移动赋值函数
                        this->tasks.pop();
                    }//unique_lock会自动释放掉
                    task();//function重载了()运算符，等价于：INVOKE<R>(f, std::forward<Args>(args)...)
                }
            });//直接传入一个可调用的对象（lambda函数），使用emplace_back来直接原地构造一个thread对象
        }
    }


public:
    //单例模式不需要任何的拷贝移动构造函数
    SingletonThreadPool& operator= (const SingletonThreadPool &) = delete;
    SingletonThreadPool(const SingletonThreadPool &) = delete;
    SingletonThreadPool(SingletonThreadPool &&) = delete;
    SingletonThreadPool& operator= (SingletonThreadPool &&) = delete;
	//析构函数必须在public中声明，否则unique_ptr无法析构
    ~SingletonThreadPool() {
        stop.store(true);
        condition.notify_all();//让所有未执行完的线程全部执行完
        for (std::thread &worker : workers)//必须用&，否则会造成资源泄漏
            if (worker.joinable())
                worker.join();//回收所有的线程
    }
    
    static SingletonThreadPool* get_thread_pool(size_t threads) {
        static std::unique_ptr<SingletonThreadPool> ptr(new SingletonThreadPool(threads));
        //这里不可以使用make_unique，因为所有的构造函数都是private的，必须使用移动构造的方式赋值给unique_ptr
        return ptr.get();
    }

    template<class F, class... Args>
    auto submit(F&& f, Args&&... args) -> std::future<std::invoke_result_t<F, Args...>> {
        //invoke_result_t是对invoke(F,Args)的放回结果做预测，通过这种方式来去确定future包裹的是什么
        using return_type = std::invoke_result_t<F, Args...>;
		//使用std::packaged_task封装函数任务，来自定义获取异步调用的结果
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)//将函数和参数绑定，实现一个可调用对象
        );


        std::future<return_type> res = task->get_future();
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            if (stop.load())
                throw std::runtime_error("submit on stopped ThreadPool");
            tasks.emplace([task]() { (*task)(); });
            //lambda函数中使用参数来捕获了task，使用*task来获取包裹的可调用对象，在使用()来执行
        }
        condition.notify_one();//通知一个线程，有新的任务加入
        return res;//返回执行结果
    }
};

// ==================== 使用示例 ====================
#include <iostream>
#include <chrono>

int main() {
    auto pool = SingletonThreadPool::get_thread_pool(4);

    std::vector<std::future<int>> results;

    for (int i = 0; i < 8; ++i) {
        results.emplace_back(
            pool->submit([i] {
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
                std::cout << "Task " << i << " done\n";
                return i;
            })
        );
    }
    for (auto &&result : results)
        std::cout << result.get() << std::endl;
    return 0;
}
```

