+++

author = "lcomedy"
title = "python-client调用kubernetes的api实践"
date = "2022-01-03"
description = "通过实例的方式，介绍如何通过python-client来调用kubernetes的api"
tags = [
    "k8s",
    "python-client",
    "python",
    "cloud native",
]
categories = [
    "Develop",
    "CloudNative",
]
series = ["博客网站"]
aliases = ["搭建博客网站"]

+++

### 创建调用kubernetes的token

#### 创建账号

monitor-client-serviceAccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-client
  namespace: monitoring
```

创建账号

```shell
kubectl apply -f monitor-client-serviceAccount.yaml
```

#### 创建角色

monitor-client-clusterRole.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitor-client
rules:
- apiGroups:
  - "monitoring.coreos.com"
  resources:
  - servicemonitors
  - prometheusrules
  - prometheuses
  - alertmanagers
  verbs:
  - get
  - list
  - watch
  - create
  - delete
  - update
  - patch
```

创建角色

```shell
kubectl apply -f monitor-client-clusterRole.yaml
```

#### 创建绑定关系

monitor-client-clusterRoleBinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor-client
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitor-client
subjects:
- kind: ServiceAccount
  name: monitor-client
  namespace: monitoring
```

用户授权生效

```shell
kubectl apply -f monitor-client-clusterRoleBinding.yaml
```

#### 获取token

```shell
kubectl describe secret $(kubectl get secret -n monitoring | grep ^monitor-client | awk '{print $1}') -n monitoring | grep -E '^token'| awk '{print $2}'
```

### 安装kubernetes python sdk

#### 模块安装

```
pip install kubernetes
```

#### 测试demo

```python
from kubernetes import client, config

ApiToken = "xxxxx"  #ApiToken，使用前面生成的token
configuration = client.Configuration()
setattr(configuration, 'verify_ssl', False)
client.Configuration.set_default(configuration)
configuration.host = "https://xxxx:6443"    #ApiHost
configuration.verify_ssl = False
configuration.debug = True
configuration.api_key = {"authorization": "Bearer " + ApiToken}
client.Configuration.set_default(configuration)

k8s_api_obj  = client.CoreV1Api(client.ApiClient(configuration))
ret = k8s_api_obj.list_namespaced_pod("dev")
print(ret)
```

### 接口操作用例

[官方文档](https://github.com/kubernetes-client/python/blob/master/kubernetes/docs/CoreV1Api.md#create_namespace)

[核心接口使用参考](https://ops-coffee.cn/t/Kubernetes-Python-API-Demo.html)

划重点了！

由于prometheusrules是自定义资源，我们通过接口获取不要使用CoreV1Api的类型来调用，因为里面没有适合你的方法。不用着急，sdk中提供了另外一个核心类CustomObjectsApi。我们可以基于这个类来进行prometheusrules数据进行增删改查。

#### 创建prometheusrules

```python
from kubernetes.client import CustomObjectsApi

def create_monitor_object(self,name,plural,body):
  try:
    	self.crd_api.create_namespaced_custom_object(group=self.group,version=self.version,namespace=self.namespace,name=name,plural=plural,body=body)
    except ApiException as e:
      print("Exception when calling CoreV1Api->create_namespaced_crd: %s\n" % e)
      return None
    
def create_monitor_prometheusrules(self,name,body):
    ## 创建一个json格式的prometheusrule类型的资源数据
    body = {
    "apiVersion": "%s/%s" % (self.group, self.version),
    "kind": "PrometheusRule",
    "metadata": {
      "labels": {
        "prometheus": "k8s",
        "role": "alert-rules"
      },
      "name": name,
      "namespace": self.namespace
    },
    "spec": body
  }
    resp = self.create_monitor_object(name=name,plural="prometheusrules",body=body)
    print("create monitor_prometheusrules-----%s"%resp)
    return resp
```

#### 查看prometheusrules

```python
    def get_monitor_object(self,name,plural):
        try:
            resp = self.crd_api.get_namespaced_custom_object(group=self.group,version=self.version,namespace=self.namespace,name=name,plural=plural)
            return resp
        except ApiException as e:
            print("Exception when calling CoreV1Api->create_namespaced_endpoints: %s\n" % e)
            return None
            

    def get_monitor_prometheusrules(self,name):
        resp = self.get_monitor_object(name,plural="prometheusrules")
        print("get monitor_prometheusrules-----%s"%resp)
        return resp
```

#### 更新prometheusrules

```python
    def patch_monitor_object(self,name,plural,body):
        try:
            self.crd_api.patch_namespaced_custom_object(group=self.group,version=self.version,namespace=self.namespace,name=name,plural=plural,body=body)
        except ApiException as e:
            print("Exception when calling CoreV1Api->create_namespaced_endpoints: %s\n" % e)
            return None
          
    def patch_monitor_prometheusrules(self,name,body):
        old_resp = self.get_monitor_prometheusrules(name=name)
        ## 判断是否存在rule,存在则更新，不存在则创建
        if old_resp:
            old_resp["spec"] = body
            resp = self.patch_monitor_object(name=name,body=old_resp,plural="prometheusrules")
            print("patch monitor_prometheusrules-----%s" % resp)
            return resp
        else:
            self.create_monitor_prometheusrules(name=name,body=body,plural="prometheusrules")
```