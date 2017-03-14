---
layout: post
title: Secrets
date: 2016-10-26 18:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


## Secrets 构建

### yaml 构建

```
test-secret.yaml：
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

<!-- more -->

构建 test-secret Secret

```bash
$ kubectl create -f demo.yaml
```

### 文件构建

创建 password.txt, username.txt:

```bash
# Create files needed for rest of example.
$ echo -n "admin" > ./username.txt
$ echo -n "1f2d1e2e67df" > ./password.txt
```

构建 db-user-pass Secret

```bash
$ kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
secret "db-user-pass" created
```

### 手动创建 Secret
Encode:

```bash
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

Decode:

```bash
$ echo "MWYyZDFlMmU2N2Rm" | base64 -d
1f2d1e2e67df
```

## 在 pod 中应用 Secret

### 案例 1：在环境变量中的使用 Secret

参数列表：

- env[x].valueFrom.secretKeyRef.

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: mycontainer
      image: redis
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
  restartPolicy: Never
```

### 案例 2：通过数据卷插件使用 Secret

参数列表：

- spec.volumes[].secret.secretName
- spec.containers[].volumeMounts[]
   - spec.containers[].volumeMounts[].readOnly = true
   - spec.containers[].volumeMounts[].mountPath

pod 应用：

```json
{
 "apiVersion": "v1",
 "kind": "Pod",
  "metadata": {
    "name": "mypod",
    "namespace": "myns"
  },
  "spec": {
    "containers": [{
      "name": "mypod",
      "image": "redis",
      "volumeMounts": [{
        "name": "foo",
        "mountPath": "/etc/foo",
        "readOnly": true
      }]
    }],
    "volumes": [{
      "name": "foo",
      "secret": {
        "secretName": "mysecret"
      }
    }]
  }
}
```

Secrets 挂载到数据卷指定目录下

参数列表：

- spec.volumes[].secret.items

```
{
 "apiVersion": "v1",
 "kind": "Pod",
  "metadata": {
    "name": "mypod",
    "namespace": "myns"
  },
  "spec": {
    "containers": [{
      "name": "mypod",
      "image": "redis",
      "volumeMounts": [{
        "name": "foo",
        "mountPath": "/etc/foo",
        "readOnly": true
      }]
    }],
    "volumes": [{
      "name": "foo",
      "secret": {
        "secretName": "mysecret",
        "items": [{
          "key": "username",
          "path": "my-group/my-username"
        }]
      }
    }]
  }
}
# 挂在目录：/etc/foo/my-group/my-username
```

### 案例 3：加密拉取镜像

myregistrykey.yaml：

```
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
  namespace: awesomeapps
data:
  .dockerconfigjson: UmVhbGx5IHJlYWxseSByZWVlZWVlZWVlZWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGx5eXl5eXl5eXl5eXl5eXl5eXl5eSBsbGxsbGxsbGxsbGxsbG9vb29vb29vb29vb29vb29vb29vb29vb29vb25ubm5ubm5ubm5ubm5ubm5ubm5ubm5ubmdnZ2dnZ2dnZ2dnZ2dnZ2dnZ2cgYXV0aCBrZXlzCg==
type: kubernetes.io/dockerconfigjson
pod 应用：
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```
