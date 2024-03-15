---
title: Lambda 函数
# tags:
#   - nodejs
date: '2023-11-04'
summary: Lambda 函数是 C++11 引入的一项特性，它允许你在代码中创建匿名函数，即没有函数名的函数。Lambda 函数通常用于需要短暂使用的小规模操作，例如在算法中作为参数传递，或在容器中进行转换等
---

Lambda 函数是 C++11 引入的一项特性，它允许你在代码中创建匿名函数，即没有函数名的函数。Lambda 函数通常用于需要短暂使用的小规模操作，例如在算法中作为参数传递，或在容器中进行转换等。

Lambda 函数的基本语法如下：

```cpp
[capture](parameters) -> return_type {
    // 函数体
}
```

其中：
- `capture`：捕获列表，用于捕获外部变量，可以是空的 `[]`，也可以通过 `=` 表示按值捕获，或通过 `&` 表示按引用捕获。
- `parameters`：参数列表，类似于普通函数的参数列表。
- `-> return_type`：返回类型，可以省略，编译器会根据函数体中的返回语句推断返回类型。
- `{}`：函数体。

以下是一个简单的 Lambda 函数的示例：

```cpp
#include <iostream>

int main() {
    // Lambda 函数，接受两个整数参数，返回它们的和
    auto add = [](int a, int b) -> int {
        return a + b;
    };

    // 使用 Lambda 函数
    int result = add(3, 4);
    std::cout << "Result: " << result << std::endl;

    return 0;
}
```

Lambda 函数可以在函数体中访问捕获的变量，例如：

```cpp
#include <iostream>

int main() {
    int x = 5;

    // Lambda 函数，捕获外部变量 x，返回 x 的平方
    auto square = [x]() -> int {
        return x * x;
    };

    // 使用 Lambda 函数
    int result = square();
    std::cout << "Result: " << result << std::endl;

    return 0;
}
```

Lambda 函数的优点之一是它们可以轻松地与标准库中的算法一起使用，以提供更灵活的功能。例如，使用 Lambda 函数对容器中的元素进行转换：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};

    // 使用 Lambda 函数将每个元素乘以 2
    std::transform(numbers.begin(), numbers.end(), numbers.begin(),
                   [](int x) -> int { return x * 2; });

    // 输出转换后的容器
    for (int num : numbers) {
        std::cout << num << " ";
    }

    return 0;
}
```

这个例子中，`std::transform` 算法与 Lambda 函数结合，将容器中的每个元素乘以 2。Lambda 函数的简洁语法使得这种操作变得更加方便。