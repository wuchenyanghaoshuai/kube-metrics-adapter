# kube-metrics-adapter
使用banzaicloud的kube-metrics-adapter
https://github.com/banzaicloud/kube-metrics-adapter
metrics-adapter比较prometheus-adapter区别很大,传统的prometheus-adapter是修改他的configmap配置文件,把语句写进去以后,然后重启该pod才可以生效
```
# kubectl edit cm prometheus-adapter -n kube-system
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: prometheus-adapter
    chart: prometheus-adapter-v0.1.2
    heritage: Tiller
    release: prometheus-adapter
  name: prometheus-adapter
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
...
```
传统的自定义指标获取方式
 hpa.yaml---->prometheus-adapter---->prometheus
 hpa.yaml里面定义了指标,这个指标是在prometheus-adapter的comfigmap里面定义的(注意上面有个as),然后这个再去从porometheus里面去获取as后的指标
```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: metrics-app-hpa 
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: metrics-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 800m   # 800m 即0.8个/秒
```

现在这个就是kube-metrics-adapter就是直接在hpa.yaml里面直接写promesql
```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  annotations:
    metric-config.external.prometheus-query.prometheus/processed-requests-per-second: |
       (round(sum(irate(nginx_server_requests{code="2xx"}[2m])) by (pod) ,1) )/2
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: centos-nginx-metrics
  metrics:
    - type: External
      external:
        metric:
          name: prometheus-query
          selector:
            matchLabels:
              query-name: processed-requests-per-second
        target:
          type: AverageValue
          averageValue: 1k
```
