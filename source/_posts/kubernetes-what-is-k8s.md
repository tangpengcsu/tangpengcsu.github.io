---
layout: post
title: 什么是 Kubernetes?
date: 2016-10-26 18:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---

Kubernetes 是一个开源的自动化部署，

<!-- more -->

## Containers 和 Pods

### Pod 生命周期

#### Pod Phase （Pod 状态）

- Pending
- Runing
- Succeeded
- Failed
- Unknow

#### Pod Conditions（Pod 条件）

#### Container Probes（容器探查）

#### Container Statuses（容器状态）

##### RestartPolicy

- Always
- OnFailure
- Never

#### Pod lifetime（Pod 生命周期）

#### Examples（范例）

##### Advanced livenessProbe example

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: gcr.io/google_containers/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
          - name: X-Custom-Header
            value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```
