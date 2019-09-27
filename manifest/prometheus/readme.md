## prometheus在k8s集群内部的搭建

[prometheus-on-k8s](https://github.com/coreos/kube-prometheus)


`注意:prometheus-on-k8s采用operators方式来部署整个集群`

**服务部署**

```
# prometheus在k8s集群上搭建
$ git clone https://github.com/coreos/kube-prometheus.git
$ cd kube-prometheus
$ kubectl apply -f manifests

# 查看相关服务

kubectl  get all -n monitoring
NAME                                       READY   STATUS    RESTARTS   AGE
pod/alertmanager-main-0                    2/2     Running   0          9m40s
pod/alertmanager-main-1                    2/2     Running   0          9m40s
pod/alertmanager-main-2                    2/2     Running   0          9m40s
pod/grafana-57bfdd47f8-xfvks               1/1     Running   0          9m44s
pod/kube-state-metrics-757c97c586-rxq8k    4/4     Running   0          2m45s
pod/node-exporter-7g5tz                    2/2     Running   0          9m44s
pod/node-exporter-9x6gz                    2/2     Running   0          9m44s
pod/node-exporter-b8xtv                    2/2     Running   0          9m44s
pod/node-exporter-kxgcj                    2/2     Running   0          9m44s
pod/node-exporter-lmxq4                    2/2     Running   0          9m44s
pod/node-exporter-lwkhs                    2/2     Running   0          9m44s
pod/prometheus-adapter-668748ddbd-5vz8b    1/1     Running   0          9m44s
pod/prometheus-k8s-0                       3/3     Running   1          9m30s
pod/prometheus-k8s-1                       3/3     Running   1          9m30s
pod/prometheus-operator-5b6cfc5846-xgsw6   1/1     Running   0          9m45s


NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       ClusterIP   10.253.3.109     <none>        9093/TCP                     9m46s
service/alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   9m40s
service/grafana                 ClusterIP   10.253.183.76    <none>        3000/TCP                     9m45s
service/kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            9m45s
service/node-exporter           ClusterIP   None             <none>        9100/TCP                     9m45s
service/prometheus-adapter      ClusterIP   10.253.248.193   <none>        443/TCP                      9m45s
service/prometheus-k8s          ClusterIP   10.253.42.237    <none>        9090/TCP                     9m45s
service/prometheus-operated     ClusterIP   None             <none>        9090/TCP                     9m30s
service/prometheus-operator     ClusterIP   None             <none>        8080/TCP                     9m46s

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   6         6         6       6            6           kubernetes.io/os=linux   9m45s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana               1/1     1            1           9m45s
deployment.apps/kube-state-metrics    1/1     1            1           9m45s
deployment.apps/prometheus-adapter    1/1     1            1           9m45s
deployment.apps/prometheus-operator   1/1     1            1           9m46s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-57bfdd47f8               1         1         1       9m45s
replicaset.apps/kube-state-metrics-757c97c586    1         1         1       2m45s
replicaset.apps/kube-state-metrics-77467ddf9b    0         0         0       9m45s
replicaset.apps/prometheus-adapter-668748ddbd    1         1         1       9m45s
replicaset.apps/prometheus-operator-5b6cfc5846   1         1         1       9m46s

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     9m40s
statefulset.apps/prometheus-k8s      2/2     9m30s


```

**服务配置**

`注意:prometheus默认数据写入本地，但是选择了容器化部署后相关归档数据需要落地存储，官方建议infuxdb`

```
# 修改远端写入存储为influxdb(CREATE DATABASE "prometheus")

# 修改manifests/prometheus-prometheus.yaml配置
  remoteRead:
  - url: http://influxdb:8086/api/v1/prom/read?db=prometheus
  remoteWrite:
  - url: http://influxdb:8086/api/v1/prom/write?db=prometheus

# 更新容器
$ kubectl apply -f prometheus-prometheus.yaml

# 配置ingress
$ cat ingress-all.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-k8s
  namespace: monitoring
spec:
  rules:
  - host: prod-prometheus.bgbiao.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus-k8s
          servicePort: 9090
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: alertmanager-main
  namespace: monitoring
spec:
  rules:
  - host: prod-alertmanager.bgbiao.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: alertmanager-main
          servicePort: 9093
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
spec:
  rules:
  - host: prod-grafana.bgbiao.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000

```

**kube-scheduler服务监控**

由于`kube-scheduler`服务启动为服务器进程，没有在k8s集群内运行，需要将服务加入到`Prometheus Operator`监控中就需要使用Endpoints方式来来集群内部访问。

定义了两种资源对象，分别是Service和Endpoints。其中Service的定义并没有使用标签选择器，而在后面定义了一个与Service同名的Endpoints，以使得它们能自动关联。Endpoints的subsets中指定了需要连接的外部服务器的IP和端口。

`注意:每个服务都需要暴露相关的服务地址`

```
# cat endpoint-KubeScheduler.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: kube-scheduler
  namespace: kube-system
  labels:
    k8s-app: kube-scheduler
subsets:
- addresses:
  - ip: 10.10.49.78 # 这里为服务节点IP list
  - ip: 10.10.49.77
  - ip: 10.10.49.79
  ports:
  - port: 10251
    name: http-metrics
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
    protocol: TCP

# 创建服务并验证状态
kubectl apply -f endpoint-KubeScheduler.yaml
kubectl get endpoints kube-scheduler -n kube-system
kubectl describe endpoints kube-scheduler -n kube-system

```
**prometheus服务监控**

```
# prometheus服务监控
$ cat prometheus-serviceMonitorKubeScheduler.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: http-metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-scheduler

# 创建服务
$ kubectl apply -f prometheus-serviceMonitorKubeScheduler.yaml
```

**Kube-Controller-Manager服务监控**

```
# 创建kube-controller-manager监控
$ cat endpoint-KubeControllerManager.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: kube-controller-manager
  namespace: kube-system
  labels:
    k8s-app: kube-controller-manager
subsets:
- addresses:
  - ip: 10.10.49.78
  - ip: 10.10.49.77
  - ip: 10.10.49.79
  ports:
  - port: 10252
    name: http-metrics
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP

$ kubectl apply -f endpoint-KubeControllerManager.yaml
$ kubectl get endpoints kube-controller-manager -n kube-system
$ kubectl describe endpoints kube-controller-manager -n kube-system
 
```

**prometheus服务自身监控**

```
# cat prometheus-serviceMonitorKubeControllerManager.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    metricRelabelings:
    - action: drop
      regex: etcd_(debugging|disk|request|server).*
      sourceLabels:
      - __name__
    port: http-metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-controller-manager

$ kubectl apply -f prometheus-serviceMonitorKubeControllerManager.yaml
 

```

**ectd服务监控**

kube-scheduler和kube-controller-manager服务的metrics可以直接访问，ectd服务的metrics接口需要进行认证，所以和以上两个服务比较多了配置证书的步骤。

创建`Secret`对象保存要使用到的密钥文件:

```
# 文件对应内容使用base64加密后填充
$ cat etcd-certs.yaml
apiVersion: v1
data:
  etcd-client-key.pem: xxxxxxxx
  etcd-client.pem: xxxxxxxx
  etcd-ca.pem: xxxxxxxx
kind: Secret
metadata:
  name: etcd-certs
  namespace: monitoring
type: Opaque

# 创建并验证
$ kubectl apply -f etcd-certs.yaml
$ kubectl get Secret etcd-certs -n monitoring
$ kubectl describe  Secret etcd-certs -n monitoring

# 创建endpoint和service
# cat endpoint-Ectd.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd
  namespace: kube-system
  labels:
    k8s-app: etcd
subsets:
- addresses:
  - ip: 10.10.49.78
  - ip: 10.10.49.77
  - ip: 10.10.49.79
  ports:
  - port: 2379
    name: http-metrics
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: etcd
  labels:
    k8s-app: etcd
spec:
  ports:
  - name: http-metrics
    port: 2379
    targetPort: 2379
    protocol: TCP

# 创建etcd的ServiceMonitor对象

# cat prometheus-serviceMonitorEtcd.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd
  namespace: monitoring
  labels:
    k8s-app: etcd
spec:
  jobLabel: k8s-app
  endpoints:
  - port: http-metrics
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/etcd-ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/etcd-client.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd-client-key.pem
      insecureSkipVerify: true
  selector:
    matchLabels:
      k8s-app: etcd
  namespaceSelector:
    matchNames:
    - kube-system

# 把需要用到证书挂载到prometheus的pod中
# cat prometheus-prometheus.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
  baseImage: quay.io/prometheus/prometheus
  nodeSelector:
    kubernetes.io/os: linux
  podMonitorSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.11.0
  # 添加一下两行配置
  secrets:
  - etcd-certs

# 应用配置生效
$ kubectl apply -f prometheus-prometheus.yaml

```

`注意:分别访问域名查看监控targets配置，grafana监控信息，alertmanage报警信息验证服务是否正常。`






**相关问题解决**

```
$ kubectl  logs prometheus-adapter-668748ddbd-4n4nx -n monitoring
I0910 03:04:23.778589       1 adapter.go:91] successfully using in-cluster auth
F0910 03:04:24.396164       1 adapter.go:252] unable to install resource metrics API: cluster doesn't provide requestheader-client-ca-file

# 原因是api-server没有增加requestheader-client-ca-file

$ 增加如下参数
--requestheader-client-ca-file={{ ssl_dir }}ca.pem \
--requestheader-allowed-names=aggregator \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User"
 

#  但是发现其他服务都没有起来，比如node-expoter还有grafana相关的服务没有启动
$  kubectl  get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
prometheus-adapter-668748ddbd-q7tqd   1/1     Running   0          2m30s

# 查看prometheus-operator
$ kubectl api-versions| grep monitoring
monitoring.coreos.com/v1

# 查看prometheus-operator支持的资源类型
$ kubectl get --raw "/apis/monitoring.coreos.com/v1"|jq .

# 查看k8s集群内部服务
$ kubectl get --raw "/apis/monitoring.coreos.com/v1/servicemonitors"|jq .

# 查看daemonset的日志
$ kubectl  describe  daemonset node-exporter -n monitoring
...

ds "node-expoter-cndf5" is forbidden: SecurityContext.RunAsUser is forbidden

# 原因是因为kube-apiserver增加了SecurityContextDeny这个准入控制规则,删掉即可

```
