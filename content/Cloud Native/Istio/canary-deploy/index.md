---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "Istio Canary Deployments in kubernetes"
subtitle: ""
summary: "Istio 金丝雀部署"
authors: []
tags:
  - Istio
categories: []
date: 2024-03-08T11:27:22+08:00
lastmod: 2024-03-15T11:27:22+08:00
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

金丝雀发布（Canary Release）也被称为灰度发布，是一种在软件部署中使用的策略，旨在降低在生产环境中引入新版本软件的风险。它通过在将更改推广到整个基础架构并使其可供所有人使用之前，先缓慢地将更改推广到一小部分用户，从而实现平滑的过渡。

这种发布方式得名于矿工使用金丝雀来探测矿井中是否有有毒气体的历史实践。如果金丝雀能够存活下来，矿工就知道矿井是安全的，可以安全下井采矿。类似地，在金丝雀发布中，新版本首先被部署到一小部分用户或服务器，作为“金丝雀”来测试新版本的性能和表现。

通过这种方式，团队可以尽早发现、调整新版本可能存在的问题，确保整体系统的稳定性。如果新版本在金丝雀测试阶段表现良好，那么就可以逐步将其推广到更多的用户或服务器。如果出现问题，可以快速回退到旧版本，从而避免对整个系统造成大的影响。

金丝雀发布通常涉及对流量的精细控制，以及对日志和服务器监控的深入分析，以了解新版本的实时表现和用户反馈。这种发布方式在Kubernetes等容器中尤为常见，可以通过暂停滚动更新来实现灰度发布。

金丝雀发布是一种有效的风险管理策略，可以帮助团队更安全、更可靠地部署新版本软件。
#### 下面我来详细讲解Canary Deployments
<iframe src="https://player.bilibili.com/player.html?bvid=BV1kN411z74k" width="100%" height="500" frameborder="0" allowfullscreen="true"></iframe>
