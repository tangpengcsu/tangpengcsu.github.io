---
layout: post
title: Heapster 在 Kubernetes 集群中的部署--包含 InfluxDB 后台 和 Grafana UI
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


> 原文引用：[https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md](https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md "https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md")

## 启动 k8s 集群

## 启动所有的 pods 和 services

为部署 Heapster 和 InfluxDB，你需要通过 [deploy/kube-config/influxdb](https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb "https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb") 创建 kubernetes 资源。确保你已经在 `root`目录下 `checkout` 有效的 `Heapster` 资源。

```bash
git clone https://github.com/kubernetes/heapster.git
```

<!-- more -->

运行：

```bash
$ kubectl create -f deploy/kube-config/influxdb/
```

默认 Grafana 服务负载均衡器的请求。 如果在你的集群中不可用，可以考虑更换 `NodePort`。为了访问 Grafana，为其指派 `extenal IP` 。默认的用户名和密码都是 `'admin'`。在登录到 Grafana 后，为其添加 InfluxDB 数据源。 InfluxDB 的 url 为 `http://localhost:8086`。数据库名为`'k8s'`。默认的用户名和密码均为`'root'`。 Grafana 的 InfluxDB 文档在[这里](http://docs.grafana.org/datasources/influxdb/ "http://docs.grafana.org/datasources/influxdb/")

为了更好的理解 InfluxDB，可以了解 [storage schema](https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md "https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md")

Grafana 通过使用模版设置填充节点和pods。

Grafana Web 界面也能够通过 api-server proxy 访问。一旦资源被创建后，就可以通过 `kubectl cluster-info` 查看其 URL。

## Troubleshooting

也可以通过 [debugging documentation](https://github.com/kubernetes/heapster/blob/master/docs/debugging.md "https://github.com/kubernetes/heapster/blob/master/docs/debugging.md") 查看。

1\. 如果 Grafana service 无法访问，可能是因为服务还没启动。使用 `kubectl` 验证 `heapster`， `influxdb` 和 `grafana` 的 pods 是否存活。

```bash
$ kubectl get pods --namespace=kube-system
...
monitoring-grafana-927606581-0tmdx        1/1       Running   0          6d
monitoring-influxdb-3276295126-joqo2      1/1       Running   0          15d
...

$ kubectl get services --namespace=kube-system monitoring-grafana monitoring-influxdb
```

2\. 如果你发现 InfluxDB 占用过多的 CPU 或 内存，可以考虑在中限制 `InfluxDB & Grafana` pod 的资源。在 []() 用 cpu: `<millicores>` and `memory: <bytes>`2\. 如果你发现 InfluxDB 占用过多的 CPU 或 内存，可以考虑在中限制 `InfluxDB & Grafana` pod 的资源。在 Controller Spec 中添加 cpu: `<millicores>` and `memory: <bytes>`，运行:

```bash
kubectl apply -f deploy/kube-config/influxdb/influxdb-grafana-controller.yaml
```

重启 Controllers 并删除就的 InfluxDB pods。
