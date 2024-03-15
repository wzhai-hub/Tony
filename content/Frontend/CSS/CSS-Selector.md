---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "CSS-selector"
subtitle: ""
summary: "CSS选择器"

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

CSS（层叠样式表）选择器用于选择 HTML 或 XML 文档中的元素，并为其应用样式。以下是一些常见的 CSS 选择器：

1. **元素选择器（Element Selector）：**
   - 选择特定的 HTML 元素。例如，`p` 选择所有段落元素。

   ```css
   p {
     /* 样式规则 */
   }
   ```

2. **类选择器（Class Selector）：**
   - 选择具有特定类的元素。以点（.）开头，后面跟类名。例如，`.example` 选择所有具有类名为 "example" 的元素。

   ```css
   .example {
     /* 样式规则 */
   }
   ```

3. **ID 选择器（ID Selector）：**
   - 选择具有特定 ID 的元素。以井号（#）开头，后面跟 ID 名。注意，ID 在文档中应唯一。例如，`#header` 选择 ID 为 "header" 的元素。

   ```css
   #header {
     /* 样式规则 */
   }
   ```

4. **后代选择器（Descendant Selector）：**
   - 选择特定元素的后代元素。使用空格分隔元素名称。例如，`article p` 选择所有在 `<article>` 元素内的段落元素。

   ```css
   article p {
     /* 样式规则 */
   }
   ```

5. **子元素选择器（Child Selector）：**
   - 选择特定元素的直接子元素。使用大于号（>）分隔元素名称。例如，`ul > li` 选择所有直接在 `<ul>` 元素下的 `<li>` 元素。

   ```css
   ul > li {
     /* 样式规则 */
   }
   ```

6. **相邻兄弟选择器（Adjacent Sibling Selector）：**
   - 选择紧接在另一个元素后的元素。使用加号（+）分隔元素名称。例如，`h2 + p` 选择紧接在 `<h2>` 元素后的 `<p>` 元素。

   ```css
   h2 + p {
     /* 样式规则 */
   }
   ```

7. **通用选择器（Universal Selector）：**
   - 选择所有元素。使用星号（*）。例如，`*` 选择文档中的所有元素。

   ```css
   * {
     /* 样式规则 */
   }
   ```

8. **属性选择器（Attribute Selector）：**
   - 选择具有特定属性的元素。例如，`[type="text"]` 选择所有 `type` 属性为 "text" 的元素。

   ```css
   [type="text"] {
     /* 样式规则 */
   }
   ```

这只是 CSS 选择器的一小部分，还有更多高级和复杂的选择器，允许更灵活和精确地选择目标元素。