## 金丝雀发布

这里使用`Flagger` (https://flagger.app/) 来自动化地实现金丝雀发布。

![image](https://user-images.githubusercontent.com/4653664/224696379-02c57b5d-2d97-49e7-afe1-dc8da2612c85.png)


### 1. 安装 `flagger`

```
# helm repo add flagger https://flagger.app

# kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml

# helm install flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set meshProvider=istio \
--set metricsServer=http://prometheus:9090

```
> 默认已经在istio-system下已经安装了prometheus组件

### 2. 初始化金丝雀发布

创建金丝雀资源

```
# kubectl apply -f catalog-release.yaml
```

观察金丝雀发布过程

```
# kubectl get canary catalog-release -w
NAME              STATUS         WEIGHT   LASTTRANSITIONTIME
catalog-release   Initializing   0        2023-03-13T10:47:35Z
catalog-release   Initialized    0        2023-03-13T10:48:21Z
```

默认情况下，`flagger`会基于`Canary`创建对应`virtualservice`资源配置，用于控制流量路由,查看自动创建的`virtualservice`资源配置

```
# kubectl get virtualservice catalog -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    kustomize.toolkit.fluxcd.io/reconcile: disabled
  creationTimestamp: "2023-03-13T10:48:21Z"
  generation: 1
  name: catalog
  namespace: istioinaction
  ownerReferences:
  - apiVersion: flagger.app/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: Canary
    name: catalog-release
    uid: ba820d19-ad64-4273-8a6b-2187cb26dbc7
  resourceVersion: "1696543"
  uid: 30310d9a-2f57-490a-8d23-463dbb68c2fd
spec:
  gateways:
  - mesh
  hosts:
  - catalog
  http:
  - match:
    - sourceLabels:
        app: webapp
    route:
    - destination:
        host: catalog-primary
      weight: 100
    - destination:
        host: catalog-canary
      weight: 0
  - route:
    - destination:
        host: catalog-primary
      weight: 100

```

目前，只有`v1`版本的`catalog`服务，所以所有的路由都指向`v1`.

```
[root@osstest2 ~]# while true; do  curl webapp.istioinaction.io/api/catalog; printf "\n";sleep 1;done
```
> 终端不要关闭，持续观察变化

![image](https://user-images.githubusercontent.com/4653664/224697502-b04f1850-3905-4994-afef-c2f9225adfd6.png)

### 3. 新版本金丝雀发布

接下来，部署`v2`版本`catalog`服务，并观察金丝雀自动发布过程。

```
# cat catalog-deployment-v2.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  template:
    metadata:
      labels:
        app: catalog
        version: v1
    spec:
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: SHOW_IMAGE
          value: "true"
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false


# kubectl apply -f catalog-deployment-v2.yaml

# kubectl get canary catalog-release -w
NAME              STATUS        WEIGHT   LASTTRANSITIONTIME
catalog-release   Progressing   0        2023-03-13T10:51:21Z
catalog-release   Progressing   10       2023-03-13T10:52:06Z
catalog-release   Progressing   10       2023-03-13T10:52:51Z
catalog-release   Progressing   20       2023-03-13T10:53:36Z
catalog-release   Progressing   30       2023-03-13T10:54:21Z
catalog-release   Progressing   40       2023-03-13T10:55:06Z
catalog-release   Progressing   50       2023-03-13T10:55:51Z
catalog-release   Promoting     0        2023-03-13T10:56:36Z
catalog-release   Finalising    0        2023-03-13T10:57:21Z
catalog-release   Succeeded     0        2023-03-13T10:58:06Z
```

此时，应该在测试终端上看到如下的效果,即全部返回`v2`版本的`catalog`服务响应：

```
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00","imageUrl":"http://lorempixel.com/640/480"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00","imageUrl":"http://lorempixel.com/640/480"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00","imageUrl":"http://lorempixel.com/640/480"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00","imageUrl":"http://lorempixel.com/640/480"}]

```

### 4. 环境清理

```
# kubectl delete canary catalog-release
# kubectl delete deploy catalog
# kubectl apply -f catalog-svc.yaml
# kubectl apply -f catalog-deployment.yaml
# kubectl apply -f catalog-dest-rule.yaml
# helm uninstall flagger -n istio-system
```
