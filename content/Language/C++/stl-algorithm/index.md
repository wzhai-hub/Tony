---
title: C++ 标准库提供了丰富的算法集合
# tags:
#   - nodejs
date: '2023-11-04'
summary: C++ 标准库提供了丰富的算法集合，这些算法位于 `<algorithm>` 头文件中。这些算法用于在各种容器（如数组、向量、列表、映射等）上执行各种操作，包括搜索、排序、变换等
---
## 算法
C++ 标准库提供了丰富的算法集合，这些算法位于 `<algorithm>` 头文件中。这些算法用于在各种容器（如数组、向量、列表、映射等）上执行各种操作，包括搜索、排序、变换等。以下是一些常见的 C++ 算法：

1. **非修改序列操作：**
   - `std::all_of`, `std::any_of`, `std::none_of`：检查条件是否对序列中的所有、任何或没有元素成立。
   - `std::for_each`：对序列中的每个元素执行指定操作。
   - `std::count`, `std::count_if`：计算序列中等于某个值或满足某个条件的元素的个数。
   - `std::accumulate`：计算序列中元素的累积和。

2. **修改序列操作：**
   - `std::copy`, `std::copy_if`, `std::copy_n`：将序列的元素复制到另一个序列。
   - `std::transform`：对序列中的每个元素应用给定的函数，并将结果存储到另一个序列。
   - `std::fill`, `std::fill_n`：用指定的值填充序列或一部分序列。
   - `std::replace`, `std::replace_if`：替换序列中等于某个值或满足某个条件的元素。

3. **删除和调整序列操作：**
   - `std::remove`, `std::remove_if`：从序列中删除等于某个值或满足某个条件的元素。
   - `std::unique`：从已排序的序列中删除重复的元素。
   - `std::reverse`：反转序列中的元素。
   - `std::rotate`：将序列中的元素循环移动到指定位置。

4. **排序和搜索操作：**
   - `std::sort`, `std::partial_sort`, `std::stable_sort`：对序列进行排序。
   - `std::binary_search`, `std::lower_bound`, `std::upper_bound`：在有序序列中进行搜索。
   - `std::max_element`, `std::min_element`：找到序列中的最大和最小元素。

5. **生成和排列操作：**
   - `std::generate`, `std::generate_n`：生成序列中的元素。
   - `std::next_permutation`, `std::prev_permutation`：生成下一个或上一个排列。

这只是 `<algorithm>` 头文件中一小部分算法的示例。这些算法提供了一组强大而灵活的工具，可用于在各种情况下高效地操作和处理数据。