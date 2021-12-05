---
title: Docker原理全面概览
categories:
  - 容器
tags:
  - docker
date: 2021-12-05 11:26:26
---

> docker天天用，原理却不清楚。当然，作为一个应用开发人员，容器做到会用，多数时候已经足够了，但如果想要进一步了解，或者对云原生有所兴趣，对docker原理的了解显得必不可少。
>
> 在介绍容器技术时，我们通常会以实体机->虚拟机->容器的顺序介绍虚拟化技术的演进，然而容器到底比虚拟机轻量在哪里，这是个值得深入探究的问题。
>
> 本文是基于《Docker容器与容器云（第二版）》一书而来，该书出版于2016年，docker处于1.10版本，现在是2021年，docker处于[20.10.11版本](https://docs.docker.com/engine/release-notes/)，架构上发生了较大的变化（主要体现在libcontainer变为containerd上），且kubernetes已经宣布将不再在kubelet中原生支持docker。因此文章部分内容和书籍会有较大差异。
>
> 又，看底层原理还是需要对Linux有较深刻的了解，而我的Linux知识还很匮乏，只是三三两两地知道一些知识点，不成系统，因此无法从很深层面讲解，这也是本文称之为“概览”的原因。

<!-- more -->

在探索Docker时，始终牢记以下几个重点，将非常有帮助

- Docker是go语言写的，其放在github上的仓库，不叫Docker，而叫Moby
- Docker是CS架构，服务端通过Restful接口暴露一个API Server，客户端将命名翻译成请求发送到服务端，由服务端调用相关模块执行具体操作，如镜像构建、创建容器等
- Docker的特色，也是Docker为K8s嫌弃的地方，在于Docker Daemon，它提供了对用户交互的支持，但是在K8s中这个组件却显得多余
- Docker有三个核心：使用namespace与宿主机进行资源隔离；使用cgroup进行资源限制；基于写时复制的文件系统完成高效读写
- Docker服务端是高度解耦的，每个功能都由单独的模块完成，如镜像管理、卷管理、容器管理、网络管理等

## Docker架构

![image-20211205114819813](https://gdz.oss-cn-shenzhen.aliyuncs.com/local/image-20211205114819813.png)

## 镜像管理



## 存储管理



## 数据卷




## 网络管理



## libcontainer和containerd



## 结尾

