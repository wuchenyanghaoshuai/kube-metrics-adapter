```
# 安装kube-metrics-adapter
git clone https://github.com/banzaicloud/kube-metrics-adapter.git
cd kube-metrics-adapter/deploy/charts
#helm 要使用v3版本以上
helm -n kube-system install asm-custom-metrics ./kube-metrics-adapter  --set prometheus.url=http://prometheus.istio-system.svc:9090
注意promethues的位置，如果你按照之前我的文档来做的话，英国是prometheus-k8s.monitoring.svc:9090
#安装完成以后
kubectl get pods -n kube-system|grep adapter
kubectl api-versions |grep "autoscaling/v2beta"
#预计输出
autoscaling/v2beta
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .
#预计输出
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "external.metrics.k8s.io/v1beta1",
  "resources": []
}
```
#vim hpa.yaml
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
# vim deploy.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-nginx-metrics
  labels:
    app: centos-nginx-metrics
spec:
  selector:
    matchLabels:
      app: centos-nginx-metrics
  replicas: 1
  template:
    metadata:
      labels:
        app: centos-nginx-metrics
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9913"
        prometheus.io/path: /metrics
    spec:
      containers:
      - name: centos-nginx-metrics
        image: wuchenyanghaoshuai/centos-nginx-metrics:v2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: 128Mi
            cpu: 0.2
          limits:
            memory: 222Mi
            cpu: 0.3
        volumeMounts:
        - name: logs
          mountPath: /opt/logs
      volumes:
      - name: logs
        nfs:
          server: 192.168.1.5
          path: /nfs
```
# 如果你没有volumes的话，直接去掉就可以了
# 然后查看pod的地址,进行ab压测
```
ab -n 1000000 -c 100 http://10.244.2.13/index.html
```
这个时候就可以配合着grafana来进行看了，看pod的cpu状态

#压测过程中 kubectl top pods
#然后get pods的时候 注意查看一下是否进行扩容了
