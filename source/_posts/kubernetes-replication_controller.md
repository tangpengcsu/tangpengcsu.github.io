---
layout: post
title: Replication Controller
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


> 已被 RS 取代

## RC 伸缩 - Resizing a Replication Controller

### 语法：

```bash
$ kubectl scale rc NAME --replicas=COUNT \
    [--current-replicas=COUNT] \
    [--resource-version=VERSION]
```

  - NAME: 待更新的 RC 名称，必选。
  - replicas: 期望伸缩的副本数，必选。
  - current-replicas: 当前 RC 副本数，可选。
  - resource-version: 匹配 RC labels[].version 的值，可选。

  <!-- more -->
  
### 滚动更新：

滚动跟新仅支持 RC，不支持 Deployment

### 通过配置文件滚动更新

#### 先决条件：

- RC 必须制定 metadata.name 的在值。
- Overwrite at least one common label in its spec.selector field.
- 使用同一的命名空间 metadata.namespace

```bash
 Update pods of frontend-v1 using new replication controller data in frontend-v2.json.
$ kubectl rolling-update frontend-v1 -f frontend-v2.json

// Update pods of frontend-v1 using JSON data passed into stdin.
$ cat frontend-v2.json | kubectl rolling-update frontend-v1 -f -
```

### 滚动更新容器镜像

```bash
$ kubectl rolling-update NAME [NEW_NAME] --image=IMAGE:TAG
```

  - NEW_NAME: 为 RC 重命名，可选。

### 回滚：

```bash
$ kubectl rolling-update NAME --rollback
```

## 模版

**yaml:**

```
apiVersion: v1
kind: ReplicationController
metadata:
  name:
  labels:
  #  labels1: values1
  #  labels2: values2
  namespace:
spec:
  replicas: int
  selector:
  #  labels1: values1
  #  labels2: values2
  template:
    metadata:
      labels:
    #    labels1: values1
    #    labels2: values2
    spec:
      # please, See pod spec schema.
```

**json：**

```json
{
  "kind": "ReplicationController",
  "apiVersion": "v1",
  "metadata": {
    "name": "frontend-controller",
    "labels": {
      "state": "serving"
    }
  },
  "spec": {
    "replicas": 2,
    "selector": {
      "app": "frontend"
    },
    "template": {
      "metadata": {
        "labels": {
          "app": "frontend"
        }
      },
      "spec": {
        "volumes": null,
        "containers": [
          {
            "name": "php-redis",
            "image": "redis",
            "ports": [
              {
                "containerPort": 80,
                "protocol": "TCP"
              }
            ],
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "dnsPolicy": "ClusterFirst"
      }
    }
  }
}
```

**案例：**

```json
{
  "kind": "ReplicationController",
  "apiVersion": "v1",
  "metadata": {
    "name": "frontend-controller",
    "labels": {
      "state": "serving"
    }
  },
  "spec": {
    "replicas": 2,
    "selector": {
      "app": "frontend"
    },
    "template": {
      "metadata": {
        "labels": {
          "app": "frontend"
        }
      },
      "spec": {
        "volumes": null,
        "containers": [
          {
            "name": "php-redis",
            "image": "redis",
            "ports": [
              {
                "containerPort": 80,
                "protocol": "TCP"
              }
            ],
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "dnsPolicy": "ClusterFirst"
      }
    }
  }
}
```
