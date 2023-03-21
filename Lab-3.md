## 流量路由

在新版本软件上线时，有诸多策略，如蓝绿部署、灰度发布、金丝雀发布等，在Istio中，可以通过流量路由功能来实现这些上线策略。

![image](https://user-images.githubusercontent.com/4653664/226556377-75362ad1-9563-4db8-9e34-c4868793c681.png)


ISTIO资源对象图解


![image](https://user-images.githubusercontent.com/4653664/223953387-ef9f1cff-fb0a-4d2f-b4e2-cfa62f21f540.png)


部署`v2`版本的`catalog`服务

```
# cat services/catalog/kubernetes/catalog-deployment-v2.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v2
  name: catalog-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v2
  template:
    metadata:
      labels:
        app: catalog
        version: v2
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
```

```
# kubectl apply -f services/catalog/kubernetes/catalog-deployment-v2.yaml
```

通过`DestinationRule`声明`catalog`服务的版本

```
# cat ch2/catalog-destinationrule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

具体版本的匹配是通过标签实现的

```
# kubectl get pods --show-labels
NAME                          READY   STATUS    RESTARTS   AGE     LABELS
catalog-5c7f8f8447-n996v      2/2     Running   0          4h1m    app=catalog,pod-template-hash=5c7f8f8447,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=catalog,service.istio.io/canonical-revision=v1,version=v1
catalog-v2-65cb96c66d-rlrdr   2/2     Running   0          19m     app=catalog,pod-template-hash=65cb96c66d,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=catalog,service.istio.io/canonical-revision=v2,version=v2
webapp-8dc87795-sqlzk         2/2     Running   0          3h59m   app=webapp,pod-template-hash=8dc87795,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=webapp,service.istio.io/canonical-revision=latest

```
应用配置

```
# kubectl apply -f ch2/catalog-destinationrule.yaml
```

针对`catalog`服务配置，带有`x-dark-launch: "v2"`的请求头将会路由到`v2`版本，其它默认路由到`v1`版本

```
# cat ch2/catalog-virtualservice-dark-v2.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - match:
    - headers:
        x-dark-launch:
          exact: "v2"
    route:
    - destination:
        host: catalog
        subset: version-v2
  - route:
    - destination:
        host: catalog
        subset: version-v1
```

应用路由策略

```
# kubectl apply -f ch2/catalog-virtualservice-dark-v2.yaml
```

访问`v2`版本`catalog`服务,`v2`比`v1`版本多了一个字段`"imageUrl"`

```
# curl -v http://webapp.istioinaction.io/api/catalog -H "x-dark-launch: v2"
*   Trying 141.147.176.63...
* TCP_NODELAY set
* Connected to webapp.istioinaction.io (141.147.176.63) port 80 (#0)
> GET /api/catalog HTTP/1.1
> Host: webapp.istioinaction.io
> User-Agent: curl/7.61.1
> Accept: */*
> x-dark-launch: v2
>
< HTTP/1.1 200 OK
< content-length: 529
< content-type: application/json; charset=utf-8
< date: Thu, 09 Mar 2023 07:18:43 GMT
< x-envoy-upstream-service-time: 12
< server: istio-envoy
<
* Connection #0 to host webapp.istioinaction.io left intact
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00","imageUrl":"http://lorempixel.com/640/480"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00","imageUrl":"http://lorempixel.com/640/480"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00","imageUrl":"http://lorempixel.com/640/480"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00","imageUrl":"http://lorempixel.com/640/480"}]
```

访问`v1`版本`catalog`服务

```
# curl -v http://webapp.istioinaction.io/api/catalog
*   Trying 141.147.176.63...
* TCP_NODELAY set
* Connected to webapp.istioinaction.io (141.147.176.63) port 80 (#0)
> GET /api/catalog HTTP/1.1
> Host: webapp.istioinaction.io
> User-Agent: curl/7.61.1
> Accept: */*
>
< HTTP/1.1 200 OK
< content-length: 357
< content-type: application/json; charset=utf-8
< date: Thu, 09 Mar 2023 07:36:53 GMT
< x-envoy-upstream-service-time: 8
< server: istio-envoy
<
* Connection #0 to host webapp.istioinaction.io left intact
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```

环境清理

```
# kubectl delete deployment,svc,gateway,virtualservice,destinationrule --all -n istioinaction
```
