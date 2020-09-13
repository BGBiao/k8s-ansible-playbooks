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


