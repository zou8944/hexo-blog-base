---
title: Kubernetes节点问题排查思路
categories:
  - 运维
tags:
  - Kubernetes
date: 2022-02-17 12:07:46
---

Kubernetes集群运行一段时间后，通常会出现磁盘占用过高、内存占用虚高等问题。这里聊聊排查这类问题的方法和思路。

<!-- more -->

## 问题点

如果是正常的Linux服务器出现这种情况，解决方案无非是ssh到该主机查看大文件然后进行删除。Kubernetes上也是如此，但问题在于，其中的节点由Kubernetes自己管理，登录秘钥管理人员很可能不知道。

所以问题点转化：无法进入Kubernetes节点进行管理

## 进入节点

有两个进入节点的方式

- 使用Lens管理软件，它是一个非常厉害的Kubernetes IDE，提供直接进入节点的能力。
- 自建pod，将节点根目录挂载在该pod的子目录。即挂载整个文件系统

第二种方式比较通用，这类deployment的Manifest如下，关键点：节点亲和性设置到想要管理的节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: admin-entry
  name: admin-entry
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin-entry
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: admin-entry
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - cn-shenzhen.172.18.18.192
      containers:
        - image: 'busybox:latest'
          name: admin-entry
          resources:
            requests:
              cpu: 10m
              memory: 512Mi
          stdin: true
          tty: true
          volumeMounts:
            - mountPath: /host-dir
              name: volume-1611735574911
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
        - hostPath:
            path: /
            type: ''
          name: volume-1611735574911
```

然后创建

```shell
# 创建
kubectl apply -f xxx.yaml
# 进入pod
kubectl exec -it admin-entry-xxx -- sh
```

## 大文件占用排查

与文件相关的有两个命令

- du：统计列举文件或文件夹的大小
- df：列出整个文件系统的占用情况。列出的是磁盘设备的占用率

一般大文件排查用du命令。涉及到三个参数

- -h：以人类可读的方式展示
- -d N：展示指定层级的目录
- -a：不只展示文件夹，文件也展示

其它参数通过`du --help`查看，对当前这个问题有如下：

```shell
# 进入挂载的目录
/ # cd /host-dir/
# 列出当前文件夹下第一层文件夹的大小
/host-dir # du -h -d 1
0	./sys
4.0K	./media
384.0K	./root
4.0K	./srv
37.5M	./etc
46.0M	./opt
32.0K	./tmp
2.2G	./usr
4.3M	./run
4.0K	./home
131.2M	./boot
0	./proc
4.0K	./mnt
33.3G	./var
16.0K	./lost+found
0	./dev
35.7G	.
```

可以看出主要占用在`/var`下。继续排查发现占用高主要问题有两个

- `/var/lib/docker/overlay2`，docker创建的容器、镜像、其他资源占用，不能随便删除。**使用`docker image prune -a -f`可以删除不用的镜像，以节省资源**。对ECS，通常可以直接发送命令达成此目标。
- `/var/log`，产生的日志，这里**我们可以删除不用的，能节省较多空间**。占用较多的是 `/var/log/messages`，来自`kubelet`和`systemd`。

## SSH访问节点

既然能够访问根文件系统，岂不是可以为所欲为：修改 `/root/.ssh/authorized_keys`，将本机ssh公钥写入，就能够远程访问了。此时，就取得了节点的完整控制权。

## 注意事项

- 遇到过进入节点打开过大的文件导致CPU占满，今儿导致节点不可用，大量应用挂掉的情况。所以如果要管理文件系统，还是新开一个节点，设好limit值，这样即时不小心占满pod的CPU、内存等资源，也不会导致节点出问题。比较保险。
- 如果觉得线上不方便查看日志，可以使用`kubectl cp`命令将其复制到本地，以更为便捷地查看。

## 总结

使用本文的方式可以管理文件系统，也可以在忘记密码的情况下进入节点管理，是一个通用的方案。

对于Kubernetes的磁盘占用率过高，通过上述两种清理闲置镜像 + 多余log文件的形式，可以节省出较多空间。如果能加个定期任务做这两件事，基本可以直接解决该问题。

```shell
#!/bin/sh
docker image prune -a -f;
rm /var/log/messages*
```

