---
title: Kubernetes初探 - 卷
date: 2019-10-22 20:12:21.0
updated: 2021-02-16 23:25:03.283
url: https://www.zou8944.com/archives/kubernetes初探-卷
categories: 运维
tags: kubernetes
---


卷是pod容器的组成部分，并非K8S中的顶级资源，其生命周期和pod一致。可以在pod的文件系统的任何位置挂在卷。如下两张图展示了同一个pod中存在多个容器时，在有卷和没有卷时的区别，可以看到在没有卷时，由于三个容器的文件系统分离，因此都各自操作自己的目录，即使他们在功能上是重复的；有卷时，将同一个卷挂在到两个容器的文件系统中，让他们共享这一块存储，既节省空间，也省去了从一个容器向另一个容器中复制的步骤。
<!-- more -->
![1571731921904](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571731921904.png)

![1571731875603](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571731875603.png)

卷消失后，卷的文件可能会保持原样，并且挂载到新的卷。这取决于卷的类型。

## 卷的类型

分为通用卷（k8s系统都会有的）和特殊卷（特殊用途或特殊k8s环境提供的卷）。这里仅介绍最常用的卷。

### emptyDir卷

最简单的卷类型，它就是一个空目录，容器可以向其中写入任何数据。与pod声明周期一致，pod消失时，卷的内容会丢失。

尽管它是最简单的卷，但是其它类型的卷都是在它的基础上发展而来的。

创建emptyDir卷时需要在pod配置文件中指定

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
    - image: luksa/fortune
      name: html-generator
      volumeMounts:
        - name: html
          mountPath: /var/htdocs
    - image: nginx:apline
      name: web-server
      volumnMounts:
        - name: html
        mountPath: /usr/share/nginx/html
        readOnly: true
  ports:
    - containerPort: 80
      protocol: TCP
  volumes:
    - name: html
      emptyDir: {}	# 默认是存在磁盘上的，可以按照如下方式配置在内存中
    #emptyDir:
    #  medium: Memory
```

### gitRepo卷

来源于emtryDir卷，它是在pod启动时检出git仓库中特定版本的内容填充到目录中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
    - image: nginx:apline
      name: web-server
      volumnMounts:
        - name: html
        mountPath: /usr/share/nginx/html
        readOnly: true
  ports:
    - containerPort: 80
      protocol: TCP
  volumes:
    - name: html
      gitRepo: 
        repository: http://git.dev.moumoux.com/ergedd/ergeddUtils
        revisioin: master
        directory: . # 将内容克隆到卷的根目录
```

由于gitRepo卷是在卷创建时从git克隆一次代码，因此当git仓库更新时pod是无法同步的。要做到同步，有如下两种方式

- 删除pod再创建
- 新建一个sidecar容器，用于同步卷中内容。Docker hub中搜索“git syc”可以看到很多相关的实现。

### hostPath卷

hostPath卷指向pod所属节点上的特定文件路径，同一个节点运行的两个pod通过hostPath卷可以看到相同的内容。它是一种持久存储的卷，因为节点文件系统中的内容并不会随着pod的释放而被删除。

它的缺点是对于pod来说不稳定，因为当发生pod调度时，可能被调度到另一个节点，这样对于这个pod来说之前写在节点上的文件就不见了。

只有需要在读取节点上的系统文件时才需要该类卷，一般应用不要使用它。

### 持久化的卷

上面说的三种卷，虽然通用，但要么不能做到持久化，要么是持久化对于pod没有实际的应用价值。

比如要为pod创建一个需要存储数据库文件的卷。这种需求要求pod无论如何变化，数据都能够持久化。解决方式是使用网络存储(NAS)。下面展示了几种合适的卷

```yaml
# 基于GCP持久盘的卷
......
spec:
  volumes:
    - name: mondodb-data
      gcpPersistentDisk:
        pdName: mondoDb
        fsType: ext4
  containers:
    - image: nginx:apline
      name: web-server
      volumnMounts:
        - name: mondodb-data
          mountPath: /usr/db
......
# 基于AWS持久盘的卷
......
spec:
  volumes:
    - name: mondodb-data
      awsElasticBlockStore:
        volumnId: my-volume
        fsType: ext4
......
# 基于普通共享NFS存储的卷
......
spec:
  volumes:
    - name: mondodb-data
      nfs:
        server: 1.2.3.4
        path: /some/path
......
```

![1571734843172](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571734843172.png)

## 持久卷

上面创建的持久化的卷，需要开发人员在配置pod时配置NAS的网络存储位置，使得开发人员、pod与底层的实际存储技术和存储地址耦合了起来。不符合k8s的策略，即对开发人员将底层技术抽象、解耦。

于是引入了**持久卷**和**持久卷声明**的概念。

持久卷（PersistentVolume）：封装了底层的存储技术，不属于任何命名空间，整个k8s共享，属于基础资源。是k8s集群管理员需要创建的资源。

持久卷声明（PersistentVolumeClaim）：用于开发人员声明需要的存储容量和访问模式，在命名空间内部。是开发人员自己需要创建的资源。

二者都是k8s的顶级资源，需要单独创建。有了持久卷和持久卷声明的示意图如下。

![1571736520272](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571736520272.png)

### 创建持久卷

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mogodb-pv
spec:
  capacity:
    storage: 1Gi  # 声明卷的大小
  accessModes:
    - ReadWriteOnce  # 可以被单个用户挂载为读写模式
    - ReadOnlyMany  # 可以被多个用户挂在为只读模式
  persistentVolumeReclaimPolicy: Retain  # 当声明被释放后，PersistentVolume将会被保留
  gcePersistentDisk:
    pdName: mongodb
    fsType
```



### 创建持久卷声明

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi  # 声明需要1Gi存储空间
  accessModes:
    - ReadWriteOnce  # 声明需要的访问模式为RWO
  storageClassName: ""
```

该声明和上面的持久卷是相匹配的，因此使用该声明时上述持久卷会被分配。

### 使用持久卷

```yaml
# 创建pod时的关于卷的声明如下
......
volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimValue: mongodb-pvc
......
```

使用后的结构

![1571738187793](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571738187793.png)

### 持久卷的回收机制

- Retain: 持久卷声明被释放后依然保留该持久卷，当PVC释放它后，它保持为释放状态，无法再被新的PVC引用。并且以后可以再配置给其它pod使用（文章是这么说，但我的问题是既然无法再被新的PVC引用，新的pod又如何去用这个被释放了的持久卷呢？）。删除该持久卷的唯一方式是手动删除持久卷资源。
- Recycle: 持久卷声明被释放后，该持久卷可以被新的PVC引用（数据应该还存在）
- Delete: 释放后删除持久卷

## StorageClass

上面我们看到了持久卷的用法，但它还有一个问题是需要集群管理员维护存储卷资源，这样不够科学。于是有了StorageClass这个资源。它是一个顶级资源，并且和持久卷一样，也是不属于任何命名空间的。

StorageClass定义了一个策略，在开发者使用PVC时创建一个新的持久卷，好处是不用管理员维护，并且与预先设置的持久卷不同，他不会耗尽持久卷存储。

### 使用

```yaml
# 声明一个StorageClass，这部分一般由管理员去做
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd  # 这是用于配置持久卷的插件，可以指定别的插件
parameters:  # 这是传给上述插件的参数
  type: pd-ssd
  zone: europe-west1-b

# PVC声明
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
    accessModes:
      - ReadWriteOnce
```

![1571746264340](https://gdz.oss-cn-shenzhen.aliyuncs.com/hexo/Kubernetes%20-%20%E5%8D%B7/1571746264340.png)

### 创建没有StorageClass的PVC

创建PVC时不指定SC，会被分配一个默认的SC。

### 强制将预先设置的PV绑定到PVC

```yaml
......
kind: PersistentVolumeClaim
spec:
  storageClassName: ""  # 将该字段设置为空，则是绑定到预先手动配置的PV，而不是动态配置的PV
......
```