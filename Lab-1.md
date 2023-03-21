## ISTIO部署

>前置条件：
> - 已部署OKE集群


### 1. 环境准备

下载istio

```
# curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.17.1 TARGET_ARCH=x86_64 sh -
# export PATH="$PATH:/root/istio-1.17.1/bin"
```

获取OKE集群的访问配置

```
# oci ce cluster create-kubeconfig --cluster-id ocid1.cluster.oc1.ap-tokyo-1.XXXXX --file $HOME/.kube/config --region ap-tokyo-1 --token-version 2.0.0  --kube-endpoint PUBLIC_ENDPOINT
```

验证集群与环境是否就绪

```
# kubectl get node
# istioctl x precheck
```

### 2. 部署istio

创建`namespace`.默认使用`istio-system`名称

```
# kubectl create ns istio-system
```

部署`istio`

```
# istioctl install --set components.cni.enabled=true
```

查看`istio`组件是否正常`Running`

```
# kubectl -n istio-system get pods

```

验证`istio`安装完成

```
# istioctl verify-install
```

安装成功效果如下：

```
1 Istio control planes detected, checking --revision "default" only
✔ ClusterRole: istiod-istio-system.istio-system checked successfully
✔ ClusterRole: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istio-reader-service-account.istio-system checked successfully
✔ Role: istiod-istio-system.istio-system checked successfully
✔ RoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istiod-service-account.istio-system checked successfully
✔ CustomResourceDefinition: wasmplugins.extensions.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: destinationrules.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: envoyfilters.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: gateways.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: proxyconfigs.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: serviceentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: sidecars.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: virtualservices.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadgroups.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: authorizationpolicies.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: peerauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: requestauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: telemetries.telemetry.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: istiooperators.install.istio.io.istio-system checked successfully
✔ HorizontalPodAutoscaler: istiod.istio-system checked successfully
✔ ClusterRole: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRole: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ConfigMap: istio.istio-system checked successfully
✔ Deployment: istiod.istio-system checked successfully
✔ ConfigMap: istio-sidecar-injector.istio-system checked successfully
✔ MutatingWebhookConfiguration: istio-sidecar-injector.istio-system checked successfully
✔ PodDisruptionBudget: istiod.istio-system checked successfully
✔ ClusterRole: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ Role: istiod.istio-system checked successfully
✔ RoleBinding: istiod.istio-system checked successfully
✔ Service: istiod.istio-system checked successfully
✔ ServiceAccount: istiod.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.14.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.14.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.15.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.15.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.16.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.16.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.17.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.17.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istio-validator-istio-system.istio-system checked successfully
✔ ClusterRole: istio-cni.istio-system checked successfully
✔ ClusterRole: istio-cni-repair-role.istio-system checked successfully
✔ ClusterRoleBinding: istio-cni.istio-system checked successfully
✔ ClusterRoleBinding: istio-cni-repair-rolebinding.istio-system checked successfully
✔ ConfigMap: istio-cni-config.istio-system checked successfully
✔ DaemonSet: istio-cni-node.istio-system checked successfully
✔ ServiceAccount: istio-cni.istio-system checked successfully
✔ HorizontalPodAutoscaler: istio-ingressgateway.istio-system checked successfully
✔ Deployment: istio-ingressgateway.istio-system checked successfully
✔ PodDisruptionBudget: istio-ingressgateway.istio-system checked successfully
✔ Role: istio-ingressgateway-sds.istio-system checked successfully
✔ RoleBinding: istio-ingressgateway-sds.istio-system checked successfully
✔ Service: istio-ingressgateway.istio-system checked successfully
✔ ServiceAccount: istio-ingressgateway-service-account.istio-system checked successfully
Checked 15 custom resource definitions
Checked 2 Istio Deployments
✔ Istio is installed and verified successfully
```
