---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "K8S Overview"
subtitle: ""
summary: ""
authors: []
tags: []
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

## K8S的出现
Kubernetes，也称为K8s，是一款开源的容器编排系统，用于自动化部署、扩展和管理容器化应用程序。以下是Kubernetes的一些主要特性：

传统的服务部署方式逐渐发展到现代的容器编排系统。以下是K8S演进过程的主要阶段：

1. **传统的服务部署**：
   - 在早期，应用程序通常直接部署在物理机上。这种方式导致资源分配不均、维护困难，以及扩展性差等问题。

2. **虚拟机部署**：
   - 为了解决物理机上的资源分配问题，虚拟化技术被引入。虚拟机（VM）允许在一台物理机上运行多个独立的操作系统实例，从而提高了资源的利用率和灵活性。如KVM。
   - 然而，虚拟机仍然面临一些问题，如启动时间长、资源消耗大等。

3. **容器部署**：
   - 随着Docker等容器技术的兴起，应用程序及其依赖被打包成轻量级的容器镜像，可以快速部署和迁移。
   - 容器提供了更好的资源隔离和更小的资源消耗，使得应用程序的部署和管理变得更加高效。
   - Docker等容器技术使得开发者能够轻松地将应用程序及其环境打包为一个可移植的单元，从而简化了部署过程。

4. **K8S的出现**：
   - 尽管容器技术解决了许多部署问题，但随着容器数量的增加，管理和编排这些容器变得越来越复杂。
   - Kubernetes（K8S）应运而生，它是一个开源的容器编排系统，用于自动化部署、扩展和管理容器化应用程序。
   - K8S提供了丰富的功能，如自我修复、弹性伸缩、服务发现和负载均衡等，使得容器化应用程序的部署和管理变得更加简单和高效。

## K8S 有哪些特性

1. **弹性伸缩**：Kubernetes支持水平和垂直的自动扩展，根据应用程序的负载情况进行弹性调整，确保资源利用效率和应用程序的高可用性。水平扩展功能允许通过简单命令或UI手动扩展，同时也支持基于CPU等资源负载率的自动水平扩展机制。
2. **服务发现与负载均衡**：Kubernetes提供了内建的DNS服务来实现服务发现，并通过负载均衡机制将流量分配到相应的后端服务上。这主要通过KubeDNS（或CoreDNS）组件实现，为每个Service配置DNS名称，允许集群内的客户端直接使用此名称发出访问请求，而Service则通过iptables或ipvs进行负载均衡。
3. **自我修复**：Kubernetes具备自我修复能力，当容器或节点出现故障时，会自动重新启动容器或替换故障节点，确保应用程序持续可用。这种自愈机制还包括容器故障后自动重启、节点故障后重新调度容器，以及其他可用节点、健康状态检査失败后关闭容器并重新创建等功能。
4. **自动发布和回滚**：Kubernetes支持“灰度”更新应用程序或其配置信息，并会监控更新过程中应用程序的健康状态。如果在更新过程中出现故障，它会立即自动执行回滚操作。
5. **安全性**：Kubernetes提供了多种安全机制，如访问控制、身份验证、密钥管理等，保护容器和集群的安全。

此外，Kubernetes集群中的节点（Node）是工作负载节点，每个Node会被Master分配一些工作负载，当某一个Node宕机时，其上的工作负载会被Master自动转移到其他节点上。关键进程如kubelet负责Pod对应的容器的创建、启动、停止等任务，并与Master节点密切协作，实现集群管理的基本功能；而kube-proxy则是实现Kubernetes Service的通信与负载均衡机制的重要组件。

<iframe width="560" height="315" src="https://www.bilibili.com/video/BV1XJ411X7Ud/?from=search&seid=4871020555062147773" frameborder="0" allowfullscreen></iframe>

<!-- 视频衔接 -->
[详细讲解视频](https://www.bilibili.com/video/BV1XJ411X7Ud/?from=search&seid=4871020555062147773)

