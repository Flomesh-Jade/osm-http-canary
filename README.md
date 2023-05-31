使用flomesh-osm实现东西向灰度发布

# 部署demo服务

部署服务端

```bash
kubectl create namespace pipy
osm namespace add pipy
kubectl apply -n pipy -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/traffic-split-v4/pipy-ok.pipy.yaml
```

部署客户端

```bash
kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/traffic-split-v4/curl.curl.yaml
```
等待依赖的 POD 正常启动

```bash
kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok --timeout=180s
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
```

# 配置主版本服务流量策略

HTTPRouteGroup配置

```bash
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: pipy-ok-service
spec:
  matches:
  - name: default
    methods:
    - GET
EOF
```

主版本服务v1，TrafficTarget配置

```bash
cat <<EOF | kubectl apply -n pipy -f -
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-access-pipy-ok-v1
spec:
  destination:
    kind: ServiceAccount
    name: pipy-ok-v1
    namespace: pipy
  rules:
  - kind: HTTPRouteGroup
    name: pipy-ok-service
    matches:
    - default
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

主版本服务可用性访问测试

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok-v1.pipy:8080
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080
(只转发给主版本v1)
```

# 配置灰度版本流量策略

HTTPRouteGroup配置，加入v2灰度条件配置

```bash
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: pipy-ok-service-v2
spec:
  matches:
  - name: v2
    headers: 
    - "version" : "v2"
    methods:
    - GET
EOF
```

灰度版本服务v2，TrafficTarget配置：

```bash
cat <<EOF | kubectl apply -n pipy -f -
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-access-pipy-ok-v2
spec:
  destination:
    kind: ServiceAccount
    name: pipy-ok-v2
    namespace: pipy
  rules:
  - kind: HTTPRouteGroup
    name: pipy-ok-service-v2
    matches:
    - v2
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

设置灰度策略

```bash
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: split.smi-spec.io/v1alpha4
kind: TrafficSplit
metadata:
  name: pipy-ok-split-v1
spec:
  service: pipy-ok
  matches:
  - kind: HTTPRouteGroup
    name: pipy-ok-service
  backends:
  - service: pipy-ok-v1
    weight: 100
EOF

cat <<EOF | kubectl apply -n pipy -f -
apiVersion: split.smi-spec.io/v1alpha4
kind: TrafficSplit
metadata:
  name: pipy-ok-split-v2
spec:
  service: pipy-ok
  matches:
  - kind: HTTPRouteGroup
    name: pipy-ok-service-v2
  backends:
  - service: pipy-ok-v2
    weight: 100
EOF

```

# 灰度访问测试

```bash
curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080 -H "version:v2"
```
