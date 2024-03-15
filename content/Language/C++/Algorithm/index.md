---
title: C++ 标准库算法
# tags:
#   - nodejs
date: '2023-11-04'
summary: C++ 标准库提供了许多算法，这些算法位于 `<algorithm>` 头文件中。这些算法可用于各种容器，如数组、向量、列表、集合等
---

C++ 标准库提供了许多算法，这些算法位于 `<algorithm>` 头文件中。这些算法可用于各种容器，如数组、向量、列表、集合等。以下是一些常见的 C++ 算法：

1. **排序算法：**
   - `std::sort`: 对容器进行排序。
   - `std::partial_sort`: 对容器的部分范围进行排序。
   - `std::nth_element`: 找到容器中第 n 小的元素。

2. **查找算法：**
   - `std::find`: 在容器中查找指定值。
   - `std::binary_search`: 在有序容器中执行二分查找。
   - `std::lower_bound`: 返回第一个不小于指定值的元素位置。
   - `std::upper_bound`: 返回第一个大于指定值的元素位置。

3. **区间算法：**
   - `std::for_each`: 对区间中的每个元素执行指定操作。
   - `std::transform`: 对区间中的每个元素进行转换。
   - `std::accumulate`: 对区间中的元素进行累积。

4. **合并和排序算法：**
   - `std::merge`: 合并两个已排序的区间。
   - `std::inplace_merge`: 对已排序的区间进行合并。

5. **删除和替换算法：**
   - `std::remove`: 从容器中删除指定值。
   - `std::remove_if`: 从容器中删除满足条件的元素。
   - `std::replace`: 将容器中的指定值替换为另一个值。

6. **生成和填充算法：**
   - `std::generate`: 使用指定函数生成容器中的值。
   - `std::fill`: 将容器的所有元素设置为指定值。

7. **比较算法：**
   - `std::equal`: 检查两个区间是否相等。
   - `std::lexicographical_compare`: 按字典序比较两个区间。

8. **其他算法：**
   - `std::min`, `std::max`: 返回两个值中的最小值和最大值。
   - `std::rotate`: 旋转容器中的元素。
   - `std::next_permutation`, `std::prev_permutation`: 获取下一个/上一个排列。

以上只是一些常见的 C++ 算法，标准库还提供了更多的算法，可用于不同的场景和需求。算法的使用可以大大简化代码，提高程序的可读性和可维护性。