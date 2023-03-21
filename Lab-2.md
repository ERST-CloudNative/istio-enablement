## 可观测性组件部署与验证

### 1. 准备工作

创建namespace

```
# kubectl create namespace istioinaction
```
设置上下文，即当前后续操作会在`istioinaction` `namespace`下进行

```
# kubectl config set-context $(kubectl config current-context)  --namespace=istioinaction
```
使能自动注入功能

```
# kubectl label namespace istioinaction istio-injection=enabled
```

### 2. 应用部署

部署`catalog`服务

```
# kubectl apply -f catalog.yaml
```

验证应用正常工作

```
# kubectl run -i -n default --rm --restart=Never dummy --image=curlimages/curl --command -- sh -c 'curl -s http://catalog.istioinaction/items/1'

{
  "id": 1,
  "color": "amber",
  "department": "Eyewear",
  "name": "Elinor Glasses",
  "price": "282.00"
}
pod "dummy" deleted
```

部署`webapp`服务

```
# kubectl apply -f webapp.yaml

# kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
catalog-5c7f8f8447-n996v   2/2     Running   0          2m56s
webapp-8dc87795-sqlzk      2/2     Running   0          63s

# kubectl run -i -n default --rm --restart=Never dummy --image=curlimages/curl --command -- sh -c 'curl -s http://webapp.istioinaction/api/catalog/items/1'

{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"}
pod "dummy" deleted

```

### 3. GateWay配置

应用相关配置

```
# kubectl apply -f grafana-gateway.yaml -f prometheus-gateway.yaml -f kiali-gateway.yaml -f jaeger-gateway.yaml
```

通过istio入口网关查看当前路由配置

```
# istioctl proxy-config routes deploy/istio-ingressgateway.istio-system -n istio-system
NAME          DOMAINS                         MATCH                  VIRTUAL SERVICE
http.8080     grafana.istioinaction.io        /*                     grafana-virtualservice.istio-system
http.8080     jaeger.istioinaction.io         /*                     jaeger-virtualservice.istio-system
http.8080     kiali.istioinaction.io          /*                     kiali-virtualservice.istio-system
http.8080     prometheus.istioinaction.io     /*                     prometheus-virtualservice.istio-system
http.8080     webapp.istioinaction.io         /*                     webapp-virtualservice.istioinaction
              *                               /healthz/ready*
              *                               /stats/prometheus*
```

在本地环境下配置以下自定义主机名和`IP`对应关系

获取入口网关的`IP`地址
```
# kubectl -n istio-system get svc istio-ingressgateway  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
141.147.176.63
```

添加以下配置到`/etc/hosts`文件中

```
141.147.176.63 grafana.istioinaction.io
141.147.176.63 prometheus.istioinaction.io
141.147.176.63 kiali.istioinaction.io
141.147.176.63 jaeger.istioinaction.io
141.147.176.63 webapp.istioinaction.io
```

### 4. 验证

模拟用户访问，更好地查看`Istio`周边组件的效果

```
# while true; do curl http://webapp.istioinaction.io/api/catalog; sleep .5; done
```

浏览器访问`grafana.istioinaction.io`

![image](https://user-images.githubusercontent.com/4653664/223925940-af3b4a4f-b5f4-4d2e-810f-029787dfc0cf.png)

浏览器访问`prometheus.istioinaction.io`

![image](https://user-images.githubusercontent.com/4653664/223925977-c2fdf872-0e73-436c-9eff-d3d29aa2fa6a.png)

浏览器访问`kiali.istioinaction.io`

![image](https://user-images.githubusercontent.com/4653664/223926058-9d28f9ae-ae81-47e7-b0c7-22c2f944ed4e.png)

浏览器访问`jaeger.istioinaction.io`

![image](https://user-images.githubusercontent.com/4653664/223926118-52b4bae7-4b02-4b09-b6f4-1e92b24d4a5b.png)

浏览器访问`webapp.istioinaction.io`

![image](https://user-images.githubusercontent.com/4653664/223926190-5b20eb4f-7c3c-4665-98d1-bb209586e28a.png)
