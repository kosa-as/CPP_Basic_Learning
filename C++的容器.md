#### 1.线性容器

- `std::array`:稍微封装过的数组，使用时必须指定大小
- `std::vector`：可以自动扩容的线性容器
- `std::list`：双向链表
- `std::forward_list`：单向链表



#### 2.非线性容器

- `std::set/std::map`:内部使用红黑树实现，插入和搜索的平均时间复杂度为o(logn),遍历的时候仍然可以保留原有的顺序
- `std::unordered_set/std::unordered_map`:内部使用hash实现的，插入和搜索的平均复杂度为o(n)，在遍历的时候不会保留原有的顺序

#### 3.元组

