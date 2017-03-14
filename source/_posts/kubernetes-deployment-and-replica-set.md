---
layout: post
title: Deployment 与 Replica Set
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---

## Deployment

### 简介

Deployment 是新一代用于 Pod 管理的对象，与 RC 相比，它提供了更加完善的功能，使用起来更加简单方便。

Deployment 与 RC 的定义也基本相同，需要注意的是 apiVersion 和 kind 是有差异的。     

Deployment 会声明 Replica Set 和 Pod

### Deployment 更新

1. rolling-update。只有当 Pod template 发生变更时，Deployment 才会触发 rolling-update。此时 Deployment 会自动完成更新，且会保证更新期间始终有一定数量的 Pod 为运行状态。
2. 其他变更，如暂停 / 恢复更新、修改 replica 数量、修改变更记录数量限制等操作。这些操作不会修改 Pod 参数，只影响 Deployment 参数，因此不会触发 rolling-update。

通过 kubectl edit 指令更新 Deployment，可以将例子中的 nginx 镜像版本改成 1.9.1 来触发一次 rolling-update。期间通过 kubectl get 来查看 Deployment 的状态，可以发现 CURRENT、UP-TO-DATE 和 AVAILABLE 会发生变化。

<!-- more -->

### 删除 Deployment

kubectl delete 指令可以用来删除 Deployment。需要注意的是通过 API 删除 Deployment 时，对应的 RS 和 Pods 不会自动删除，需要依次调用删除 Deployment 的 API、删除 RS 的 API 和删除 Pods 的 API。
使用 RS 管理 Pod

## Replica Set - RS

Replica Set（简称 RS ）是 k8s 新一代的 Pod controller。与 RC 相比仅有 selector 存在差异，RS 支持了 set-based selector（可以使用 in、notin、key 存在、key 不存在四种方式来选择满足条件的 label 集合）。Deployment 是基于 RS 实现的，我们可以使用 kubectl get rs 命令来查看 Deployment 创建的 RS：

```bash
$ kubectl get rs
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-1564180365   3         3         6s
nginx-deployment-2035384211   0         0         36s
```

由 Deployment 创建的 RS 的命名规则为 "<Deployment 名称>-<pod template 摘要值 >"。由于之前的操作中我们触发了一次 rolling-update，因此会查看到两个 RS。更新前后的 RS 都会保留下来。

### 弹性伸缩

与 RC 相同，只需要修改. spec.replicas 就可以实现 Pod 的弹性伸缩。

### 重新部署

如果设置 Deployment 的 .spec.strategy.type==Recreate 时，更新时会将所有已存在的 Pod 杀死后再创建新 Pod。与 RC 不同的是，修改 Deployment 的 Pod template 后更新操作将会自动执行，无需手动删除旧 Pod。

### 更完善的 rolling-update

与  RC  相比，Deployment  提供了更完善的 rolling-update 功能：

1. Deployment 不需要使用 kubectl rolling-update 指令来触发 rolling-update，只需修改 pod template 内容即可。这条规则同样适用于使用 API 来修改 Deployment 的场景。这就意味着使用 API 集成的应用，无须自己实现一套基于 RC 的 rolling-udpate 功能，Pod 更新全都交给 Deployment 即可。
2. Deployment 会对更新的可用性进行检查。当使用新 template 创建的 Pod 无法运行时，Deployment 会终止更新操作，并保留一定数量的旧版本 Pod 来提供服务。例如我们更新 nginx 镜像版本为 1.91（一个不存在的版本），可以看到以下结果：
3. 此外 Deployment 还支持在 rolling-update 过程中暂停和恢复更新过程。通过设置. spec.paused 值即可暂停和恢复更新过程。暂停更新后的 Deployment 可能会处于与以下示例类似的状态：
4. 支持多重更新，在更新过程中可以执行新的更新操作。Deployment 会保证更新结果为最后一次更新操作的执行结果。
5. 影响更新的一些参数：
  1. spec.minReadySeconds 参数用来设置确认运行状态的最短等待时间。更新 Pod 之后，Deployment 至少会等待配置的时间再确认 Pod 是否处于运行状态。也就是说在更新一批 Pod 之后，Deployment 会至少等待指定时间再更新下一批 Pod。
  2. spec.strategy.rollingUpdate.maxUnavailable 用来控制不可用 Pod 数量的最大值，从而在删除旧 Pod 时保证一定数量的可用 Pod。如果配置为 1，且 replicas 为 3。则更新过程中会保证至少有 2 个可用 Pod。默认为 1。
  3. spec.strategy.rollingUpdate.maxSurge 用来控制超过期望数量的 Pod 数量最大值，从而在创建新 Pod 时限制总量。如配置为 1，且 replicas 为 3。则更新过着中会保证 Pod 总数量最多有 4 个。默认为 1。
  4. 后两个参数不能同时为 0。

## Deployment 模版：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```
