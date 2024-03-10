CSS 中的函数主要用于进行各种计算、处理或转换，以便在样式表中使用。以下是一些常见的 CSS 函数：

1. **rgb() 和 rgba()：**
   - `rgb()` 用于定义颜色的红、绿、蓝成分。
   - `rgba()` 是 `rgb()` 的扩展，允许指定颜色的透明度。

   ```css
   color: rgb(255, 0, 0);      /* 红色 */
   background-color: rgba(0, 128, 255, 0.5);  /* 半透明蓝色 */
   ```

2. **hsl() 和 hsla()：**
   - `hsl()` 用于定义颜色的色相、饱和度和亮度。
   - `hsla()` 是 `hsl()` 的扩展，允许指定颜色的透明度。

   ```css
   color: hsl(120, 100%, 50%);           /* 绿色 */
   background-color: hsla(240, 100%, 50%, 0.7);  /* 半透明紫色 */
   ```

3. **calc()：**
   - 用于执行运算，可以在长度、百分比等值之间进行计算。

   ```css
   width: calc(50% - 20px);
   ```

4. **var()：**
   - 用于使用 CSS 变量。允许在样式表中声明和使用变量。

   ```css
   :root {
     --main-color: #3498db;
   }

   .element {
     color: var(--main-color);
   }
   ```

5. **url()：**
   - 用于引用外部资源，例如图片或字体。

   ```css
   background-image: url('image.jpg');
   ```

6. **linear-gradient() 和 radial-gradient()：**
   - 用于创建线性渐变和径向渐变。

   ```css
   background: linear-gradient(to right, red, yellow);
   background: radial-gradient(circle, blue, green);
   ```

7. **min() 和 max()：**
   - 用于比较两个值，并返回其中的最小值或最大值。

   ```css
   width: min(300px, 100%);
   height: max(200px, 50%);
   ```

8. **clamp()：**
   - 在一个范围内返回一个值。

   ```css
   width: clamp(200px, 50%, 500px);
   ```

这只是一些常见的 CSS 函数，还有其他一些用于处理文本、颜色、图像等的函数。函数提供了更多的灵活性和动态性，使得在样式表中能够更好地进行计算和处理。

Transform 函数
另一个例子是 transform 的不同取值，如 rotate()。
