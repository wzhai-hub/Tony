---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "CSS-border"
subtitle: ""
summary: "CSS元素边框"

authors: []
# tags: []
# tags:
#   - CSS
categories: []
date: 2024-03-08T11:27:22+08:00
lastmod: 2024-03-08T11:27:22+08:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

在CSS中，`border` 是用来定义元素边框的属性。元素的边框是围绕元素内容的线条，可以设置边框的样式、宽度和颜色。

`border` 属性包含了多个子属性，常见的有：

1. **border-width**: 定义边框的宽度，可以是像素、em、百分比等单位。
2. **border-style**: 定义边框的样式，例如实线、虚线、点线等。
3. **border-color**: 定义边框的颜色。
4. **border-radius**: 定义边框的圆角。

这些属性可以单独设置，也可以合并在一个 `border` 属性中，例如：

```css
.element {
  border: 2px solid #000;
}
```

上述代码表示一个具有2像素宽度、黑色实线边框的元素。你可以根据需要调整宽度、样式和颜色。
