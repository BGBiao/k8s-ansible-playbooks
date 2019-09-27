## 使用ansible快速构建一套高可用的k8s集群

### 基本依赖包和相关包下载

**相关依赖资源下载**

`注意:k8s相关资源需要科学上网才能下载`

[k8s-v1.16](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.16.md)

[k8s-v1.16-amd64](https://dl.k8s.io/v1.16.0/kubernetes-server-linux-amd64.tar.gz)

[dokcer二进制包清单](https://download.docker.com/linux/static/stable/x86_64/)

[etcd-v3.4.1](https://github.com/etcd-io/etcd/releases)

[etcd-v3.4.1-amd64](https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-amd64.tar.gz)

[flanneld](https://github.com/coreos/flannel/releases)

`k8s镜像包:`

百度云盘备份的k8s-v1.16.0的二进制包:`链接:https://pan.baidu.com/s/16jiXa0LZZhgeYkwSs8tv9g  密码:0rl2`

```
# 使用上述百度云盘备份的k8s包地址进行下载
➜  ls k8s-server-amd64-v1.16.0.tar.gz
k8s-server-amd64-v1.16.0.tar.gz
# 使用sha512sum计算压缩包的hash值,和官方k8s-v1.16中的二进制包hash值进行对比(一致则说明包没有被修改过)
➜  sha512sum k8s-server-amd64-v1.16.0.tar.gz
a6bdac1eba1b87dc98b2bf5bf3690758960ecb50ed067736459b757fca0c3b01dd01fd215b4c06a653964048c6a81ea80b61ee8c7e4c98241409c091faf0cee1  k8s-server-amd64-v1.16.0.tar.gz
```

**注意:由于需要部署集群环境，并且以后有扩容需求，一般会建议将相关依赖包下载到本地，并使用http发布出去供内网下载**

```
$ tree -L  1 .
.
├── docker-19.03.1.tgz
├── etcd-v3.4.1-linux-amd64.tar.gz
├── flannel-v0.11.0-linux-amd64.tar.gz
├── k8s-server-amd64-v1.16.0.tar.gz
├── nohup.out
├── readme.md
└── runhttp.sh

$ curl -s http://172.29.202.140:8881
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="docker-19.03.1.tgz">docker-19.03.1.tgz</a>
<li><a href="etcd-v3.4.1-linux-amd64.tar.gz">etcd-v3.4.1-linux-amd64.tar.gz</a>
<li><a href="flannel-v0.11.0-linux-amd64.tar.gz">flannel-v0.11.0-linux-amd64.tar.gz</a>
<li><a href="k8s-server-amd64-v1.16.0.tar.gz">k8s-server-amd64-v1.16.0.tar.gz</a>
<li><a href="nohup.out">nohup.out</a>
<li><a href="readme.md">readme.md</a>
<li><a href="runhttp.sh">runhttp.sh</a>
</ul>
<hr>
</body>
</html>

```

**ansible基本环境配置**


`注意:本系列文档统一使用ansible进行配置和部署管理`

```
# 安装ansible
$ pip install ansible

# 使用ansible生成ssh秘钥
$ ansible -i hosts all -m shell -a 'ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa' -k
# 将sshkey添加到本地认证文件
$ ssh-keyscan `cat hosts | grep 1` >> /root/.ssh/known_hosts
# 将ansible本地公钥认证发放给各个节点(ssh-copy-id类似)
$ ansible -i hosts all -m authorized_key -a "user=root state=present key=\"{{ lookup('file', '/root/.ssh/id_rsa.pub') }}\"" -k


# 免密验证
# 查看内核和getenforce
$ ansible -i hosts all -m shell -a "uname -r && getenforce "
172.29.202.145 | CHANGED | rc=0 >>
3.10.0-862.el7.x86_64
Disabled

172.29.202.155 | CHANGED | rc=0 >>
3.10.0-862.el7.x86_64
Disabled

172.29.202.151 | CHANGED | rc=0 >>
3.10.0-862.el7.x86_64
Disabled
```

### 主机初始化

`注意:需要确保主机的时间和ntp是同步的(使用阿里云ecs这些是基础保障)`

- 1. 升级内核(可选)
- 2. 格式化数据盘(推荐)
- 3. 更新内核参数
- 4. 关闭非必要服务(setenforce,firewalld)
- 5. 加载内部必要模块

**格式化数据盘**

`注意:此步骤可直接略过，但通常生产或者需要维护稳定状态的的集群，建议设置成独立存储`

```
# 格式化数据盘(docker数据存储)
## 使用独立的磁盘/dev/vdb创建分区或者在已有磁盘上创建独立分区/dev/vda2(使用剩余全部资源)
# 将/dev/vdb正块盘进行分区
$ ansible -i hosts all -m shell -a "echo -e 'm\nn\np\n\n\n\nt\n83\np\nw\nq\n' | fdisk /dev/vdb && partprobe && mkfs.ext4 /dev/vdb1 && mkdir /data && mount /dev/vdb1 /data"

# 或者将/dev/vda 剩下的空间分给/dev/vda2
$ ansible -i hosts all -m shell -a "echo -e 'm\nn\np\n2\n\n\nt\n2\n83\np\nw\nq\n' | fdisk /dev/vda && partprobe && mkfs.ext4 /dev/vda2 && mkdir /data -p && mount /dev/vda2 /data"

# 查看磁盘挂载情况:
$ ansible -i hosts master -m shell -a "df -H | grep /data"
172.29.202.145 | CHANGED | rc=0 >>
/dev/vda2       254G   63M  241G    1% /data

172.29.202.155 | CHANGED | rc=0 >>
/dev/vda2       254G   63M  241G    1% /data

172.29.202.151 | CHANGED | rc=0 >>
/dev/vda2       254G   63M  241G    1% /data

# 设置磁盘自动挂载(指定分区)
ansible -i hosts all -m shell -a "echo \$(blkid /dev/vda2 | awk  '{print \$2}') /data ext4 defaults 1 1 >> /etc/fstab && mount -a"

```

**关闭非必要服务以及os初始化**

`注意:` 当然也可以更新下内核(指定的生产环境下建议打必要的kernel patch)

```
# 执行主机初始化相关操作(安装基础软件包,关闭非必要服务,初始化目录,更新内核参数,自动加载必要内核模块,检查内核模块)
$ ansible-playbook -i hosts -e host=all os-init.yml
....
....
TASK [debug info] *****************************************************************************************************
ok: [172.29.202.145] => {
    "msg": "br_netfilter           22256  0 \nbridge                146976  1 br_netfilter\nip_vs                 141432  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr"
}
ok: [172.29.202.155] => {
    "msg": "br_netfilter           22256  0 \nbridge                146976  1 br_netfilter\nip_vs                 141432  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr"
}
ok: [172.29.202.151] => {
    "msg": "br_netfilter           22256  0 \nbridge                146976  1 br_netfilter\nip_vs                 141432  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr"
}
....

```

###  K8S集群部署

`注意:在不是k8s集群时理应对集群做一个初始的规划`

ip | 节点角色 | 部署组件 | 备注
--- | --- | --- | ---
172.29.202.145 | master | kube-apiserver,kube-controller-manager,kube-scheduler,etcd,docker,flanneld,kubelet,kube-proxy |
172.29.202.155 | master | kube-apiserver,kube-controller-manager,kube-scheduler,etcd,docker,flanneld,kubelet,kube-proxy |
172.29.202.151 | master | kube-apiserver,kube-controller-manager,kube-scheduler,etcd,docker,flanneld,kubelet,kube-proxy |

`注意:k8s整个生态组件有一定的依赖关系`

- etcd: 无依赖
- flanneld: 依赖etcd(网络组件可选)
- docker: 依赖flanneld
- kube-apiserver: 依赖etcd
- kube-controller-manager
- kube-scheduler
- kubelet: 依赖etcd和k8s-master
- kube-proxy

#### tls/ssl证书构建

`注意:证书在任意节点生成即可. 这里使用bootstrap的证书轮转，因此不需要给每个节点角色都创建证书`

```
# 下载证书相关文件(工具包,可以在ansible主机上统一生成)
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod a+x *amd64

$ mv cfssl_linux-amd64 /usr/sbin/cfssl
$ mv cfssljson_linux-amd64 /usr/sbin/cfssljson
$ mv cfssl-certinfo_linux-amd64 /usr/sbin/cfssl-certinfo
```

**0. 创建根证书(CA)**

`注意:k8s集群和etcd集群公用一个ca进行证书颁发,后续创建的所有证书都由根证书签名`

```
# 创建ca证书配置文件
$ cat ca-ssl/ca-config.json
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
## signing 表示该证书可用于签名其它证书，生成的ca.pem证书找中CA=TRUE
## server auth 表示client可以用该证书对server提供的证书进行验证
## client auth 表示server可以用该证书对client提供的证书进行验证


# 创建证书签名请求文件
$ cat ca-ssl/ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
 }
}

## CN CommonName,kube-apiserver从证书中提取该字段作为请求的用户名(User Name)，浏览器使用该字段验证网站是否合法
## O Organization,kube-apiserver 从证书中提取该字段作为请求用户和所属组(Group)
## kube-apiserver将提取的User、Group作为RBAC授权的用户和标识

# 生成证书和私钥
## 生成ca证书和秘钥(ca-key.pem,ca.pem)
## ca证书和秘钥以及ca的配置文件在每个节点都需要进行验证
$ cd ca-ssl
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
....
....
$ ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

```

**1. 创建etcd证书**

```
# 创建etcd证书和私钥
$ cat etcd-ssl/etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.29.202.145",
    "172.29.202.155",
    "172.29.202.151"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

# 使用前面创建的ca证书私钥和ca配置文件创建etcd证书和私钥

$ cfssl gencert -ca=../ca-ssl/ca.pem -ca-key=../ca-ssl/ca-key.pem -config=../ca-ssl/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
...
...
$ ls
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

## 创建证书时需要指定CA证书中的profile(用来指定CA)

```


**2. 创建k8s-master请求证书**

`注意:为了保证k8s-master的真正高可用，需要给master-api进行高可用配置`

```
# 指定了master的三台机器,虚拟的slb地址(140),以及未来高可用的域名
# 10.253.0.1 为整个k8s集群的默认service地址，很多插件服务需要使用这个地址调用k8s-master-api
$ cat k8s-ssl/k8s-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "172.29.202.145",
      "172.29.202.155",
      "172.29.202.151",
      "172.29.202.140",
      "k8s-master-api.bgbiao.cn",
      "10.253.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
 
# 生成master的证书和秘钥
$ cd k8s-ssl
$ cfssl gencert -ca=../ca-ssl/ca.pem -ca-key=../ca-ssl/ca-key.pem -config=../ca-ssl/ca-config.json -profile=kubernetes k8s-csr.json | cfssljson -bare k8s
....
....
$ ls
k8s.csr  k8s-csr.json  k8s-key.pem  k8s.pem

```


**3. 创建k8s-admin请求证书**

`注意:k8s-admin证书为管理员证书，主要配合k8s的rolebinding来生成Kubeconfig`

```
```

**4. 创建kube-proxy证书**

```
$ cat kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

$ cfssl gencert -ca=../ca-ssl/ca.pem -ca-key=../ca-ssl/ca-key.pem -config=../ca-ssl/ca-config.json -profile=kubernetes kube-proxy-csr.json  | cfssljson -bare kube-proxy

$ ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem


```


#### etcd集群安装

`注意:使用配置好的etcd-install.yaml进行快速部署etcd集群,需要修改yaml中的etcd_url变量以及etcd_version变量,vars/varsfile.yml中的download_url变量修改为内网包存储地址`

[etcd-v3.4.1](https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-amd64.tar.gz)

[etcd-v3.3.11](https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.3.11-linux-amd64.tar.gz)

`注意:etcdv3.4版本之后默认使用了V3的API接口，相对应的很多启动参数也有变化，因此当前集群仍然使用etcdv3.3.11进行部署`

```
## etcd集群的每个节点都有一个节点id,因此需要使用该种方式同步三次配置文件
$  ansible-playbook -i hosts -e host=master[0] -e nodename=etcd01 etcd-install.yml
$  ansible-playbook -i hosts -e host=master[1] -e nodename=etcd02 etcd-install.yml
$  ansible-playbook -i hosts -e host=master[2] -e nodename=etcd03 etcd-install.yml

# 启动etcd集群
$ ansible -i hosts master -m shell -a 'systemctl daemon-reload && systemctl start etcd && systemctl enable etcd' -f 3
...
...

# 模拟测试集群
$ ansible -i hosts master -m shell -a 'etcdctl --ca-file=/data/etcd/ssl/ca.pem --cert-file=/data/etcd/ssl/etcd.pem --key-file=/data/etcd/ssl/etcd-key.pem --endpoints="https://172.29.202.145:2379" cluster-health'
172.29.202.151 | CHANGED | rc=0 >>
member 49c4877710e02605 is healthy: got healthy result from https://172.29.202.155:2379
member 605a722396df1a6e is healthy: got healthy result from https://172.29.202.151:2379
member 6514badf62176cfd is healthy: got healthy result from https://172.29.202.145:2379
cluster is healthy
......

```

#### flanneld组件部署

`注意:docker依赖flanneld,flanneld依赖etcd`

`注意:当前flanneld版本为0.11，该版本不支持etcd的v3接口`

```
# 在etcd集群中提前创建pod网络(默认前缀为/coreos.com)
$ export FLANNEL_ETCD_PREFIX="/kubernetes/network"

# 向etcd集群中写入flanneld网络信息(etcd v2接口)
$ etcdctl --ca-file=/data/etcd/ssl/ca.pem --cert-file=/data/etcd/ssl/etcd.pem --key-file=/data/etcd/ssl/etcd-key.pem --endpoints="https://172.29.202.145:2379" set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"20.0.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'


# 使用ansible-playbooks部署etcd集群(需要修改yaml文件的etcd_endpoints变量为etcd集群地址)
$ ansible-playbook -i hosts -e host=all flanneld-install.yml
....
....

# 批量启动flanneld
$ ansible -i hosts all -m shell -a " systemctl daemon-reload && systemctl enable flanneld && systemctl restart flanneld"

# 检查flanneld是否正常启动(每个节点上会创建一个flannel.1的虚拟网卡,从etcd中的子网获取一个24位的网段)
$ ansible -i hosts all -m shell -a " ifconfig flannel.1"


```


#### docker部署

```
# 批量部署docker基础环境
$ ansible-playbook -i hosts -e host=all docker-install.yml

# 启动docker
$ ansible -i hosts all -m shell -a " systemctl daemon-reload && systemctl enable docker && systemctl restart docker "
...

# 查看docker启动状态
# 查看flannel.1和docker0的ip地址即可判断docker是否使用了flannel网络(一个子网地址,一个是子网的第一个ip地址(充当网关))
$ ansible -i hosts all -m shell -a " ifconfig flannel.1 | grep inet && ifconfig docker0 | grep inet"
....
....

```

#### k8s-master部署

- kube-apiserver
- kube-scheduler
- kube-controller-manager

`注意:k8s-master的证书我们前面已经生成了,在k8s-ssl目录`

`注意:k8s里的状态数据使用etcd v3接口存储`

`注意: 需要修改yaml配置中的etcd_endpoints变量`

```
# 生成Bootstrapping配置文件(用来对kubelet第一次授权)
$ echo `head -c 16 /dev/urandom | od -An -t x | tr -d ' '`,kubelet-bootstrap,10001,\"system:kubelet-bootstrap\"  > templates/token.csv
$ cat templates/token.csv
a193d1d3db2080f06b265187c9b9e391,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 使用ansible-playbooks统一部署k8s-master基础环境(下载k8s-server包,拷贝并初始化)
$ ansible-playbook -i hosts -e host=master k8s-master-install.yml
...
...

# 批量启动k8s-master节点全部组件
$ ansible -i hosts master -m shell -a "systemctl daemon-reload && systemctl restart kube-apiserver kube-controller-manager kube-scheduler && systemctl enable kube-apiserver kube-controller-manager kube-scheduler"
....
....

# 确认Master节点是否启动(k8sv1.15之前的版本可以确认master节点是否正常)
$ ansible -i hosts master -m shell -a "kubectl get cs,nodes"
172.16.21.25 | SUCCESS | rc=0 >>
NAME                                 STATUS    MESSAGE             ERROR
componentstatus/scheduler            Healthy   ok
componentstatus/controller-manager   Healthy   ok
componentstatus/etcd-1               Healthy   {"health":"true"}
componentstatus/etcd-0               Healthy   {"health":"true"}
componentstatus/etcd-2               Healthy   {"health":"true"}

# k8sv1.16之后的版本这样启动没有效果(仅name和age)
$ ansible -i hosts master -m shell -a "kubectl get cs,nodes"
172.29.202.155 | CHANGED | rc=0 >>
NAME                                 AGE
componentstatus/scheduler            <unknown>
componentstatus/controller-manager   <unknown>
componentstatus/etcd-2               <unknown>
componentstatus/etcd-0               <unknown>
componentstatus/etcd-1               <unknown>

```


#### 部署k8s-node节点

- docker
- flanneld
- kubelet
- kube-proxy

`注意:通常由于master节点也需要监控，因此上述组件默认在全部节点都会安装，只需要在相应的node节点配置Kubelet和kube-proxy即可`


`注意:部署node节点前，需要在装有Kubectl客户端的机器上创建Kube-bootstrap文件`


**在任意一台有ssl相关证书的节点上生成bootstrap.kubeconfig**

`注意:由于master节点上均有ssl证书和客户端配置，可以选择在任意节点上进行生成`

```
  142  export BOOTSTRAP_TOKEN=`cat /data/kubernetes/cfg/token.csv  | awk -F ',' '{print $1}'`
  143  echo ${BOOTSTRAP_TOKEN}
  144  ifconfig eth0
  145  export KUBE_APISERVER="https://172.29.202.155:6443"
  146  kubectl config set-cluster kubernetes --certificate-authority=/data/kubernetes/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=bootstrap.kubeconfig
  147  kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=bootstrap.kubeconfig
  148   kubectl config set-context default --cluster=kubernetes  --user=kubelet-bootstrap --kubeconfig=bootstrap.kubeconfig
  149  kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
  150  kubectl config set-cluster kubernetes --certificate-authority=/data/kubernetes/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-proxy.kubeconfig
  151  kubectl config set-credentials kube-proxy --client-certificate=/data/kubernetes/ssl/kube-proxy.pem --client-key=/data/kubernetes/ssl/kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
  152  kubectl config set-context default --cluster=kubernetes --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
  153   kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
  154  kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

 1287  ansible -i hosts 172.29.202.155 -m fetch -a "src=bootstrap.kubeconfig dest=./templates/ flat=yes"
 1288  ansible -i hosts 172.29.202.155 -m fetch -a "src=kube-proxy.kubeconfig dest=./templates/ flat=yes"

```

**开始批量部署node**

```
# 注意:由于我们在master节点上部署node组件，其实可以直接下发配置即可
$ ansible-playbook -i hosts -e host=master k8s-slave-install.yml --tags=config
....
....
# 批量启动node节点各个角色
$ ansible -i hosts all -m shell -a "systemctl daemon-reload && systemctl restart kubelet kube-proxy && systemctl enable kubelet kube-proxy && systemctl status kubelet kube-proxy"

# 查看csr请求状态(正处于pending状态) 在任意一个k8s节点查看
$ kubectl  get csr
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-KChgnUues8c8TqOqVlbbYsAELzA8P632vfcGtpPCn1w   63s   kubelet-bootstrap   Pending
node-csr-g7dqjomg_sXtldCQA_Dgo4cYsWLH49VL2Z79Cj_rlLU   63s   kubelet-bootstrap   Pending
node-csr-o0Swsg7I14BFAggupTyVq0yL9quB_yc0z4HsdsAZ2Xw   63s   kubelet-bootstrap   Pending

# 手动接受csr请求
$ kubectl certificate approve node-csr-KChgnUues8c8TqOqVlbbYsAELzA8P632vfcGtpPCn1w
certificatesigningrequest.certificates.k8s.io/node-csr-KChgnUues8c8TqOqVlbbYsAELzA8P632vfcGtpPCn1w approved
$ kubectl  get csr
NAME                                                   AGE    REQUESTOR           CONDITION
node-csr-KChgnUues8c8TqOqVlbbYsAELzA8P632vfcGtpPCn1w   102s   kubelet-bootstrap   Approved,Issued
node-csr-g7dqjomg_sXtldCQA_Dgo4cYsWLH49VL2Z79Cj_rlLU   102s   kubelet-bootstrap   Pending
node-csr-o0Swsg7I14BFAggupTyVq0yL9quB_yc0z4HsdsAZ2Xw   102s   kubelet-bootstrap   Pending

Requesting User：请求 CSR 的用户，kube-apiserver 对它进行认证和授权；
Subject：请求签名的证书信息；
证书的 CN 是 system:node:kube-node2， Organization 是 system:nodes，kube-apiserver 的 Node 授权模式会授予该证书的相关权
限


# 查看当前集群节点
$ kubectl  get nodes
NAME             STATUS   ROLES    AGE     VERSION
172.29.202.145   Ready    <none>   12s     v1.16.0
172.29.202.151   Ready    <none>   9m57s   v1.16.0
172.29.202.155   Ready    <none>   8s      v1.16.0
```

**配置证书自动轮转请求**

`注意:由于集群使用Bootstrap方式对kubelet客户端进行csr证书准许，这样每次在新增node节点后都需要人为去手动请求。k8s内置了自动请求和证
书轮转。`

`注意:需要在master节点配置，由master节点角色进行管控`

```
# 配置证书自动轮转
$ cat auto-approve-node-csr.yml
# A ClusterRole which instructs the CSR approver to approve a user requesting
# node client credentials.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: approve-node-client-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/nodeclient"]
  verbs: ["create"]
---
# A ClusterRole which instructs the CSR approver to approve a node renewing its
# own client credentials.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: approve-node-client-renewal-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeclient"]
  verbs: ["create"]
---
# A ClusterRole which instructs the CSR approver to approve a node requesting a
# serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: approve-node-server-renewal-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]

# 绑定集群相关权限角色
$ kubectl apply -f auto-approve-node-csr.yml
clusterrole.rbac.authorization.k8s.io/approve-node-client-csr created
clusterrole.rbac.authorization.k8s.io/approve-node-client-renewal-csr created
clusterrole.rbac.authorization.k8s.io/approve-node-server-renewal-csr created

$ kubectl create clusterrolebinding node-client-auto-approve-csr --clusterrole=approve-node-client-csr --group=system:kubelet-bootstrap
clusterrolebinding.rbac.authorization.k8s.io/node-client-auto-approve-csr created

$ kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=approve-node-server-renewal-csr --group=system:nodes
clusterrolebinding.rbac.authorization.k8s.io/node-server-auto-renew-crt created

$  kubectl create clusterrolebinding node-client-auto-renew-crt --clusterrole=approve-node-client-renewal-csr --group=system:nodes
clusterrolebinding.rbac.authorization.k8s.io/node-client-auto-renew-crt created

```


**新增节点**

`注意:由于上面已经配置了证书轮转，理论上每次主要有Kubelet进程启动，就会自己办法证书`

一个权限的node加入k8s集群作为集群节点需要安装如下组件:
- [系统初始化](os-init.yml)
- [安装flanneld](flanneld-install.yml)
- [安装docker](docker-install.yml)
- [安装k8s-slave组件](k8s-slave-install.yml)


`注意: 我们在生成k8s-csr.json的时候指定了master-api的slb和域名，将master三个节点的6443负载到slb并且解析到该域名上，后期所有的Kubelet和Kube-proxy直接配置master-api的域名即可，这样可保证真正的高可用`

```
# 初始化机器(不包含磁盘/data的初始化)
$ ansible-playbook -i hosts -e host=node os-init.yml
...
$ ansible-playbook -i hosts -e host=node flanneld-install.yml
...
$ ansible-playbook -i hosts -e host=node docker-install.yml
...

$ ansible-playbook -i hosts -e host=node k8s-slave-install.yml
...

# 依次启动flanneld,docker,kubelet和kube-proxy服务
$ ansible -i hosts node -m shell -a " systemctl daemon-reload && systemctl restart flanneld docker kubelet kube-proxy && systemctl enable flanneld docker kubelet kube-proxy"

# 在k8s-master上查看集群节点信息
# 由于配置了证书自动轮转，可以看到node节点相关初始化完成之后，节点(172.29.202.159)立刻会加入到集群
$ kubectl  get nodes
NAME             STATUS   ROLES    AGE   VERSION
172.29.202.145   Ready    <none>   14h   v1.16.0
172.29.202.151   Ready    <none>   14h   v1.16.0
172.29.202.155   Ready    <none>   14h   v1.16.0
172.29.202.159   Ready    <none>   59s   v1.16.0

# 查看整个集群状况
$ ansible -i hosts master -m shell -a "kubectl get nodes,ep"
172.29.202.145 | CHANGED | rc=0 >>
NAME                  STATUS   ROLES    AGE    VERSION
node/172.29.202.145   Ready    <none>   14h    v1.16.0
node/172.29.202.151   Ready    <none>   14h    v1.16.0
node/172.29.202.155   Ready    <none>   14h    v1.16.0
node/172.29.202.159   Ready    <none>   3m1s   v1.16.0

NAME                   ENDPOINTS                                                     AGE
endpoints/kubernetes   172.29.202.145:6443,172.29.202.151:6443,172.29.202.155:6443   14h

```

**设置集群标签和污点**

`注意:由于我们master节点也安装了kubelet，此时不容易发现那个节点是master，那个节点是node，可以给节点设置一个角色标签`

```
$ kubectl label node 172.29.202.145 node-role.kubernetes.io/master=""
$ kubectl label node 172.29.202.151 node-role.kubernetes.io/master=""
$ kubectl label node 172.29.202.155 node-role.kubernetes.io/master=""

# 查看集群节点状态
$ kubectl  get nodes
NAME             STATUS   ROLES    AGE    VERSION
172.29.202.145   Ready    master   14h    v1.16.0
172.29.202.151   Ready    master   14h    v1.16.0
172.29.202.155   Ready    master   14h    v1.16.0
172.29.202.159   Ready    <none>   7m7s   v1.16.0

# 设置污点，不允许k8s调度pod到master节点
$ kubectl taint nodes 172.29.202.145 node-role.kubernetes.io/master=:NoSchedule
$ kubectl taint nodes 172.29.202.151 node-role.kubernetes.io/master=:NoSchedule
$ kubectl taint nodes 172.29.202.155 node-role.kubernetes.io/master=:NoSchedule

## 此时正常的业务容器不会调度到master节点上

# 给pod配置容忍
$ cat pod.yml
...
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Equal"
  value: ""
  effect: "NoSchedule"
或者
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"

```

`注意:理解Taints,Tolerations和Affinity`

- Taints 和 Node affinity 是对立的概念，用来允许一个 node 拒绝某一类 pods
- Taints 和 tolerations 配合起来可以保证 pods 不会被调度到不合适的 nodes 上干活
- 一个 node 上可以有多个 taints
- 将tolerations 应用到 pods 来允许被调度到合适的 nodes 上干活


#### 相关插件部署

addon:
- [X] [coredns](./manifest/coredns/)
- [X] [traefik](./manifest/traefik/)
- [X] [dashboard](./manifest/dashboard/)
- [X] [prometheus](./manifest/prometheus/)
- [X] [elk](./manifest/elk/)
- [ ] [spinnaker](./manifest/spinnaker)


##### 相关问题处理

**1. master节点上的环境无法`kubectl exec -it`到pod内部**

```
# 
$ kubectl  exec -it myapp-78674b8487-5sszh -n myapp bash
error: unable to upgrade connection: Forbidden (user=system:anonymous, verb=create, resource=nodes, subresource=proxy)

# 给匿名用户绑定cluster-admin集群角色
$ kubectl create clusterrolebinding system:anonymous   --clusterrole=cluster-admin   --user=system:anonymous

$ kubectl  exec -it myapp-78674b8487-5sszh -n myapp bash
[root@myapp-78674b8487-5sszh /]# date
Tue Sep 24 11:51:07 CST 2019

```
**2. master-api高可用性**

`注意:我们在部署k8s集群时，在k8s的证书请求文件中增加了k8s-master-api.bgbiao.cn这个域名，我们可以给k8s-master的api配置高可用的vip，并解析到该地址即可满足api的高可用性(通常在企业内部都会有独立、高可用的slb服务，直接用就可以)`

这里采用一个`nginx`反向代理配置来模拟该高可用的需求(通常没必要为k8s集群单独建立一个keepalive来保证master-api的高可用，直接用企业内部或者云平台的负载设备即可)

```
# nginx核心配置(其实使用L7层负载就可以了，我这里使用的nginx的4层代理)
$ cat nginx.conf
stream {
        upstream tcp_proxy {
                hash $remote_addr consistent;
                server 172.29.202.145:6443;
                server 172.29.202.155:6443;
                server 172.29.202.151:6443;
        }
        server {
                listen 6443;
                proxy_connect_timeout 10s;
                proxy_timeout 10s;
                proxy_pass tcp_proxy;
        }
}

# 测试nginx主机代理的k8s-master-api的接口
$ curl -k https://172.29.202.140:6443 -I
HTTP/1.1 200 OK
Cache-Control: no-cache, private
Content-Type: application/json
Date: Tue, 24 Sep 2019 04:06:07 GMT

# 接下来就可以把域名:k8s-master-api.bgbiao.cn解析到172.29.202.140上了
$ curl -k https://k8s-master-api.bgbiao.cn:6443 -I
HTTP/1.1 200 OK
Cache-Control: no-cache, private
Content-Type: application/json
Date: Tue, 24 Sep 2019 04:08:02 GMT

```
`注意:因为在部署集群时，每个kubeconfig的生成的时候指定的是https://172.29.202.155:6443 作为master-api的，强烈建议改成上述我们模拟的nginx的slb或者直接写成上述域名，这样可以保证整体高可用性`

**3. 集群外部kubeconfig配置生成**

`注意:此时master节点上的权限是可以操作整个集群的，因此为了外部通过接口统一管理集群，需要生成一个和master节点权限一致的配置`

```
# 拷贝master节点的kubeconfig
$ cp  /data/kubernetes/cfg/kubelet.kubeconfig .

# 修改客户端用的kubeconfig(把证书数据搞进去)
# 修改了kubelet.kubeconfig的配置文件
$ kubectl config set-credentials default-auth --client-key=/data/kubernetes/ssl/kubelet-client-current.pem --client-certificate=/data/kubernetes/ssl/kubelet-client-current.pem --kubeconfig=kubelet.kubeconfig --embed-certs=true

# 修改kubelet.kubeconfig中的server地址，改为https://k8s-master-api.bgbiao.cn:6443或者https://172.29.202.140:6443

# 测试kubeconfig
$ kubectl --kubeconfig kubelet.kubeconfig get nodes
NAME             STATUS   ROLES    AGE   VERSION
172.29.202.145   Ready    master   15h   v1.16.0
172.29.202.151   Ready    master   15h   v1.16.0
172.29.202.155   Ready    master   15h   v1.16.0
172.29.202.159   Ready    <none>   62m   v1.16.0

# 虽然可以简单的查看很多资源，但是在操作k8s资源时依然会有一些问题
# 因为我们是以master节点172.29.202.145上的证书和配置生成的Kubeconfig配置文件，所以需要给该节点赋予一些权限
$ kubectl --kubeconfig kubelet.kubeconfig get sa
Error from server (Forbidden): serviceaccounts is forbidden: User "system:node:172.29.202.145" cannot list resource "serviceaccounts" in API group "" in the namespace "default": can only create tokens for individual service accounts

$  kubectl  --kubeconfig kubelet.kubeconfig exec -it myapp-78674b8487-5sszh -n myapp bash
Error from server (Forbidden): pods "myapp-78674b8487-5sszh" is forbidden: User "system:node:172.29.202.145" cannot create resource "pods/exec" in API group "" in the namespace "myapp"

# 给节点用户绑定一个cluster-admin的集群权限
$ kubectl create clusterrolebinding node-master --clusterrole=cluster-admin --user="system:node:172.29.202.145"

# 再次测试
$ kubectl  --kubeconfig kubelet.kubeconfig exec -it myapp-78674b8487-5sszh -n myapp bash
[root@myapp-78674b8487-5sszh /]# date
Tue Sep 24 12:23:30 CST 2019

# 接下来该kubelet.kubeconfig文件就是整个集群管控的入口，只要客户端可以连接master-api就可以通过kubectl操作集群

# 将该kubeconfig备份到ansible项目中
ansible -i hosts 172.29.202.145 -m fetch -a "src=kubelet.kubeconfig dest=files/kubelet.kubeconfig flat=yes"
```


##### 插件部署

```
# 拷贝一个kubectl客户端到ansible主机
$ scp 172.29.202.145:/usr/bin/kubectl /usr/bin/

# 测试
$ kubectl --kubeconfig files/kubelet.kubeconfig get ns
NAME              STATUS   AGE
default           Active   16h
kube-node-lease   Active   16h
kube-public       Active   16h
kube-system       Active   16h
myapp             Active   16h

```

**1. 部署coredns**

```
# 使用ansible-playbooks项目中的coredns
$ kubectl --kubeconfig files/kubelet.kubeconfig apply -f manifest/coredns/coredns.yml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created

# 查看coredns的pod(等待READY为1/1即正常可用)
$ kubectl --kubeconfig files/kubelet.kubeconfig get pods -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-647f8f6476-l7pfc   0/1     Running   0          26s
coredns-647f8f6476-mgb4b   0/1     Running   0          26s

# 创建测试dns的实例
$ kubectl --kubeconfig files/kubelet.kubeconfig create -f manifest/myapp.yml
namespace/myapp created
service/myapp created
deployment.apps/myapp created

$ kubectl --kubeconfig files/kubelet.kubeconfig get pods -n myapp
NAME                     READY   STATUS    RESTARTS   AGE
myapp-77df67ddb4-6lnvc   1/1     Running   0          69s

# 测试pod内部可解析集群内部service名称和外部域名
$ kubectl --kubeconfig files/kubelet.kubeconfig exec -it myapp-77df67ddb4-6lnvc -n myapp bash
[root@myapp-77df67ddb4-6lnvc /]# cat /etc/resolv.conf
nameserver 10.253.0.250
search myapp.svc.cluster.local. svc.cluster.local. cluster.local. openstacklocal
options ndots:5
[root@myapp-77df67ddb4-6lnvc /]# ping myapp
PING myapp.myapp.svc.cluster.local (10.253.49.44) 56(84) bytes of data.
64 bytes from 10.253.49.44 (10.253.49.44): icmp_seq=1 ttl=64 time=0.071 ms
64 bytes from 10.253.49.44 (10.253.49.44): icmp_seq=2 ttl=64 time=0.064 ms
^C
--- myapp.myapp.svc.cluster.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2254ms
rtt min/avg/max/mdev = 0.064/0.067/0.071/0.008 ms
[root@myapp-77df67ddb4-6lnvc /]# ping www.baidu.com
PING www.a.shifen.com (180.101.49.11) 56(84) bytes of data.
64 bytes from 180.101.49.11 (180.101.49.11): icmp_seq=1 ttl=51 time=10.3 ms
^C
--- www.a.shifen.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 10.314/10.314/10.314/0.000 ms

```

**2. 部署traefik**

`注意:`
- 1. k8s-v1.16 版本之后`DaemonSet`使用`apps/v1`接口，并且需要有在`io.k8s.api.apps.v1.DaemonSetSpec`中需要包含`selector`字段
- 2. `traefik2.X`版本增加了`TCP`的支持，所以部署方式上有差异

```
# 创建traefik集群
$ kubectl --kubeconfig files/kubelet.kubeconfig apply  -f manifest/traefik/
configmap/traefik-config unchanged
serviceaccount/traefik-ingress-controller unchanged
daemonset.apps/traefik-ingress-controller created
service/traefik-ingress-service created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller unchanged
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller unchanged
service/traefik-web-ui unchanged
ingress.extensions/traefik-web-ui unchanged

# 查看traefik-ingress-controller
$ kubectl --kubeconfig files/kubelet.kubeconfig get pods -n kube-system  | grep traefik
traefik-ingress-controller-2n5nh   1/1     Running   0          3m8s
traefik-ingress-controller-2zvr4   1/1     Running   0          3m8s
traefik-ingress-controller-6g7tp   1/1     Running   0          3m8s
traefik-ingress-controller-6kwvj   1/1     Running   0          3m8s

# 测试traefik(带上servername访问每个node节点测试)
$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.29.202.159
<a href="/dashboard/">Found</a>.


```

**3. 部署官方dashborad**

`注意:基于最新的官方dashboard改造，去除了tls认证和授权`

```
$ kubectl --kubeconfig files/kubelet.kubeconfig apply  -f manifest/dashboard/kubernetes-dashboard-httpv2.0.0-beat4.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
ingress.extensions/kubernetes-dashboard created

# 查看dashboard相关pod
## kubernetes-dashboard为dashboard
## dashboard-metrics-scraper为dashboard中的基础服务监控，需要metrics-server和promethe的支持
$ kubectl --kubeconfig files/kubelet.kubeconfig get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
dashboard-metrics-scraper   ClusterIP   10.253.36.187    <none>        8000/TCP                       2m53s
kubernetes-dashboard        NodePort    10.253.227.233   <none>        443:37783/TCP,9090:46764/TCP   2m54s

# 测试(天然支持pod webshell)
$ curl -H 'host:prod-dashboard.bgbiao.cn' 172.29.202.159  -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 1262
Content-Type: text/html; charset=utf-8
Date: Tue, 24 Sep 2019 06:03:37 GMT
Last-Modified: Thu, 29 Aug 2019 09:14:59 GMT
Vary: Accept-Encoding

```

**4. 部署metrics**

[metrics部署](./manifest/metrics-server/)


**5. 部署prometheus**

[prometheus部署](./manifest/prometheus/readme.md)

**6. 部署flk**

[flk部署](./manifest/elk/)
