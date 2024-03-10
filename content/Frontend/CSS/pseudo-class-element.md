CSS 中的伪类（pseudo-class）和伪元素（pseudo-element）是用于选择文档中特定状态或特定部分的选择器。它们以冒号（`:`）或双冒号（`::`）开头。虽然在一些情况下单冒号和双冒号可以互换使用，但在新的规范中，建议对于伪元素使用双冒号以区分伪类和伪元素。

### 伪类（Pseudo-class）：

1. **`:hover`：**
   - 选择器匹配用户悬停在元素上时的状态。

   ```css
   a:hover {
     color: red;
   }
   ```

2. **`:active`：**
   - 选择器匹配元素被用户激活（例如鼠标点击）时的状态。

   ```css
   button:active {
     background-color: #ccc;
   }
   ```

3. **`:focus`：**
   - 选择器匹配元素获得焦点时的状态。

   ```css
   input:focus {
     border: 2px solid blue;
   }
   ```

4. **`:first-child`：**
   - 选择器匹配是父元素的第一个子元素的元素。

   ```css
   li:first-child {
     font-weight: bold;
   }
   ```

5. **`:nth-child()`：**
   - 选择器匹配父元素中指定位置的子元素。例如，`:nth-child(odd)` 匹配父元素中的奇数位置的子元素。

   ```css
   tr:nth-child(even) {
     background-color: #f2f2f2;
   }
   ```

### 伪元素（Pseudo-element）：

1. **`::before`：**
   - 选择器匹配元素的内容之前插入的虚拟元素。常用于添加样式化的内容。

   ```css
   p::before {
     content: "前缀 ";
   }
   ```

2. **`::after`：**
   - 选择器匹配元素的内容之后插入的虚拟元素。也常用于添加样式化的内容。

   ```css
   p::after {
     content: " 后缀";
   }
   ```

3. **`::first-line`：**
   - 选择器匹配元素的第一行文本。

   ```css
   p::first-line {
     font-weight: bold;
   }
   ```

4. **`::first-letter`：**
   - 选择器匹配元素的第一个字母。

   ```css
   p::first-letter {
     font-size: 150%;
   }
   ```

总的来说，伪类用于选择元素的特定状态，而伪元素用于选择元素的特定部分或生成的内容。它们提供了更灵活的选择器，允许开发者根据元素的状态或结构应用样式。