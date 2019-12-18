## Kuboard 快速部署指南

```
# 快速部署Kuboard
$ kubectl apply kuboard.yaml

# 在Kuboard中配置的是nodeport(32567) 访问后输入token
# 获取token(sysadmin权限)
$ kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d

```
