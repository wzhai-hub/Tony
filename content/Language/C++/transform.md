`std::transform` 是 C++ 标准库中的一个算法，位于头文件 `<algorithm>` 中。该算法用于对指定范围的元素执行某种操作，并将结果存储到另一个范围中。

`std::transform` 的基本语法如下：

```cpp
template<class InputIt, class OutputIt, class UnaryOperation>
OutputIt transform(InputIt first1, InputIt last1, OutputIt d_first, UnaryOperation unary_op);
```

其中：
- `first1` 和 `last1` 定义了输入范围，即要进行操作的元素范围。
- `d_first` 定义了输出范围，即将结果存储的目标位置。
- `unary_op` 是一个一元操作函数或函数对象，它将作用于输入范围内的每个元素，并将结果存储到输出范围。

以下是一个使用 `std::transform` 的简单示例，将一个数组中的每个元素加倍并存储到另一个数组中：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> input = {1, 2, 3, 4, 5};
    std::vector<int> output(input.size());  // 用于存储结果的目标容器

    // 使用 std::transform 将每个元素加倍并存储到输出容器中
    std::transform(input.begin(), input.end(), output.begin(),
                   [](int x) -> int { return x * 2; });

    // 输出结果容器
    for (int num : output) {
        std::cout << num << " ";
    }

    return 0;
}
```

在这个例子中，Lambda 函数 `[](int x) -> int { return x * 2; }` 作为一元操作函数传递给 `std::transform`，它将对输入范围内的每个元素执行操作（乘以2），并将结果存储到输出容器中。

`std::transform` 的灵活性使其可以与不同的操作函数结合使用，从而实现各种元素转换和操作。