```
# 下载官方coredns示例
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed

wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/deploy.sh

# 修改dns相关配置
# podip网段:20.0.0.0/16
# svc网段:10.253.0.0/16 (10.253.0.250分给了dnsserver 10.253.0.1默认分给了kubernetes的集群内ip)
bash deploy.sh -r 20.0.0.0/16 -i 10.253.0.250 > coredns.yaml

sed -i '/image/ s/1.5.0/1.3.1/g' coredns.yaml

sed -i '/ready/d' coredns.yaml

sed -i '/readinessProbe:/,+4d' coredns.yaml

```

`注意事项:`

- 1. kubelet的配置中需要制定dns为:10.253.0.250
- 2. k8s的证书请求中需要配置10.253.0.1(因为coredns在给集群配置dns的时候需要调用kubernetes的api,即10.253.0.1:443)
