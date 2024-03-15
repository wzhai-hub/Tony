---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "CSS-Transition"
subtitle: ""
summary: "CSS 过度"

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

在CSS中，过渡（Transition）是一种通过在状态变化时平滑地改变属性值的效果。过渡可以让元素的样式在变化时产生平滑的动画效果，而不是突变。过渡通常用于鼠标悬停、点击或其他状态变化的交互效果。

要使用过渡，你需要定义以下几个关键属性：

1. **transition-property：** 指定要过渡的 CSS 属性的名称。可以是一个或多个属性，多个属性用逗号分隔。

2. **transition-duration：** 指定过渡效果的持续时间，以秒（s）或毫秒（ms）为单位。

3. **transition-timing-function：** 指定过渡效果的时间函数，控制动画速度的变化。常见的有 `linear`（线性）、`ease`（缓入缓出，默认值）、`ease-in`（缓入）、`ease-out`（缓出）等。

4. **transition-delay：** 可选，指定过渡效果何时开始，以秒（s）或毫秒（ms）为单位。

以下是一个简单的例子，演示了当鼠标悬停在一个元素上时，背景色发生过渡效果：

```css
/* 定义过渡效果的属性 */
.box {
  width: 100px;
  height: 100px;
  background-color: blue;
  transition-property: background-color;
  transition-duration: 1s;
  transition-timing-function: ease;
}

/* 鼠标悬停时改变属性值 */
.box:hover {
  background-color: red;
}
```

在上述例子中，当鼠标悬停在 `.box` 元素上时，背景色会从蓝色平滑过渡到红色，过渡的持续时间为1秒，使用了缓入缓出的时间函数。这就是CSS过渡的基本使用方式。