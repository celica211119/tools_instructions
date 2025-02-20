# 0 操作系统初始化配置
关闭selinux
```
setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
```
关闭apparmor
```
systemctl stop apparmor
systemctl disable apparmor
```
关闭swap
```
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```
将桥接的IPV4流量传递到iptables的链
```
vi /etc/sysctl.d/k8s.conf
```
添加以下内容
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
生效
```
sysctl --system
```
时间同步
```
ntpdate ntp.aliyun.com
ntpdate -su 192.168.0.245
```
验证同步成功
```
echo $?
0
```

# 1 下载二进制文件
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.29.3/kubernetes-server-linux-amd64.tar.gz
tar -zxvf kubernetes-server-linux-amd64.tar.gz
```

# 2 部署k8s-master
## 2.1 在master1节点部署
创建工作目录并复制二进制文件
```
mkdir /opt/kubernetes/{bin,cfg,ssl} -p
cp kube-apiserver kube-controller-manager kube-scheduler /opt/kubernetes/bin
cp kubectl /usr/bin/
```
## 2.2 部署kube-apiserver
### 2.2.1 在/opt/kubernetes/ssl/目录下创建所需证书
创建ca-config.json
```
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },		
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```
配置说明
```
"ca-config.json"：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数。后续在签名证书时使用某个 profile。
"signing"：表示该证书可用于签名其它证书。生成的 ca.pem 证书中 CA=TRUE。
	"server auth"：表示client可以用该CA对server提供的证书进行验证。
	"client auth"：表示server可以用该CA对client提供的证书进行验证。
```
创建ca-csr.json
```
{
    "CN": "kubernetes CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "shanghai",
            "ST": "shanghai"
        }
    ]
}
```
配置说明
```
	"CN"：Common Name，从证书中提取该字段作为请求的用户名 (User Name)。浏览器使用该字段验证网站是否合法。
	"O"：Organization，从证书中提取该字段作为请求用户所属的组 (Group)。
```
生成自签CA证书
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```


