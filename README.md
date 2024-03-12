# istio-skupper

## 前置条件

安装ServiceMesh和它所以依赖的Operators，安装Service Interconnect Operator

## ServiceMesh控制面集群范围的实例

```
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: basic
spec:
  addons:
    grafana:
      enabled: true
    jaeger:
      install:
        storage:
          type: Memory
    kiali:
      enabled: true
    prometheus:
      enabled: true
  mode: ClusterWide
  policy:
    type: Istiod
  profiles:
    - default
  proxy:
    injection:
      autoInject: true
  telemetry:
    type: Istiod
  tracing:
    sampling: 10000
    type: Jaeger
  version: v2.4
```

## 纳管项目

```
oc new-project cross-site

oc label ns cross-site istio-injection=enabled
```

## 发布应用

后端

```
oc create deployment hello-world-backend --image quay.io/jonkey/skupper/hello-world-backend:20230225
```

前端

```
oc create deployment hello-world-frontend --image quay.io/jonkey/skupper/hello-world-frontend:20230225

oc expose deployment hello-world-frontend --port 8080 

oc set env deployment hello-world-frontend BACKEND_SERVICE_HOST_="hello-world-backend.cross-site.svc.cluster.local"

oc set env deployment hello-world-frontend BACKEND_SERVICE_PORT_="8080"

oc get route
```

## 配置skupper

创建实例skupper1和skupper2

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: skupper-site
  namespace: cross-site
data:
  console: 'true'
  console-authentication: openshift
  flow-collector: 'true'
  name: skupper1
  router-console: 'false'
  router-mode: interior
  service-controller: 'true'
  service-sync: 'true'
```

调整skupper部署

skupper-router

```
spec:
  template:
    metadata:
      labels:
        maistra.io/expose-route: 'true'
```

 skupper-service-controller

```
spec:
  template:
    metadata:
      labels:
        maistra.io/expose-route: 'true'
      annotations:
        traffic.sidecar.istio.io/excludeOutboundIPRanges: 0.0.0.0/0
```

连接namespace

```
oc project cross-stie
skupper token create ~/secret.token
```

```
skupper link create ~/secret.token
```

## 暴漏后端服务

分别暴漏服务

```
skupper expose deployment/hello-world-backend --port 8080
```

## 暴漏前端页面

访问前端页面

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-world-frontend
spec:
  gateways:
    - hello-world-gateway
  hosts:
    - hello-world-frontend-cross-site.apps.ocp11.jonkey.com
  http:
    - route:
        - destination:
            host: hello-world-frontend.cross-site.svc.cluster.local
            port:
              number: 8080
          weight: 100
```

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: hello-world-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - hosts:
        - hello-world-frontend-cross-site.apps.ocp11.jonkey.com
      port:
        name: http
        number: 80
        protocol: HTTP
```

