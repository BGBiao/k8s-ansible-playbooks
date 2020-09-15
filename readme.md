## 基于ansible使用二进制快速部署k8s-v1.19

[详细部署说明](./ansible-install-k8s-all.md)

### tls/ssl证书构建

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

#### 0.创建根证书(CA)

注意:k8s集群和etcd集群公用一个ca进行证书颁发,后续创建的所有证书都由根证书签名

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


# 生成key和签名证书
# 使用cfssl gencert 命令来生成新key和签名证书
## 该指令可用于从CSR配置(CSRJSON 上述的ca-csr.json)中生成key和证书；使用CA的key和CSR来重签CA证书；或者使用CA的key和证书来重签CA证书
$ pushd 
$ cfssl gencert -initca ca-csr.json
......
......
2020/09/13 07:14:58 [INFO] signed certificate with serial number 8586413332462455829602406972761074712627171945
{"cert":"-----BEGIN CERTIFICATE-----\nMIIDwDCCAqigAwIBAgIUAYEHMOpThhfzOhAEcUk8BXkc0mkwDQYJKoZIhvcNAQEL\nBQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl\naUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr\ndWJlcm5ldGVzMCAXDTIwMDkxMzA3MTAwMFoYDzIxMjAwODIwMDcxMDAwWjBlMQsw\nCQYDVQQGEwJDTjEQMA4GA1UECBMHQmVpSmluZzEQMA4GA1UEBxMHQmVpSmluZzEM\nMAoGA1UEChMDazhzMQ8wDQYDVQQLEwZTeXN0ZW0xEzARBgNVBAMTCmt1YmVybmV0\nZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQChH10PemY3yg5XL1zg\nYMhtNHJ1Crz9iUcYpK7aK8609v6XyzvqzjMlIbL9Fz8yC8+n6Tul3BsgirScIoei\njxStV2c89CLVVDTf+9CxfyWVrogH1zoUJ15qyxUXb4vS95q2wRDcwkckW9T8UzLB\npoJnjEyf/FAR9u4UOxaNaZzpCkNdY6JSOOU5fMI/XBdUxHx9aaefHxvVEQpI7hAH\nZwXqEAQIvWLrZsq6kqFQyppja/pkn9Y9IDRD4fb/csdie7dpITkuxS78YvoF0UzA\nnzxVzGxb6s5D+LLQAiWfoKhztd+L4GwcwLzie0m6ddAz7sR321YCfWbcju/BRcYm\nta99AgMBAAGjZjBkMA4GA1UdDwEB/wQEAwIBBjASBgNVHRMBAf8ECDAGAQH/AgEC\nMB0GA1UdDgQWBBR11h6uvnICj2nE2GWHPnygS6d+mzAfBgNVHSMEGDAWgBR11h6u\nvnICj2nE2GWHPnygS6d+mzANBgkqhkiG9w0BAQsFAAOCAQEAe2FMMKtiYmTs3SGi\nKIPbGmo246gGGV9BGZ4pgkWtWYa0TeV2S4fINHZsunxhuUuXEzRXQe/M4KfOE3Fk\nAC3yrdO7mReaX8Jw97VLsU35IXNYvKruQb5NjDn71j08UEUMGRqr7mXsl1E3Z7uM\nt2CQXjHAG6qFK/i76nQOOAWmHPQAMzJtTVbXXCJGpeW9PBMVR7awA356gn7ciuN8\njgH/vQ5223zMcI3V5KTQ9ssqU0UIrQ65+p9a+KTkhtVxmOv2lYiuTBra9aP/S0Vf\nAHGC+e9RNPxq+fs6uPspk6yy/LNK5Eb1H9xibxZ3abD8X7Ujl6Duo3+goUFonv6E\nZ9u4Uw==\n-----END CERTIFICATE-----\n","csr":"-----BEGIN CERTIFICATE REQUEST-----\nMIICqjCCAZICAQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAO\nBgNVBAcTB0JlaUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMw\nEQYDVQQDEwprdWJlcm5ldGVzMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC\nAQEAoR9dD3pmN8oOVy9c4GDIbTRydQq8/YlHGKSu2ivOtPb+l8s76s4zJSGy/Rc/\nMgvPp+k7pdwbIIq0nCKHoo8UrVdnPPQi1VQ03/vQsX8lla6IB9c6FCdeassVF2+L\n0veatsEQ3MJHJFvU/FMywaaCZ4xMn/xQEfbuFDsWjWmc6QpDXWOiUjjlOXzCP1wX\nVMR8fWmnnx8b1REKSO4QB2cF6hAECL1i62bKupKhUMqaY2v6ZJ/WPSA0Q+H2/3LH\nYnu3aSE5LsUu/GL6BdFMwJ88VcxsW+rOQ/iy0AIln6Coc7Xfi+BsHMC84ntJunXQ\nM+7Ed9tWAn1m3I7vwUXGJrWvfQIDAQABoAAwDQYJKoZIhvcNAQELBQADggEBAAFV\ncqE7Er1A5gCTn6K6Ol/cbu02EiAjkmV1jXuNw/40S9tdUBkithwC27qmrYU+Za26\nxjHq6YUMDxP13OHUZPUBaT8U1dw+/u93Jx7ZitYfIhG6QyU+c5WpLvjmL1UT3v9G\nfXsAD9FFtSbaGgbUdCn8o+GJuQ+zNuoLetF2/YLqVI2RwbIY9+3VWFS6CvGethVo\nUFGbR3lF5Jktx1r+NrqZT1ERyIkr2IK8uVLDCMpGboqBX32FfeAFHABlZUpe4Hjz\nT0Te2sFAkA6wWjUhaZMW815yMygkTI4snwt1uxfBN4I1x8AgAndY8a2mRuEd83zy\nAfTv9ANX7msvihRZVQU=\n-----END CERTIFICATE REQUEST-----\n","key":"-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEAoR9dD3pmN8oOVy9c4GDIbTRydQq8/YlHGKSu2ivOtPb+l8s7\n6s4zJSGy/Rc/MgvPp+k7pdwbIIq0nCKHoo8UrVdnPPQi1VQ03/vQsX8lla6IB9c6\nFCdeassVF2+L0veatsEQ3MJHJFvU/FMywaaCZ4xMn/xQEfbuFDsWjWmc6QpDXWOi\nUjjlOXzCP1wXVMR8fWmnnx8b1REKSO4QB2cF6hAECL1i62bKupKhUMqaY2v6ZJ/W\nPSA0Q+H2/3LHYnu3aSE5LsUu/GL6BdFMwJ88VcxsW+rOQ/iy0AIln6Coc7Xfi+Bs\nHMC84ntJunXQM+7Ed9tWAn1m3I7vwUXGJrWvfQIDAQABAoIBAB2J8Xa3+ut5eL2V\nKlLci4Ix3lYE3PciZs1my8OlymS076IGmXqHySqijf0GeQiEz9I52TykKLkDlO8X\nCYTM9H5/CqdLHuO7Z2I0+WLBK7PQZpIBbF1rhkzP5JMCWUEZMd0VcjD20TIiP97u\npdyI2VmAiD/AczGH8sf0uUK9vQ2gEIw/265i7js04f/r9Dy3sp1vHMzNUcDaYVdt\n+qEhoF8xOZM07YJQjcQsDbRDVjU+Sppf//hwPiiApR6pENSFqCDk2k5WKGoTDoem\nnjP3kFfz48miN35N6Yh5DFrfSkIa+V/cPileTMV1cuSob48AtLcImZ69CjncSO2C\nK/WJBmECgYEA0kHIk79WIyb1r2DovvpNIT4Uf2HgxrbvWS33fvKyjFH7s4ZAa+ff\nG5YoVpF9mnUNA1G0nSxIilLwhZN/F8m/87Ya7fz2ik1GaT+aDApqNKxoF+k7gPQP\nuuVRck0K5hXsGXB1mEs7uFwqflHFc1/6w9KuA6yZ/7EfFH1Pf8aUAskCgYEAxC0M\nFye4qgkWGeeBxDGPCtq6bEX6CF4HDeZp+94iEOAgOtpYpJ2Y8Nuipp1xjU9ephgg\nSmD47qOwccGO6oS3UZyL4eoD2feWQasyHviEwMNnBSIfwYznCHFLKAraV708vA3K\nBnkF5iAJ68WD5BLS1YE0YatK9E6wSt/W9yRjTRUCgYEAzd/h8WGZi0P7r2UZoN5f\npZwu3+fL+2dmh5Dt1Vz5HVKtPcTH0aCyIkXua418SkAwpL5dNsUEpoS9xF1/RaCj\nlpQKXFukYBl4R1gik4WjJr5mEnuqawMPX/ZowJ3VfSOcEfC/BIcuC8AbT6LrzqP9\nW78v6qMYC3i4MQzeSgP8K5kCgYEAwujm7FKg3P/uH4qumZmLv4MWWeEkzQ9u/taB\nUqefPRkRrKeoDtYuUJBICDbBzV6gcXHjE0NJ0QB9nGhtcICwCrv5F1qEvRmLBm/r\nem38p/D8+FKxLoKqQO8fdwdhbG8uWsFwigHQZJZMhR5XLlGtfEfFHY0tCZLtAVdo\no2BZ8QkCgYEAqfZ0VEMJPS3riHSjahG/ZTqAjBkLVXosyKpkQaZ/zTVVOQYjmHOb\nC5aV4GvbeIEtjmPmZOc6KTiKatfN/IRJybzl1QAi97xUqiPZ5pJV5vhF1fjVk7e5\n2yVjZe9PSAIHQeipGXhv6MLuP+hgNQHUIT+PotuTpeZzC/R1tDzIHNo=\n-----END RSA PRIVATE KEY-----\n"}

# 如上指令会根据当前CA-CSR的配置来生成CSR请求，CA的key以及CA的证书，但是会放在一个大json串中，因此通常我们需要使用cfssljson工具进行转储
# cfssljson 是一个专门从CFSSL响应体中解析并保存到json文件的一个工具
$ cfssl gencert -initca ca-csr.json  | cfssljson -bare ca
2020/09/13 07:20:26 [INFO] generating a new CA key and certificate from CSR
2020/09/13 07:20:26 [INFO] generate received request
2020/09/13 07:20:26 [INFO] received CSR
2020/09/13 07:20:26 [INFO] generating key: rsa-2048
2020/09/13 07:20:27 [INFO] encoded CSR
2020/09/13 07:20:27 [INFO] signed certificate with serial number 306065836815286281292411979507270171176198785172


# 此时会生成三个文件ca.csr,ca-key.pem,ca.pem 即所谓的ca请求数据，ca的KEY，以及ca证书。
$ ll -t | head -4
total 20
-rw-r--r--  1 root root 1001 Sep 13 07:20 ca.csr
-rw-------  1 root root 1679 Sep 13 07:20 ca-key.pem
-rw-r--r--  1 root root 1363 Sep 13 07:20 ca.pem

# 需要注意的是，通常情况下我们的证书可能会过期，在过期前，我们需要使用ca-key.pem 和ca.pem或CSRJSON(ca-csr.json) 来对ca证书进行重签，以使之继续可用。


```

#### 1.创建etcd证书

```
# 创建etcd证书的请求文件
$ cat etcd-ssl/etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.0.230",
    "192.168.0.145",
    "192.168.0.23"
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


# 由于我们前面已经初始化好了CA相关的key和证书，这里etcd集群直接使用之前的CA相关证书和配置即可
$ pushd etcd-ssl/

# 这里我们在生成etcd的证书时会指定一个配置文件，也就是前面的ca-config.json，该文件中统一声明了证书一些公共配置，比如有效期，profiles等基本配置，因为我们的etcd证书是为kubernetes和etcd来使用，因此也需要指定profile(在profile中我们定义了kubernetes这个配置，其中也声明了它的用途和期限)

$ cfssl gencert -ca="../ca-ssl/ca.pem" -ca-key="../ca-ssl/ca-key.pem"   -config="../ca-ssl/ca-config.json" -profile=kubernetes etcd-csr.json
.....
.....


# 如上依然会生成证书相关的三个文件，即key,cert,csr 此时，我们依然需要使用cfssljosn工具将其保存成json文件

$ cfssl gencert -ca="../ca-ssl/ca.pem" -ca-key="../ca-ssl/ca-key.pem"   -config="../ca-ssl/ca-config.json" -profile=kubernetes etcd-csr.json  | cfssljson -bare etcd

$ ll -t * | head -4
-rw-r--r-- 1 root root 1062 Sep 13 07:44 etcd.csr
-rw------- 1 root root 1675 Sep 13 07:44 etcd-key.pem
-rw-r--r-- 1 root root 1436 Sep 13 07:44 etcd.pem
 
```


#### 2.创建k8s-master相关证书

`注意:` 通常对外暴露的master-api是一个无状态的服务，我们会使用一些负载均衡技术来对外统一暴露master-api接口，因此建议将master-api使用负载均衡技术，并且让客户端(二次开发或者node节点)使用域名进行接入。

```

# 192.168.0.110 为我们的master-api的负载均衡地址(也可是keepalive+nginx的虚拟地址，保证高可用即可)
# 10.253.0.1 为集群中默认的kubernetes的svc地址，一般CRD通常都会调用内部的kubernetes的默认svc地址，来和集群进行通信
$ cat k8s-ssl/k8s-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.0.230",
      "192.168.0.145",
      "192.168.0.145",
      "192.168.0.110",
      "k8s-master-api.goops.top",
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


# 生成master节点的证书和key

$ pushd k8s-ssl
$ cfssl gencert -ca=../ca-ssl/ca.pem -ca-key=../ca-ssl/ca-key.pem -config=../ca-ssl/ca-config.json -profile=kubernetes k8s-csr.json | cfssljson -bare k8s
.....
.....

$ ll -t k8s* | head -4
-rw-r--r--. 1 root root 1301 Sep 13 08:02 k8s.csr
-rw-r--r--. 1 root root 1679 Sep 13 08:02 k8s-key.pem
-rw-r--r--. 1 root root 1675 Sep 13 08:02 k8s.pem
-rw-r--r--  1 root root  616 Sep 13 07:56 k8s-csr.json

```


#### 3.创建kube-proxy证书

`注意:` 由于kube-proxy其实也是整个集群的一个角色，因此直接使用k8s-master的证书也是没有问题的，但由于kube-proxy的角色相对比较重要，通常我们会给指定的User(`system:kube-proxy`)单独创建一个证书。


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

$ ll -t kube-proxy* | head -4
-rw-r--r--. 1 root root 1009 Sep 13 08:08 kube-proxy.csr
-rw-r--r--. 1 root root 1675 Sep 13 08:08 kube-proxy-key.pem
-rw-r--r--. 1 root root 1403 Sep 13 08:08 kube-proxy.pem
-rw-r--r--  1 root root  230 Sep 13 08:08 kube-proxy-csr.json


```

至此，我们已经按照我们的规划，将CA证书和key，etcd的证书和key，k8s的证书和key以及kube-proxy的证书和key基本准备完毕，接下来就是来初始化各个集群角色了。

### 安装Etcd集群

`注意:` 我们使用etcd-v3.4.13版本来搭建集群，值得关注的是，在etcdv3.4版本之后默认使用了V3的API接口，后面的整个集群部署需要注意这里(flannel暂不支持V3接口)


```
# 注意: 该hosts文件在ansible-playbooks 项目下

$ cat hosts
[master]
192.168.0.230 nodename=etcd01
192.168.0.145 nodename=etcd02
192.168.0.23 nodename=etcd03

# 执行ansible-playbook 的etcd-install 剧本

$ ansible-playbook  -i hosts -e host=all etcd-install.yml

PLAY [all] ************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************
ok: [192.168.0.23]
ok: [192.168.0.145]
ok: [192.168.0.230]

.....
.....
.....


# 更新完成后确认下配置

$ ansible -i hosts all -m shell -a "cat /data/etcd/cfg/etcd.conf | egrep 'ETCD_NAME|PEER_URLS'"
192.168.0.230 | CHANGED | rc=0 >>
ETCD_NAME="etcd01"
ETCD_LISTEN_PEER_URLS="https://192.168.0.230:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.230:2380"
192.168.0.23 | CHANGED | rc=0 >>
ETCD_NAME="etcd03"
ETCD_LISTEN_PEER_URLS="https://192.168.0.23:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.23:2380"
192.168.0.145 | CHANGED | rc=0 >>
ETCD_NAME="etcd02"
ETCD_LISTEN_PEER_URLS="https://192.168.0.145:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.0.145:2380"

# 批量启动集群(这里只是为了演示整个ansible剧本的标准化和准确性，其实启动也可以直接添加至etcd-install剧本中)

$ ansible -i hosts all -m shell -a 'systemctl daemon-reload && systemctl restart etcd && systemctl enable etcd'
....
....




# 测试etcd集群是否正常

$ ansible -i hosts all -m shell -a 'etcdctl --cacert="/data/etcd/ssl/ca.pem" --key="/data/etcd/ssl/etcd-key.pem" --cert="/data/etcd/ssl/etcd.pem" --endpoints=192.168.0.230:2379,192.168.0.145:2379,192.168.0.23:2379 endpoint status'
192.168.0.145 | CHANGED | rc=0 >>
192.168.0.230:2379, c2932a8aaaebe259, 3.4.13, 20 kB, false, false, 40, 9, 9,
192.168.0.145:2379, 4b1c1019f5515c8c, 3.4.13, 20 kB, true, false, 40, 9, 9,
192.168.0.23:2379, a2721e6aeb23ec8e, 3.4.13, 20 kB, false, false, 40, 9, 9,
192.168.0.23 | CHANGED | rc=0 >>
192.168.0.230:2379, c2932a8aaaebe259, 3.4.13, 20 kB, false, false, 40, 9, 9,
192.168.0.145:2379, 4b1c1019f5515c8c, 3.4.13, 20 kB, true, false, 40, 9, 9,
192.168.0.23:2379, a2721e6aeb23ec8e, 3.4.13, 20 kB, false, false, 40, 9, 9,
192.168.0.230 | CHANGED | rc=0 >>
192.168.0.230:2379, c2932a8aaaebe259, 3.4.13, 20 kB, false, false, 40, 9, 9,
192.168.0.145:2379, 4b1c1019f5515c8c, 3.4.13, 20 kB, true, false, 40, 9, 9,
192.168.0.23:2379, a2721e6aeb23ec8e, 3.4.13, 20 kB, false, false, 40, 9, 9,


```


### Kube-Master部署

- kube-apiserver
- kube-scheduler
- kube-controller-manager

`注意:` k8s-master的证书我们前面已经生成了,在k8s-ssl目录(master节点统一使用一套证书)

`注意:` 在`k8s-master-install.yml`中的`app_version`需要改成官方默认下载的二进制包的目录；`etcd_endpoints`需要改成上面的etcd的地址。

```
# 生成Bootstrapping配置文件(用来对kubelet第一次授权)
$ echo `head -c 16 /dev/urandom | od -An -t x | tr -d ' '`,kubelet-bootstrap,10001,\"system:kubelet-bootstrap\"  > templates/token.csv
$ cat templates/token.csv

# 使用ansible-playbooks统一部署k8s-master基础环境
$ ansible-playbook -i hosts -e host=master k8s-master-install.yml
.....
.....

# 批量启动k8s-master节点全部组件
$ ansible -i hosts master -m shell -a "systemctl daemon-reload && systemctl restart kube-apiserver kube-controller-manager kube-scheduler && systemctl enable kube-apiserver kube-controller-manager kube-scheduler"


# 检查master节点角色是否正常
# 这里可以看到在v1.19版本中，v1版本的ComponentStatus 已经被弃用
$ ansible -i hosts all -m shell -a "kubectl get cs,nodes"
192.168.0.230 | CHANGED | rc=0 >>
NAME                                 STATUS    MESSAGE             ERROR
componentstatus/controller-manager   Healthy   ok
componentstatus/scheduler            Healthy   ok
componentstatus/etcd-0               Healthy   {"health":"true"}
componentstatus/etcd-2               Healthy   {"health":"true"}
componentstatus/etcd-1               Healthy   {"health":"true"}   Warning: v1 ComponentStatus is deprecated in v1.19+
192.168.0.145 | CHANGED | rc=0 >>
NAME                                 STATUS    MESSAGE             ERROR
componentstatus/controller-manager   Healthy   ok
componentstatus/scheduler            Healthy   ok
componentstatus/etcd-1               Healthy   {"health":"true"}
componentstatus/etcd-0               Healthy   {"health":"true"}
componentstatus/etcd-2               Healthy   {"health":"true"}   Warning: v1 ComponentStatus is deprecated in v1.19+
192.168.0.23 | CHANGED | rc=0 >>
NAME                                 STATUS    MESSAGE             ERROR
componentstatus/scheduler            Healthy   ok
componentstatus/controller-manager   Healthy   ok
componentstatus/etcd-0               Healthy   {"health":"true"}
componentstatus/etcd-2               Healthy   {"health":"true"}
componentstatus/etcd-1               Healthy   {"health":"true"}   Warning: v1 ComponentStatus is deprecated in v1.19+


```

在`kube-master`节点的所有组件(kube-apiserver,kube-controller-manager,kube-scheduler)部署成功后，我们可以使用`kubectl get cs,nodes`来查看组件的状态是否健康，但是我们也关注到了一个细节，就是有一条`Warning`的日志提示`v1版本的 ComponentStatus`接口将在以后的版本被废弃掉。

通过查阅官方[v1.19-Release-Notes-changes-by-kind](https://kubernetes.io/docs/setup/release/notes/#changes-by-kind)中提到下面一句话: 

> Kube-apiserver: the componentstatus API is deprecated. This API provided status of etcd, kube-scheduler, and kube-controller-manager components, but only worked when those components were local to the API server, and when kube-scheduler and kube-controller-manager exposed unsecured health endpoints. Instead of this API, etcd health is included in the kube-apiserver health check and kube-scheduler/kube-controller-manager health checks can be made directly against those components' health endpoints. (#93570, @liggitt) [SIG API Machinery, Apps and Cluster Lifecycle]

大概意思就是说: `kube-apiserver` 在未来废弃了`componentstatus`这个API。该API提供了etcd,kube-shceduler,kube-controller-manager 组件的状态，但是它仅当这些组件和API Server在一起部署时，并且当kube-scheduler和kube-controller-manager 暴露了非安全的健康端点时，才正常工作。为了代替该API，etcd的健康检查呗包含进了kube-apiserver 的健康检查逻辑中，并且kube-sheduler/kube-controller-manager 可以直接针对自己的健康检查端点进行检查。因此，componentstatus 这个接口就没有必要了。

看到这里，会不会觉得，顶级项目的设计和演进就是这么干净和利落，回头在想想自己公司具体的代码吧😭


### Kube-node部署

注意: 在K8S的整个组件和概念中，其实node节点仅有两个角色，即`kube-proxy`和`kubelet`，前者主要负责将所有的pod的网络通过一些列的规则在整个集群打通，而后者主要负责整个pod的生命周期管理。 而至于真正Pod中的容器生命周围管理则通过Plugin的方式由一些Runtime进行接入，通常我们常用且生产成熟的就是大名鼎鼎的Docker了；但像Docker这种Runtime 其本身的网络又无法支撑整个集群以及业务的需求，因此网络上的支撑，也由kubelet来通过Plugin的方式来进行CNI接入，在通用开源的解决方案里，比较出名的就是 Flannel 和 Calico 两个组件了。

因此，一个可真正运行的node节点包含以下组件和插件: 

- kubelet: Pod的生命周期管理
- kube-proxy: 集群网络代理
- Runtime: 容器运行时(Docker)
- CNI: 容器网络接口(Flannel/Calico) 【由于flannel较为简单，之前一直使用flannel，但是flannel默认不支持ETCD V3接口，这次将使用Calico来构建容器网络】


#### 0.生成bootstrap配置文件和kube-proxy配置文件

````
$ export BOOTSTRAP_TOKEN=`cat /data/kubernetes/cfg/token.csv  | awk -F ',' '{print $1}'`
$ echo ${BOOTSTRAP_TOKEN}

# 为了验证集群，我们可以先使用master的任意节点来部署node节点(后期可替换成k8s证书中hosts部分的地址)
# 当然如果有完备的高可用API网关或者SLB之类的，可以直接将kube apiserver的接口对外地址配置
$ export KUBE_APISERVER="https://192.168.0.230:6443"

$ ls /data/kubernetes/ssl/ca.pem
# 配置bootstrap配置文件
$ kubectl config set-cluster kubernetes --certificate-authority=/data/kubernetes/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=bootstrap.kubeconfig
$ kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=bootstrap.kubeconfig
$ kubectl config set-context default --cluster=kubernetes  --user=kubelet-bootstrap --kubeconfig=bootstrap.kubeconfig

# 在当前环境中使用该上下文
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 配置kube-proxy的配置文件
$ kubectl config set-cluster kubernetes --certificate-authority=/data/kubernetes/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-credentials kube-proxy --client-certificate=/data/kubernetes/ssl/kube-proxy.pem --client-key=/data/kubernetes/ssl/kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-context default --cluster=kubernetes --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

# 在当前环境中使用该上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 在集群中创建一个用于bootstrap 的clusterrolebinding 将system:node-bootstrapper 角色绑定至用户kubelet-bootstrap 
$ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

然后将生成的`bootstrap.kubeconfig` 和 `kube-proxy.kubeconfig` 拷贝到项目的 `templates/` 下。


#### 1.批量部署Node节点角色

`注意:` 由于我们当前将master节点也充当node节点的角色，因此下载和部署kubernetes源码包的部分就可以略过了，唯一需要关心的就是`kubelet`和`kube-proxy`的配置问题了。

```
$ ansible-playbook -i hosts -e host=master k8s-slave-install.yml --tags=config
....
....

PLAY RECAP ************************************************************************************************************************************
192.168.0.145              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.0.23               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.0.230              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```


需要注意的是，我们在kubelet 的启动逻辑里设置了kubelet 必须在docker.service 启动之后进行启动，为了保证服务的尽可能短的影响周期。

此时，如果节点上的`docker`服务没有启动，直接启动`kubelet`会提示`kubelet.service`不存在。


#### 3.部署Runtime 运行时Docker组件

```
# 这里需要先部署docker的运行时
$ ansible-playbook -i hosts -e host=all docker-install.yml
....
....
PLAY RECAP ************************************************************************************************************************************
192.168.0.145              : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.0.23               : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.0.230              : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


# 启动docker，kubelet，kube-proxy 服务
$ ansible -i hosts all -m shell -a "systemctl daemon-reload && systemctl restart docker"
.....
.....


```


#### 4.配置Node节点，并生成kubelet证书

```
$ ansible -i hosts all -m shell -a "systemctl daemon-reload && systemctl restart kubelet kube-proxy"
....
....


# kubelet启动后，会自动向kube apiserver 发起认证请求。
$ kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-01keemXrwk85QRvmxmKD7BL8Pe8-UpCYmEOdHovJMTE   40s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-mhdJODMPp8mQwJrnW-Gl2baQlWWgcrOvrYt3jmbw-dM   40s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-rgY6WzL2gwGFa5XyWLgJ4_ypJUPtbruuLloygUQ80P8   2m3s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 手动接受node的请求
$ kubectl certificate approve node-csr-rgY6WzL2gwGFa5XyWLgJ4_ypJUPtbruuLloygUQ80P8
.....
.....

# 查看csr请求状态（发现已经都被Approved）
$ kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-01keemXrwk85QRvmxmKD7BL8Pe8-UpCYmEOdHovJMTE   3m11s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-mhdJODMPp8mQwJrnW-Gl2baQlWWgcrOvrYt3jmbw-dM   3m11s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-rgY6WzL2gwGFa5XyWLgJ4_ypJUPtbruuLloygUQ80P8   4m34s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

# 当node节点成功被允许后，我们之前设置的一些配置就会自动向kubelet 进行颁发证书。
# 当没有接受node的csr请求时，客户端的证书属于颁发中，当接受后，该证书正式生成，从此kubelet就可以愉快的玩耍了

$ ls -lt /data/kubernetes/ssl/* | head -5
lrwxrwxrwx 1 root root   59 Sep 13 16:10 /data/kubernetes/ssl/kubelet-client-current.pem -> /data/kubernetes/ssl/kubelet-client-2020-09-13-16-10-05.pem
-rw------- 1 root root 1228 Sep 13 16:10 /data/kubernetes/ssl/kubelet-client-2020-09-13-16-10-05.pem
-rw-r--r-- 1 root root 2274 Sep 13 16:05 /data/kubernetes/ssl/kubelet.crt
-rw------- 1 root root 1675 Sep 13 16:05 /data/kubernetes/ssl/kubelet.key

# 此时我们可以查看集群的节点状态信息

$ kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
192.168.0.145   Ready    <none>   4m33s   v1.19.0
192.168.0.23    Ready    <none>   4m17s   v1.19.0
192.168.0.230   Ready    <none>   4m11s   v1.19.0

```

至此，我们已经可以看到，我们的node节点现在看起来也已经就绪了。不过真的完全就绪了吗？还记得前面说到Node节点真正可运行，除了Runtime的Docker之外，还是需要CNI 来将集群的Pod打通的。

#### 4.部署CNI插件

`注意:` 由于新版本的etcd默认是V3接口的，并且Flannel默认不支持V2接口，因此我们使用Clico进行安装。

`注意:` 由于我们是使用额外的CNI插件，因此在Kubelet的配置中需要明确指定如下配置: 

```
--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin

# 在指定了上述配置后，重启kubele组件会重启成功，但是会发现kubectl get nodes 时节点处于未就绪状态，此时需要安装CNI插件后即可
```

[Clico](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)

[从头开始安装Calico网络](https://docs.projectcalico.org/getting-started/kubernetes/hardway/)


```
$ mkdir -p /etc/cni/net.d /opt/cni/bin

$ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml


$ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
 
$ watch kubectl get pods -n calico-system

$ kubectl get pods -n tigera-operator
NAME                              READY   STATUS    RESTARTS   AGE
tigera-operator-b96747c7d-mmv9z   1/1     Running   0          6m41s

$ kubectl get pods -n calico-system
NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-69fbbf7967-sv9jk   1/1     Running             0          5m35s
calico-node-cgrdz                          0/1     Init:0/2            0          5m35s
calico-node-cr7v8                          1/1     Running             0          5m35s
calico-node-vnw7t                          0/1     Init:0/2            0          5m35s
calico-typha-559c75f66d-j59mm              1/1     Running             0          3m37s
calico-typha-559c75f66d-nlgqg              0/1     ContainerCreating   0          5m36s
calico-typha-559c75f66d-rckgk              0/1     ContainerCreating   0          3m37s


# 可以看到，当一个calico-node节点安装成功后，对应的node节点状态就会变为就绪状态
$ kubectl get nodes
NAME            STATUS     ROLES    AGE   VERSION
192.168.0.145   NotReady   <none>   21h   v1.19.0
192.168.0.23    NotReady   <none>   21h   v1.19.0
192.168.0.230   Ready      <none>   21h   v1.19.0


$ kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-69fbbf7967-sv9jk   1/1     Running   0          13h
calico-node-cgrdz                          1/1     Running   0          13h
calico-node-cr7v8                          1/1     Running   0          13h
calico-node-vnw7t                          1/1     Running   0          13h
calico-typha-559c75f66d-j59mm              1/1     Running   0          13h
calico-typha-559c75f66d-nlgqg              1/1     Running   0          13h
calico-typha-559c75f66d-rckgk              1/1     Running   0          13h

$ kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
192.168.0.145   Ready    <none>   35h   v1.19.0
192.168.0.23    Ready    <none>   35h   v1.19.0
192.168.0.230   Ready    <none>   35h   v1.19.0

# 当节点都就绪后，每个节点会有具体的CNI插件，CNI的配置信息以及CNI的pod的kubeconfig

$ ls /opt/cni/bin/
bandwidth  calico  calico-ipam  flannel  host-local  install  loopback  portmap  tuning

$ ls /etc/cni/net.d/
10-calico.conflist  calico-kubeconfig

# 其实这里能够看出来，我们的calico的CNI插件，默认是使用的kubernetes的datastore方式
$ cat /etc/cni/net.d/10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "datastore_type": "kubernetes",
      "mtu": 1410,
      "nodename_file_optional": false,
      "log_file_path": "/var/log/calico/cni/cni.log",
      "ipam": {
          "type": "calico-ipam",
          "assign_ipv4" : "true",
          "assign_ipv6" : "false"
      },
      "container_settings": {
          "allow_ip_forwarding": false
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {"type": "portmap", "snat": true, "capabilities": {"portMappings": true}}
  ]

```

#### 5.配置Calico网络

[calicoctl](https://docs.projectcalico.org/getting-started/clis/calicoctl/install)

```
# 下载calicoctl 
$ curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.16.1/calicoctl

$ chmod a+x calicoctl && mv calicoctl /usr/sbin/

```

[配置calicoctl连接到数据存储层](https://docs.projectcalico.org/getting-started/clis/calicoctl/configure/)

Calicoctl 可以配置从etcd集群或者Kubernetes集群的API中去进行读写相关的数据。

- [etcd datastore](https://docs.projectcalico.org/getting-started/clis/calicoctl/configure/etcd)
- [kubernetes API datastore](https://docs.projectcalico.org/getting-started/clis/calicoctl/configure/kdd)

注意: `calicoctl`程序默认从配置文件`/etc/calico/calicoctl.cfg`中进行读取相关的配置文件，也可以使用`--config`参数指定配置文件。同时，如果程序无法从配置文件中读取，将会从默认的环境变量中进行相关配置的使用。

```
# calicoct.cfg 配置文件实例
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "etcdv3"
  etcdEndpoints: "http://etcd1:2379,http://etcd2:2379"
  ...

```

**配置ETCD的数据存储**

```
# 将etcd相关的证书准备到固定的位置

$ cat  /etc/calico/calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://192.168.0.230:2379,https://192.168.0.23:2379,https://192.168.0.145:2379
  etcdKeyFile: /data/etcd/ssl/etcd-key.pem
  etcdCertFile: /data/etcd/ssl/etcd.pem
  etcdCACertFile: /data/etcd/ssl/ca.pem


# 获取calicoctl 的calico节点
$ calicoctl get nodes
NAME

```

**配置Kubernetes API的数据存储**

```
$  export KUBECONFIG=/data/kubernetes/cfg/kubelet.kubeconfig
$  export DATASTORE_TYPE=kubernetes

$ calicoctl get nodes -o wide
NAME            ASN         IPV4               IPV6
192.168.0.145   (unknown)   192.168.0.145/24
192.168.0.23    (unknown)   192.168.0.23/24
192.168.0.230   (unknown)   192.168.0.230/24

# 也可以配置到配置文件中
$ cat /etc/calico/calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/data/kubernetes/cfg/kubelet.kubeconfig"


```

也可以将`calicoctl` 直接以pod的方式部署: 

```
# etcd为后端存储的pod
$ kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl-etcd.yaml

# kubernetes api 为后端存储的Pod
$  kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml


# 使用calicoctl 查看集群相关信息
$ kubectl exec -ti -n kube-system calicoctl -- /calicoctl get profiles -o wide

$ alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl"

# 然后就可以在节点上开心的执行calicoctl 命令了

$ calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.0.145 | node-to-node mesh | up    | 03:06:11 | Established |
| 192.168.0.23  | node-to-node mesh | up    | 03:06:45 | Established |
+---------------+-------------------+-------+----------+-------------+

```
