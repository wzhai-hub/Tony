在CSS中，属性选择器允许你根据元素的属性值选择元素。属性选择器以方括号括起来，并使用属性名称和可选的值来指定选择条件。以下是一些常见的属性选择器：

1. **存在性属性选择器（[attribute]）：**
   - 选择具有指定属性的元素，不论其值是什么。

   ```css
   /* 选择所有包含 title 属性的元素 */
   [title] {
     color: blue;
   }
   ```

2. **等值属性选择器（[attribute=value]）：**
   - 选择具有指定属性和值的元素。

   ```css
   /* 选择所有 class 属性值为 "button" 的按钮元素 */
   button[class="button"] {
     background-color: green;
   }
   ```

3. **包含值属性选择器（[attribute~=value]）：**
   - 选择具有指定属性且属性值包含指定单词的元素。

   ```css
   /* 选择所有包含 class 属性值包含 "active" 的元素 */
   .active {
     font-weight: bold;
   }
   ```

4. **开始值属性选择器（[attribute^=value]）：**
   - 选择具有指定属性值以指定值开头的元素。

   ```css
   /* 选择所有 href 属性以 "https" 开头的链接元素 */
   a[href^="https"] {
     color: green;
   }
   ```

5. **结束值属性选择器（[attribute$=value]）：**
   - 选择具有指定属性值以指定值结尾的元素。

   ```css
   /* 选择所有 src 属性以 ".jpg" 结尾的图像元素 */
   img[src$=".jpg"] {
     border: 1px solid #ccc;
   }
   ```

6. **包含子串属性选择器（[attribute*=value]）：**
   - 选择具有指定属性值包含指定子串的元素。

   ```css
   /* 选择所有 alt 属性包含 "logo" 子串的图像元素 */
   img[alt*="logo"] {
     width: 100px;
   }
   ```

属性选择器使得你可以更灵活地根据元素的属性选择并应用样式，从而实现对文档中不同元素的定制化样式。