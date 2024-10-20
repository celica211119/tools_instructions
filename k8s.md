# 1 前期准备
## 1.1 组件版本
操作系统及其他工具
|软件|版本|说明|
|---|---|---|
|操作系统||内核版本：4.1。|
|操作系统||内核版本：4.1。|
|cfssl||开源证书管理工具。|

k8s
|软件|版本|说明|
|---|---|---|
|etcd||存储系统，用于保存集群中的相关数据。可以安装在单独的服务器上。|
|docker||容器运行环境。|
|kubelet||master派到node节点代表，管理本机容器；一个集群中每个节点上运行的代理，它保证容器都运行在Pod中， 负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理。|
|kube-apiserver||集群控制的入口，提供HTTP REST服务，同时交给etcd存储，提供认证、授权、访问控制、API注册和发现等机制。|
|kube-controller-manager||Kubernetes集群中所有的资源对象的自动化控制中心，管理集群中藏柜后台任务，一个资源对应一个控制器。|
|kube-scheduler||负责Pod的调度，选择node节点应用部署。|
|kube-proxy||提供网络代理，负载均衡等操作。|
## 1.2 角色说明
|IP|角色|安装组件|
|---|---|---|
|ip1|master1|kubelet，kube-apiserver，kube-controller-manager，kube-scheduler，kube-proxy，docker，etcd，cfssl|
|ip2|master2(下期计划)|kubelet，kube-apiserver，kube-controller-manager，kube-scheduler，kube-proxy，docker，etcd，cfssl|
|ip3|node1|kubelet，kube-proxy，docker，etcd，cfssl|
## 1.3 操作系统初始化配置
在对应节点上执行操作
```
# 根据规划设置主机名
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-master2
hostnamectl set-hostname k8s-node2
```
在所有节点都要执行操作
```
# 关闭系统防火墙
# 临时关闭
systemctl stop firewalld
# 永久关闭
systemctl disable firewalld

# 关闭selinux
# 临时关闭
setenforce 0
# 永久关闭
sed -i 's/enforcing/disabled/' /etc/selinux/config

# 关闭swap
# 临时关闭
swapoff -a
# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 添加hosts
cat >> /etc/hosts << EOF
ip1 k8s-master1
ip2 k8s-master2
ip3 k8s-node1
EOF

# 将桥接的IPV4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# 生效
sysctl --system

# 时间同步
# 使用阿里云时间服务器进行临时同步
yum install ntpdate
ntpdate ntp.aliyun.com

# 下载cfssl相关工具
mkdir /software-cfssl
wget https://pkg.cfssl.org/R1.2/cfssl_linux-arm64 -P /software-cfssl/
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-arm64 -P /software-cfssl/
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-arm64 -P /software-cfssl/
# 给cfssl相关工具赋可执行权限，并复制到对应目录下
cd /software-cfssl/
chmod +x *
cp cfssl_linux-arm64 /usr/local/bin/cfssl
cp cfssljson_linux-arm64 /usr/local/bin/cfssljson
cp cfssl-certinfo_linux-arm64 /usr/bin/cfssl-certinfo

#创建工作目录
mkdir -p ~/TLS/{etcd,k8s}
```
## 1.4 下载各个组件对应的二进制文件
```
待补充
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-arm.tar.gz //待确认
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz //待确认
```
# 2 部署etcd集群
etcd 是一个分布式键值对存储系统，由coreos 开发，内部采用 raft 协议作为一致性算法，用于可靠、快速地保存关键数据，并提供访问。  
通过分布式锁、leader选举和写屏障(write barriers)，来实现可靠的分布式协作。etcd 服务作为Kubernetes集群的主数据库，在安装Kubernetes各服务之前需要首先安装和启动。  
可以与k8s节点复用，也可以部署在k8s机器之外，只要保证apiserver能够连接到。  
|节点名称|IP|
|---|---|
|etcd-1|ip1|
|etcd-2|ip2|
|etcd-3|ip3|
## 2.1 创建自签证书颁发机构(CA)
```
cat > ~/TLS/etcd/ca-config.json << EOF
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
EOF
```
配置说明：  
"ca-config.json"：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数。后续在签名证书时使用某个 profile。  
"signing"：表示该证书可用于签名其它证书。生成的 ca.pem 证书中 CA=TRUE。  
"server auth"：表示client可以用该CA对server提供的证书进行验证。  
"client auth"：表示server可以用该CA对client提供的证书进行验证。  
```
cat > ~/TLS/etcd/ca-csr.json << EOF
{
    "CN": "etcd CA",
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
EOF
```
“CN”：Common Name，从证书中提取该字段作为请求的用户名 (User Name)。浏览器使用该字段验证网站是否合法。  
“O”：Organization，从证书中提取该字段作为请求用户所属的组 (Group)。  
```
# 生成自签CA证书
cd ~/TLS/etcd/
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls
# 结果应该如下
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
## 2.2 使用自签CA签发etcd https证书
```
cd ~/TLS/etcd/
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "ip1",
    "ip2",
    "ip3",
    ],
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
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
ls
# 结果应该如下
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  server.csr  server-csr.json  server-key.pem  server.pem
```
## 2.3 部署etcd集群
### 2.3.1 在master1节点部署
```
# 创建工作目录并复制二进制文件
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar -xf etcd.tar.gz
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
# 创建etcd配置文件
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://ip1:2380"
ETCD_LISTEN_CLIENT_URLS="https://ip1:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://ip1:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://ip1:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://ip1:2380,etcd-2=https://ip2:2380,etcd-3=https://ip3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
配置说明：  
ETCD_NAME：节点名称，集群中唯一。  
ETCD_DATA_DIR：数据目录。  
ETCD_LISTEN_PEER_URLS：集群通讯监听地址。  
ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址。  
ETCD_INITIAL_CLUSTER：集群节点地址。  
ETCD_INITIALCLUSTER_TOKEN：集群Token。  
ETCD_INITIALCLUSTER_STATE：加入集群的状态，new是新集群，existing表示加入已有集群。  
### 2.3.2 systemd管理etcd
```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
 
[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```
### 2.3.3 将master1节点所有生成的文件拷贝到master2节点和node1节点
```
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/

scp -rp /opt/etcd/ ip2:/opt/
scp -rp /usr/lib/systemd/system/etcd.service ip2:/usr/lib/systemd/system/

scp -rp /opt/etcd/ ip3:/opt/
scp -rp /usr/lib/systemd/system/etcd.service ip3:/usr/lib/systemd/system/
```
### 2.3.4 在所有节点查看文件与服务是否存在
```
tree /opt/etcd/
tree /usr/lib/systemd/system/ | grep etcd
```
### 2.3.5 修改master2节点和node1节点中etcd.conf配置文件
```
# master2节点修改为
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://ip2:2380"
ETCD_LISTEN_CLIENT_URLS="https://ip2:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://ip2:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://ip2:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://ip1:2380,etcd-2=https://ip2:2380,etcd-3=https://ip3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

# node1节点修改为

ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://ip3:2380"
ETCD_LISTEN_CLIENT_URLS="https://ip3:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://ip3:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://ip3:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://ip1:2380,etcd-2=https://ip2:2380,etcd-3=https://ip3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
### 2.3.6 启动etcd集群
etcd需要多个节点同时启动，单个启动会卡住。下面每个命令在三台服务器上同时执行。
```
# 启动服务
systemctl daemon-reload
systemctl start etcd
# 查看服务状态
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://ip1:2379,https://ip2:2379,https://ip3:2379" endpoint health --write-out=table
+------------------+--------+-------------+-------+
|     ENDPOINT     | HEALTH |    TOOK     | ERROR |
+------------------+--------+-------------+-------+
| https://ip1:2379 |   true |  9.679688ms |       |
| https://ip2:2379 |   true | 11.282224ms |       |
| https://ip3:2379 |   true |  9.897207ms |       |
+------------------+--------+-------------+-------+
```
如果出现结果不符，使用下面命令进行错误排查
```
less /var/log/message
journalctl -u etcd
```
# 3 部署docker
所有节点使用相同步骤
## 3.1 下载并解压二进制包
```
tar -xf docker.tgz
mv docker/* /usr/bin/
```
## 3.2 配置镜像加速
```
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```
## 3.3 docker.service配置
```
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/bin/dockerd --selinux-enabled=false --insecure-registry=127.0.0.1
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
#TasksMax=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
EOF
```
## 3.4 启动docker并设置开机启动
```
systemctl daemon-reload
systemctl start docker
```
