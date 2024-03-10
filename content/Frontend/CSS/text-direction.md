在CSS中，处理不同文本方向的属性主要涉及到 `direction` 和 `unicode-bidi` 属性。这两个属性通常用于处理文本的方向和嵌套文本块的双向性。

### 1. `direction` 属性：

`direction` 属性用于设置文本方向，可以是从左到右（`ltr`）或从右到左（`rtl`）。

- **从左到右 (`ltr`)：**
  ```css
  .left-to-right {
    direction: ltr;
  }
  ```

- **从右到左 (`rtl`)：**
  ```css
  .right-to-left {
    direction: rtl;
  }
  ```

### 2. `unicode-bidi` 属性：

`unicode-bidi` 属性用于设置文本块的双向性，即文本块的方向性如何被处理。

- **`unicode-bidi: normal`：**
  - 正常的双向性。

  ```css
  .normal-bidi {
    unicode-bidi: normal;
  }
  ```

- **`unicode-bidi: embed`：**
  - 表示文本块应该按照其自己的方向嵌套。

  ```css
  .embed-bidi {
    unicode-bidi: embed;
  }
  ```

- **`unicode-bidi: bidi-override`：**
  - 表示文本块的方向应该由其内部的 Unicode 方向字符来确定。

  ```css
  .bidi-override {
    unicode-bidi: bidi-override;
  }
  ```

### 组合使用：

通常，这两个属性会一起使用，例如在一个包含多语言文本的元素中：

```css
.multilingual {
  direction: rtl;
  unicode-bidi: embed;
}
```

这表示文本块的整体方向是从右到左，而内部文本块的双向性则由其自己的方向嵌套。

这些属性在处理多语言文本或涉及不同文本方向的情况下非常有用，以确保文本在显示时具有正确的方向性。