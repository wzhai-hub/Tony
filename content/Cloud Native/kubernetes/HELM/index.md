---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "K8S package tool HELM introduce"
subtitle: ""
summary: "理解kubernetes 应用部署管理软件Helm功能"
authors: []
# tags: []
tags:
  - kubernetes
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

## K8s应用程序部署的打包工具Helm 概述介绍
- Helm file format: YAML/GO 模板函数
- Helm是Kubernetes生态系统中的一个重要工具，主要用于管理Kubernetes应用资源。它类似于Ubuntu的apt、CentOS的yum或Python的pip，专门用于对Kubernetes应用进行统一打包、分发、安装、升级以及回退等操作。

- Helm包括三个基本概念：Chart、Repository和Release。
  - Chart：Helm的应用包，它包括了应用的所有Kubernetes manifest模版，类似于YUM RPM或Apt dpkg文件。Chart封装了Kubernetes原生应用程序的一系列yaml文件,可以在部署应用时自定义应用程序的一些元数据，以便于应用程序的分发。
  - Repository：Helm包的存储仓库，类似于软件包的仓库或存储中心，用于存储和检索Helm Chart。
  - Release：Chart的部署实例，每个Chart可以部署一个或多个release。

Helm是Kubernetes应用中不可或缺的一部分，它大大简化了Kubernetes应用的管理和部署过程，提高了应用的稳定性和可靠性。


<iframe src="https://player.bilibili.com/player.html?bvid=BV1tm4y177MC" width="100%" height="500" frameborder="0" allowfullscreen="true"></iframe>

