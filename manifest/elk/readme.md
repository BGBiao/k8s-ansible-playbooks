## 使用statefulset部署有状态的es集群

`注意:主动将es集群中的节点分为两种角色，即数据节点和仲裁节点(master-node,data-node)`

`注意:data节点是有状态的，一方面因为数据的有状态性，另外一方面因为数据恢复时的节点顺序性`

`注意:es,kibana,log-pilot,app均在不同的namespace中`

**创建es集群**

```
# 创建es集群(3master,3node)
$ kubectl  apply -f es-cluster.yaml
..

# 查看集群角色
$ kubectl  get pods,svc,ingress -n ns-elastic
NAME                                        READY   STATUS    RESTARTS   AGE
pod/elasticsearch-data-0                    1/1     Running   0          17m
pod/elasticsearch-data-1                    1/1     Running   0          17m
pod/elasticsearch-data-2                    1/1     Running   0          9m6s
pod/elasticsearch-master-6bd7dbcfcb-8zvjj   1/1     Running   0          17m
pod/elasticsearch-master-6bd7dbcfcb-j7ftr   1/1     Running   0          17m
pod/elasticsearch-master-6bd7dbcfcb-sdn6w   1/1     Running   0          17m

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch-data-service   ClusterIP   None             <none>        9200/TCP,9300/TCP   17m
service/elasticsearch-discovery      ClusterIP   10.253.172.144   <none>        9300/TCP            17m
service/elasticsearch-service        ClusterIP   10.253.179.148   <none>        9200/TCP            17m

NAME                                       HOSTS                            ADDRESS   PORTS   AGE
ingress.extensions/elasticsearch-ingress   prod-es-cluster.bgbiao.cn             80      17m

# 查看集群状态
$ curl -H 'host:prod-es-cluster.bgbiao.cn' 172.16.21.26
{
  "name" : "vMyWzRg",
  "cluster_name" : "elasticsearch-cluster",
  "cluster_uuid" : "MlX-u1XhRE-pHymbcjtMtw",
  "version" : {
    "number" : "6.4.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "fe40335",
    "build_date" : "2018-10-30T23:17:19.084789Z",
    "build_snapshot" : false,
    "lucene_version" : "7.4.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}


# 查看集群节点信息
$ curl -H 'host:prod-es-cluster.bgbiao.cn' 172.16.21.26/_cat/nodes?v
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
20.0.87.10           22          93   2    0.03    0.05     0.08 m         -      b6_F86O
20.0.14.8            24          98   2    0.56    0.29     0.18 m         -      FjiKUSt
20.0.38.4            17          91   2    0.05    0.09     0.14 m         *      6omqml8
20.0.14.7            11          98   2    0.56    0.29     0.18 di        -      kJDcJ_r
20.0.38.9            13          91   2    0.05    0.09     0.14 di        -      Cdon-NY
20.0.87.11           10          93   2    0.03    0.05     0.08 di        -      vMyWzRg

```

**创建daemonset类型的log-pilot**

```
$ kubectl create  -f log-polit.yaml 

# 使用了tolerations，所以会在master节点上也调度
$ kubectl  get pods -n kube-system | grep log-pilot
log-pilot-4fvnn                    1/1     Running   0          58s
log-pilot-5f8gr                    1/1     Running   0          58s
log-pilot-6dwfk                    1/1     Running   0          58s
log-pilot-74ct8                    1/1     Running   0          58s
log-pilot-c6jtq                    1/1     Running   0          58s
log-pilot-z9n8w                    1/1     Running   0          58s

```

**创建kibana集群**

```
# 创建kibana集群
$ kubectl apply -f kibana.yaml
deployment.apps/kb-cluster unchanged
service/kb-cluster-svc unchanged
ingress.extensions/prod-kibana.bgbiao.cn unchanged

$ kubectl  get pods | grep kb
kb-cluster-7559d66fc5-jdghh   1/1     Running   0          73m
kb-cluster-7559d66fc5-z4wds   1/1     Running   0          73m

```







**创建测试业务**

```
$ kubectl create  -f app-test.yml

$ kubectl  get pods -n soul-data
NAME                                  READY   STATUS    RESTARTS   AGE
soul-data-progress-54bc86c645-5hdx4   1/1     Running   0          24m
soul-data-progress-54bc86c645-qn5j7   1/1     Running   0          24m

# 当业务容器启动后，相关日志自动写入到es集群中
$ curl -s  prod-es-cluster.bgbiao.cn/_cat/indices?v | grep soul-data
green  open   soul-data-progress-2019.09.10 PJVMOeQuToGUMCz1nalt7g   5   1        251            0    932.2kb          453kb
green  open   soul-data-xxb-2019.09.10      TKhxBDDEQJaSevoebS3RxA   5   1         43            0    195.6kb         97.8kb


```
