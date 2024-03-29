---
title: Kubernetes的安全构建概述
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Security
date: 2022-09-06 17:28:43
---

# 当谈论K8s安全时，我们在谈论什么？

其实，我们在谈论所有。借官方手册的描述——4个C：Cloud、Cluster、Container、Code，即在这四个层面上进行防护
<!--more-->

<img src="https://gdz.oss-cn-shenzhen.aliyuncs.com/local/image-20220907105246188.png" alt="image-20220907105246188" style="zoom:30%;"/>

- Cloud即云基础设施，包含ECS节点、负载均衡、网关等
- Cluster即K8s集群本身，集群内部暴露的访问接口尤其是ApiServer、集群管理角色和权限、外部进入的不安全的配置等
- Container层面，主要是需要扫描镜像是否有非法操作，比如基于不安全的镜像构建、是否挂载了本地镜像容器扫描、是否以管理员身份启动服务等
- Code层面则与程序员的日常更为紧密：代码中是否包含了敏感信息如密码、是否依赖了包含漏洞的三方库等

# 具体来说

## Cloud

尽管基础设施涉及到的内容更加传统也更加复杂，却四4个C最容易实现的，因为我们通常并不会自己搭建和维护K8s这个如此复杂的工具，而是将其放在云平台。云平台为我们提供了如下安全便利

- 操作系统：ECS操作系统安全漏洞扫描和修复。**我们需要根据建议修复漏洞，当前没管。**
- 隔绝外部访问：安全组功能，尤其是节点加入集群时会重装系统，并且使用单独的更加严格的安全组，防止通过端口非法进入节点
- 监控：基础服务的监控功能。在阿里云的云监控产品中，我们设置了对所有使用中的基础服务的监控功能，并在指标异常时报警
- 账号：基于角色的账户管理系统，**这需要账号管理员承担更多责任：遵循最小授权原则，不能图方便将管理员权限赋予非必要人士**

如上几点，对当前的我们来说，基本够用了。

## Cluster

这是实现安全管理的重点，包含集群组件和应用组件的防护。

### 集群组件

外部可能访问到的集群组件如下

- API Server

  - 应当只允许HTTPS访问
  - 应当开启RBAC

- Kubelet

  有暴露API，默认情况下允许匿名访问。应当开启身份验证，拒绝匿名访问

- etcd

  集群敏感信息存储在etcd，应当限制etcd的访问，并在敏感信息如Secret落盘前加密。

此外，还应当开启集群审计日志，方便问题追溯。

### 应用组件

这部分即我们添加应用时应当遵循的规则

- 资源占用风险：集群节点资源耗尽导致应用下线，尽管数据未泄露，但造成了事实上的宕机，也是安全风险。所以应该为每个应用设置request和limit资源限制

- Pod准入：更准确地说，应该是应用创建配置文件扫描，拒绝不合法的应用被创建，不合法应用举例

  - 拥有特权的pod
  - 能够访问主机磁盘的pod
  - 以root身份运行容器的pod
  - ...

  以上这类pod，如果被作为突破口入侵，将会危机整个集群。

  Pod的准入机制需要配合SecurityContext使用

  > 你可能会觉得pod的这些限制和后面所说的容器限制有类似的地方，但要分清楚它们是不同层面的东西：在Pod层一定要限制，因为一个Pod包含多个容器，有可能包含的三方容器未遵循安全规范。

- 网络策略：基于零信任原则，一个应用不应该能够被预期之外的应用访问，这可以通过网络策略实现。不过直接管理网络策过于繁琐，这可以通过服务网格或增强版网络插件实现。

- Ingress的TLS支持：除了集群组件，应用暴露的Ingress是另一个外部访问接口，应当增加TLS支持

- 定期扫描集群配置，识别有问题的配置

### 实施

在使用ACK的语境下，Master节点由阿里云提供和维护，它们解决了etcd敏感数据加密的问题。

其它我们已经做了的事情

- API Server加密
- 开启并配置RBAC
- 为每个应用设置request和limit资源限制，不过当前由于应用启动占用CPU过高，无法处理多个应用同时启动过慢的问题
- Ingress的TLS支持

其它我们**没做但可以做的事情**

- 开启Pod准入：ACK不支持官方的准入控制器，但自己实现了一个类似功能策略管理器，可以开启

  这是一个连锁改动，开启准入意味着要指定准入规则，并将现有的十几个应用配置依照新的准入规则修改

- 定期扫描集群配置：ACK提供扫描插件，可以直接开启使用

其它我们没做但将来可以做的事情

- 网络策略控制：管理负担太重，需要的时候再开吧

## Container

这部分主要关注容器镜像安全

- 镜像安全：镜像依赖是否有漏洞
- 镜像签名：防止镜像被篡改
- 禁止root用户：构建镜像时应该以普通用户身份启动应用，而不是root用户

关于这部分，阿里云容器镜像服务企业版提供镜像扫描和镜像签名功能，针对镜像签名功能需要在K8s安装验签插件。而企业版开启费用过高：1600元/月。

为此，我们可以使用开源工具trivy替代：将trivy加入CI流程。

> 这里涉及到一个问题：加入trivy容器，但其检测报告所报漏洞需要较强的安全专业知识判断是否修复以及如何修复。所以接入的作用存疑。
>
> 相反，在开发人员较少的情况下，自觉使用正规渠道的基础镜像，结合Pod准入机制，应该是更加实际且实用的方式。
>
> 即，**相比现在的使用方式，什么都不用改**。

## Code

代码层面本身，更多地涉及到的是编码安全问题

- Web API的安全防护：这是另外一个故事了
- 敏感信息硬编码检测：可以使用Gitlab提供的支持，也可以手动将gitleaks加入CI，实验下来，后者更为方便。
- 三方依赖库安全扫描：使用trivy同样可以达成，其支持多种语言。

所以我们可以做的事情有

- gitleaks加入CI，防止密码硬编码
- trivy加入CI，生成检测安全报告（需要设置为允许失败）

# 总结

之前我们有意无意地已经支持了一些安全措施，但还有可以做的事情，总结一下需要做的事情

- 关注ECS漏洞报告，及时修复
- 开启pod准入机制
- 定期扫描集群配置
- 引入gitleaks防止密码硬编码
- 引入trivy扫描三方依赖和容器镜像

# 关于零信任

“零信任”三个字时不时会出现在技术手册和文章中。顾名思义，就是在系统中，谁都是不可信任的。要在这种情况下完成可靠的通信，就是零信任模型需要解决的课题。零信任模型的几个特征

- 持续验证：针对每次通信都进行验证
- 微分段：将资源分成若干分段，以进行授权管理
- 最小访问权限：对一个访问账户，非必要不授权

- 设备访问控制：除了验证访问的逻辑账户，还要验证访问的设备
- 横向移动预防：防止攻击传播，即如果一个应用受到攻击，如果攻击者可以通过该应用继续渗透到其它应用，就叫横向移动。

Google和微软都有推出自己的零信任产品，如BeyondCorp，它用于在不安全网络环境下构建一个安全的企业内部网，让企业员工随时随地都能够访问企业内部资源，是对传统VPN网络的升级。依照上面的几个特征，来看它是怎么做的

- 持续验证：每次通信都验证，除了用户登录信息外，还需要传入网络环境信息、设备信息等，经过计算得出对应安全等级，再决定是否需要权限降级
- 微分段：将企业资源以统一的管理形式进行注册管理
- 最小访问权限：用户需要精细化授权管理，这需要仰仗管理员
- 设备访问控制：终端用户（即员工）需要在手机上安装客户端以实时监测设备安全状况
- 横向移动预防：这个暂未看到明确的描述，不过资源隔离应该就能做到🤔

> 我的环境怎么就零信任了这个姑且放一边（要好好解释可能需要具备一定的安全渗透知识），在没有具备专业安全知识的情况下，我们需要假设零信任成立，然后才能进行讨论

## 零信任与K8s

回顾零信任的几个特征，主要针对 客户端 - 服务端 这样的南北流量网络，K8s内部还包含了服务端内部的东西向流量，因此预防攻击的横向移动也很重要，解法通常有两个

- mTLS：保证应用不被未授权访问、通信中的网络嗅探。但如果黑客进入节点或pod，获取到了TLS秘钥，这层防护就没用了。
- 网络策略：控制应用只能访问依赖的应用，控制应用只能被依赖的应用访问。

# 一些想法

安全这个课题很复杂，是个无底洞，我想大概有几个方面

- 专业知识要求高：操作系统、网络、实践。。。安全本身就是融合了各种知识的学科，安全漏洞产生往往是对一项技术的深入研究得出的，别说防住，要想理解漏洞的产生及防守方式，就需要对该技术同样深入了解。这就导致需要学习的东西非常多。然而大家精力是有限的，这造成恶性循环。
- 应对安全问题的态度：开发人员，由于缺乏专业知识，只能做出有限的防范措施；管理人员，安全的产出与投入并不正相关，产品前期投入性价比不高，没有又不行，如何把握这个度是个问题

我认为正确的态度是“安全无法尽善尽美”，应该始终选择高性价比地引入安全措施。

# 参考文档

1. [云原生安全白皮书](https://cloud.tencent.com/developer/article/1855108)
2. [Kubernetes安全白皮书](https://lib.jimmysong.io/kubernetes-hardening-guidance/introduction/)
3. [Kubernetes安全手册](https://kubernetes.io/zh-cn/docs/concepts/security/overview/)
4. [Pod安全标准](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/)
5. [Pod安全准入控制](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission/)
6. [ACK安全手册](https://help.aliyun.com/document_detail/86788.html)
7. [trivy](https://github.com/aquasecurity/trivy)
8. [gitleaks](https://github.com/zricethezav/gitleaks)
9. [Google BeyondCorp零信任技术的论文](https://cloud.google.com/beyondcorp?hl=zh-cn)
10. [零信任知识介绍](https://lib.jimmysong.io/blog/zero-trust-developer-guide/)
11. [零信任网络介绍](https://www.catonetworks.com/zero-trust-network-access/#What_is_ZTNA_(Zero_Trust_Network_Access))
12. [mTLS指南](https://lib.jimmysong.io/blog/mtls-guide/)
13. [Cloudflare使用mTLS实现零信任安全](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-mutual-tls/)
