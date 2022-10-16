---
title: 关于ConfigMap配置更新机制
date: 2022/10/11
tag: Kubernetes
author: Slide
categories: Kubernetes
---

工作中遇到部分组件，它们依赖的一些配置需要通过ConfigMap挂载到容器内部，但ConfigMap自动更新存在一定延时，这可能会带来一系列的问题。在不能滚动重建的前提下，会存在例如配置刷新不及时等问题，如果本身组件不支持热加载的话，就需要通过一些自定义任务去做这些事。

<!--more-->

## 原因

### Kubelet

Kubelet在每个Node节点都会安装，负责维护该节点上的所有容器，并监视容器的健康状态。同步容器需要的数据，数据可能来自配置文件，也可能来自Etcd。Kubelet通过启动参数--sync-frequency来控制同步的间隔时间。它的默认值是1min，所以更新ConfigMap的内容后，真正容器中的挂载内容变化可能在`0~1min`之后。


![indexFile-2](/images/cm.png)


在K8s的官方文档中，说明了ConfigMap 既可以通过 Watch 操作实现内容传播（默认形式），也可实现基于 TTL 的缓存，还可以直接经过所有请求重定向到 API 服务器。 因此，从 ConfigMap 被更新的那一刻算起，到新的主键被投射到 Pod 中去，这一时间跨度可能与 Kubelet 的同步周期加上高速缓存的传播延迟相等。 这里的传播延迟取决于所选的高速缓存类型，分别对应 Watch 操作的传播延迟、高速缓存的 TTL 时长或者 0。

基于此总延迟时间：

```
TotalDelayTime = kubelet sync-frequency + watch manager delay
```

## 解决

### 应用本身监听本地配置文件

在RocketMQ中，Acl的更新就通过内核监听配置文件从而实现了热更新。在RocketMQ On K8s中，一种比较常见的做法是，将Acl的配置文件通过ConfigMap挂载，但由于本身ConfigMap的延迟，在用户体验上来说，并不是一种很好的解决方案。更好的做法是通过内核提供的API接口完成Acl配置的更新。

### 使用 Sidecar 来监听本地配置文件变更

这也是业内比较标准的做法。Prometheus 的 Helm Chart 中使用的就是这种方式。这里有一个很实用的镜像叫做 [configmap-reload](https://github.com/jimmidyson/configmap-reload)，它会去 watch 本地文件的变更，并在发生变更时通过 HTTP 调用通知应用进行热更新。同时Emqx Operator对于插件的加载也使用了这种方式。[emqx-operator](https://github.com/emqx/emqx-operator/blob/main/sidecar/reloader/main.go)



## Reference
[ConfigMaps | Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

[Kubernetes Pod 中的 ConfigMap 配置更新 (aleiwu.com)](https://aleiwu.com/post/configmap-hotreload/#热更新二-使用-sidecar-来监听本地配置文件变更)