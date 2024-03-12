---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "几分钟彻底理解K8S自动扩缩容"
subtitle: ""
summary: "几分钟彻底理解K8S自动扩缩容"
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

<!-- https://www.bilibili.com/video/BV1hF411y7uP/?spm_id_from=333.999.0.0&vd_source=d74f70dd1a3f3abb97c3a0481b65032c -->
<iframe src="https://player.bilibili.com/player.html?bvid=BV1hF411y7uP" width="100%" height="500" frameborder="0" allowfullscreen="true"></iframe>


Kubernetes（K8s）的自动扩缩容功能是通过Horizontal Pod Autoscaling（HPA）实现的。HPA是K8s的一种资源类型，它可以根据集群中Pod的实时负载情况，自动调整Pod的副本数量，以实现资源的弹性伸缩。

具体来说，HPA控制器会周期性地检查每个被管理的Pod的资源利用率，如CPU使用率、内存使用率等。这些检查是基于Metrics Server插件获取的容器度量数据进行的，因为API Server本身不会暴露这些度量数据。当Pod的资源使用超过设定的目标值时，HPA会按照用户定义的扩缩容策略增加Pod的副本数量，以满足业务负载上升的需求；反之，如果资源使用低于某个阈值，HPA则可能减少Pod副本数，以减少资源的浪费。

在实际操作中，当HPA决定进行扩缩容时，它会更新对应的Deployment或其它控制器的replicas字段。这一更新操作会触发实际的Pod实例创建或删除过程，使得实际运行的Pod数量与HPA计算出的目标副本数相符。

值得注意的是，HPA并不适用于所有类型的对象。例如，对于DaemonSet这类无法缩放的对象，HPA是无法起作用的。

总的来说，K8s的自动扩缩容功能通过HPA实现了对Pod资源的弹性管理，有效提高了集群的资源利用率和应用的稳定性。

<!-- 视频讲解 -->
