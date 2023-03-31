## 故障注入与重试

### 1. 故障注入

这里通过脚本实现对catalog服务注入`500`故障


```
# cat chaos.sh

#!/usr/bin/env bash

if [ $1 == "500" ]; then

    POD=$(kubectl get pod | grep catalog | awk '{ print $1 }')
    echo $POD

    for p in $POD; do
        if [ ${2:-"false"} == "delete" ]; then
            echo "Deleting 500 rule from $p"
            kubectl exec -c catalog -it $p -- curl  -X POST -H "Content-Type: application/json" -d '{"active":
        false,  "type": "500"}' localhost:3000/blowup
        else
            PERCENTAGE=${2:-100}
            kubectl exec -c catalog -it $p -- curl  -X POST -H "Content-Type: application/json" -d '{"active":
            true,  "type": "500",  "percentage": '"${PERCENTAGE}"'}' localhost:3000/blowup
            echo ""
        fi
    done
fi
```

针对访问`catalog`服务的请求，`100%`响应`500`故障

```
# ./chaos.sh 500 100

catalog-5c7f8f8447-n996v
blowups=[object Object]
```

验证注入的故障是否生效

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
< HTTP/1.1 500 Internal Server Error
< date: Thu, 09 Mar 2023 06:14:36 GMT
< content-length: 29
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 24
< server: istio-envoy
<
* Connection #0 to host webapp.istioinaction.io left intact
error calling Catalog service

```

### 2. 重试机制

为了演示 `Istio` 为应用程序自动执行重试的能力，这里将`catalog`服务配置为在被调用时产生`50%` 的错误

```
# ./chaos.sh 500 50
```

配置重试策略,一旦遇到`5xx`错误即开始重试，重试次数为3次，单次重试超时时间为2`s`.

```
# cat catalog-virtualservice.yaml

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
    retries:
      attempts: 3
      retryOn: 5xx
      perTryTimeout: 2s

# kubectl apply -f catalog-virtualservice.yaml
```

验证重试机制的效果，可以看到基于重试策略，`5xx`异常明显减少。

```
# while true; do curl -I http://webapp.istioinaction.io/api/catalog; sleep 2; done
HTTP/1.1 200 OK
content-length: 357
content-type: application/json; charset=utf-8
date: Thu, 09 Mar 2023 06:26:39 GMT
x-envoy-upstream-service-time: 31
server: istio-envoy

HTTP/1.1 200 OK
content-length: 357
content-type: application/json; charset=utf-8
date: Thu, 09 Mar 2023 06:26:41 GMT
x-envoy-upstream-service-time: 6
server: istio-envoy

HTTP/1.1 200 OK
content-length: 357
content-type: application/json; charset=utf-8
date: Thu, 09 Mar 2023 06:26:43 GMT
x-envoy-upstream-service-time: 6
server: istio-envoy

...
```

详细细节可以通过`Jaeger`进行查看,可以看到当请求失败后，`webapp`服务重试`3`次访问`catalog`服务

![image](https://user-images.githubusercontent.com/4653664/223945297-35a47fbc-40cb-4176-9598-96fddd8bb9e3.png)

> The interval between retries (25ms+) is variable and determined automatically by Istio

最后，清理注入的故障

```
# ./chaos.sh 500 delete
```

