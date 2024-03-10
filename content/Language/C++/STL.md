C++ 标准库（Standard Library）提供了许多库函数，涵盖了广泛的功能领域。这些库函数分为几个主要的头文件，并包括各种类、函数和算法。以下是一些常见的 C++ 库函数的主要类别和示例：

1. **输入输出库 (`<iostream>`、`<iomanip>`、`<fstream>`等)：**
   - `std::cin`, `std::cout`, `std::cerr` 等标准输入输出流。
   - `std::ifstream`, `std::ofstream` 用于文件输入输出。
   - 输入输出流相关的格式化和控制符，如 `std::setw`, `std::setprecision`。

2. **容器库 (`<vector>`、`<list>`、`<map>`、`<set>`等)：**
   - `std::vector`, `std::list`, `std::map`, `std::set` 等容器类。
   - 各种算法，如 `std::sort`, `std::find`, `std::transform`，位于 `<algorithm>` 头文件中。

3. **字符串处理库 (`<string>`、`<cstring>`等)：**
   - `std::string` 类及其相关操作，如拼接、查找、替换等。
   - 字符串处理函数，如 `std::strlen`, `std::strcpy`，位于 `<cstring>` 头文件中。

4. **数学库 (`<cmath>`等)：**
   - `std::abs`, `std::sqrt`, `std::sin`, `std::cos` 等数学函数。
   - 位于 `<cmath>` 头文件中，包含了数学函数的声明。

5. **时间和日期库 (`<chrono>`、`<ctime>`等)：**
   - `std::chrono` 提供了时间和时钟相关的功能。
   - 时间和日期处理函数，如 `std::time`, `std::localtime`，位于 `<ctime>` 头文件中。

6. **异常处理库 (`<stdexcept>`等)：**
   - 异常类，如 `std::exception`, `std::runtime_error`, `std::logic_error`。
   - 异常处理相关函数，如 `std::try`, `std::catch`，位于 `<stdexcept>` 头文件中。

7. **文件系统库 (`<filesystem>`等)：**
   - `std::filesystem` 提供了对文件系统的访问和操作。
   - 位于 `<filesystem>` 头文件中，包含了文件系统相关的类和函数。

8. **智能指针库 (`<memory>`等)：**
   - `std::shared_ptr`, `std::unique_ptr`, `std::weak_ptr` 提供了智能指针的支持。
   - 位于 `<memory>` 头文件中，包含了与智能指针相关的类和函数。

9. **多线程库 (`<thread>`、`<mutex>`等)：**
   - `std::thread` 用于创建和管理线程。
   - 同步和互斥相关类，如 `std::mutex`, `std::condition_variable`。
   - 位于 `<thread>` 和 `<mutex>` 头文件中。

10. **正则表达式库 (`<regex>`等)：**
    - `std::regex` 用于支持正则表达式的操作。
    - 位于 `<regex>` 头文件中。

这只是 C++ 标准库中的一部分库函数，还有其他头文件和类提供了更多的功能。标准库的丰富性使得 C++ 程序员能够方便地使用各种功能而无需重新实现。