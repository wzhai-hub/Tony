---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "kubernetes overview in 10 minutes"
subtitle: ""
summary: "10分钟彻底理解kubernetes(K8S)分布式云原生系统"

authors: [Tony zhai]
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

1. 传统裸机部署方式->虚拟机部署->容器化->容器编排[kubernetes]
2. K8S组件
   - masternode: kube-apiserver,kube-controller-manager,etcd,kube-schedular
   - worknode: kubelet,kube-proxy
3. K8S 概念有哪些呢？
   - pod:
     - ReplicaSet
     - Deployments
     - StatefulSet
     - DaemonSet
     - Job
     - CronJob
   - volume:
     - EmptyDir
     - HostPath
     - PersistentVolumeClaim(PVC)
     - PersistentVolume(PV)
     - ConfigMap
     - Secret
     - NFS(Network File System)
     - Glusterfs
     - PersistentVolume(PV)
   - svc:
     - Four types of services
       - Cluster IP
       - NodePort
       - Load Balancer
       - ExternalName
   - Headless Services
   - K8S service Discovery,通过KubeDNS（或CoreDNS）组件实现.


## 可以观看我的视频讲解《10分钟彻底理解kubernetes(K8S)分布式云原生系统》
<iframe src="https://player.bilibili.com/player.html?bvid=BV1cF411X7nV" width="100%" height="500" frameborder="0" allowfullscreen="true"></iframe>