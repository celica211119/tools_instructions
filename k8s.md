# 1 前期准备
## 1.1 组件版本
操作系统及其他工具
|软件|版本|说明|
|---|---|---|
|操作系统|统信OS|内核版本：4.19.90-2305.1.0.0199.56.uel20.aarch64。|
|操作系统|麒麟OS|内核版本：4.19.90-25.36.v2101.ky10.aarch64。|
|cfssl|v1.6.4|开源证书管理工具。|
|calico|v3.20.6|网络插件。|

k8s
|软件|版本|说明|
|---|---|---|
|etcd|v3.5.9|存储系统，用于保存集群中的相关数据。可以安装在单独的服务器上。|
|docker|v24.0.5|容器运行环境。|
|kubelet|v1.29.3|master派到node节点代表，管理本机容器；一个集群中每个节点上运行的代理，它保证容器都运行在Pod中， 负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理。|
|kube-apiserver|v1.29.3|集群控制的入口，提供HTTP REST服务，同时交给etcd存储，提供认证、授权、访问控制、API注册和发现等机制。|
|kube-controller-manager|v1.29.3|Kubernetes集群中所有的资源对象的自动化控制中心，管理集群中藏柜后台任务，一个资源对应一个控制器。|
|kube-scheduler|v1.29.3|负责Pod的调度，选择node节点应用部署。|
|kube-proxy|v1.29.3|提供网络代理，负载均衡等操作。|
## 1.2 角色说明
|ip|角色|安装组件|
|---|---|---|
|ip1|master1|kubectl，kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，docker，etcd，cfssl|
|ip2|master2|kubectl，kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，docker，etcd，cfssl|
|ip3|worker1|kubelet，kube-proxy，Docker，cri-dockerd，etcd，cfssl|
## 1.3 操作系统初始化配置
在对应节点上执行操作
```
# 根据规划设置主机名
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-master2
hostnamectl set-hostname k8s-worker1

# 设置静态ip
vi /etc/sysconfig/network-scripts/ifcfg-对应网卡名
```
在所有节点都要执行操作
```
# 设置主机名和ip地址解析
vi /etc/hosts
# 在文件末尾添加以下内容
ip1 k8s-master1
ip2 k8s-master2
ip3 k8s-worker1

# 关闭系统防火墙并禁止开机启动
systemctl disable --now firewalld

# 关闭selinux
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sestatus

# 关闭swap
# 临时关闭
swapoff -a
# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 时间同步
yum install ntpdate
# 验证时间同步
ntpdate ntp.aliyun.com
ntpdate -su 192.168.0.245 //
echo $?
# 同步成功显示为
0
# 设置自动同步
crontab -e
* */1 * * * ntpdate -su 192.168.0.245 >> /tmp/ntpdate.log

# 开启内核路由转发和网桥过滤
vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
# 加载模块并查看
modprobe br_netfilter && lsmod | grep br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf

# 添加主机ipvs管理工具及模块
yum -y install ipvsadm ipset sysstat conntrack libseccomp
vi /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
modprobe -- br_netfilter
#添加权限并测试运行
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

# 内核升级

# 设置ssh免密登录



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
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-arm64.tar.gz //待确认
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz //待确认
wget https://storage.googleapis.com/kubernetes-release/release/v1.29.3/kubernetes-server-linux-arm64.tar.gz
wget https://github.com/projectcalico/calicoctl/releases/download/v3.20.6/calicoctl //最新，2022.08.02更新
```
# 2 部署etcd集群
etcd 是一个分布式键值对存储系统，由coreos 开发，内部采用 raft 协议作为一致性算法，用于可靠、快速地保存关键数据，并提供访问。  
通过分布式锁、leader选举和写屏障(write barriers)，来实现可靠的分布式协作。etcd 服务作为Kubernetes集群的主数据库，在安装Kubernetes各服务之前需要首先安装和启动。  
可以与k8s节点复用，也可以部署在k8s机器之外，只要保证apiserver能够连接到。  
|节点名称|ip|
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
      "kubernetes": {
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
```
配置说明：
"ca-config.json"：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数。后续在签名证书时使用某个 profile。
"signing"：表示该证书可用于签名其它证书。生成的 ca.pem 证书中 CA=TRUE。
"server auth"：表示client可以用该CA对server提供的证书进行验证。
"client auth"：表示server可以用该CA对client提供的证书进行验证。
```
```
cat > ~/TLS/etcd/ca-csr.json << EOF
{
    "CN": "kubernetes CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "kubernetes",
            "OU": "CN"
        }
    ]
}
EOF
```
```
配置说明：
"CN"：Common Name，从证书中提取该字段作为请求的用户名 (User Name)。浏览器使用该字段验证网站是否合法。
"O"：Organization，从证书中提取该字段作为请求用户所属的组 (Group)。
```
```
# 生成自签CA证书
cd ~/TLS/etcd/
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls
# 结果应该如下
ca-config.json ca.csr ca-csr.json ca-key.pem ca.pem
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
ETCD_DATA_DIR="/opt/etcd/data/default.etcd"
ETCD_INITIAL_CLUSTER="etcd-1=http://ip1:2380,etcd-2=http://ip2:2380,etcd-3=http://ip3:2380"
ETCD_UNSUPPORTED_ARCH=arm64
EOF
```
```
配置说明：
ETCD_NAME：节点名称，集群中唯一。
ETCD_DATA_DIR：数据目录。
ETCD_INITIAL_CLUSTER：集群节点地址。
ETCD_UNSUPPORTED_ARCH：集群架构。
### 2.3.2 systemd管理etcd
```
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
--listen-peer-urls http://ip1:2380 \
--listen-client-urls http://ip1:2379,http://127.0.0.1:2379 \
--initial-advertise-peer-urls http://ip1:2380 \
--advertise-client-urls http://ip1:2379 \
--initial-cluster-token etcd-cluster \
--initial-cluster-state new \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
### 2.3.3 将master1节点所有生成的文件拷贝到master2节点和node1节点
```
scp -rp /opt/etcd/ ip2:/opt/
scp -rp /usr/lib/systemd/system/etcd.service ip2:/usr/lib/systemd/system/

scp -rp /opt/etcd/ ip3:/opt/
scp -rp /usr/lib/systemd/system/etcd.service ip3:/usr/lib/systemd/system/
```
### 2.3.4 在所有节点查看文件与服务是否存在
```
tree /opt/etcd/
tree /usr/lib/systemd/system/ | grep etcd
查看数据库目录是否有其他版本的数据
ls /opt/etcd/data/
rm -r /opt/etcd/data/default.etcd
```
### 2.3.5 修改master2节点和node1节点中etcd.conf配置文件和service
### 2.3.6 启动etcd集群
etcd需要多个节点同时启动，单个启动会卡住。下面每个命令在三台服务器上同时执行。
```
# 启动服务
systemctl daemon-reload
systemctl start etcd
# 查看集群成员
etcdctl member list -w table
# 查看各个节点状态
+------------------+---------+--------+-----------------+-----------------+------------+
|        ID        | STATUS  |  NAME  |   PEER ADDRS    |  CLIENT ADDRS   | IS LEARNER |
+------------------+---------+--------+-----------------+-----------------+------------+
| adff72f24ac33f4b | started | etcd-1 | http://ip1:2380 | http://ip1:2379 |      false |
| c47ea72f05f56814 | started | etcd-3 | http://ip2:2380 | http://ip2:2379 |      false |
| d36b782983b31e67 | started | etcd-2 | http://ip3:2380 | http://ip3:2379 |      false |
+------------------+---------+--------+-----------------+-----------------+------------+
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="http://ip1:2379,http://ip2:2379,http://ip3:2379" endpoint health --write-out=table
+-----------------+--------+-------------+-------+
|     ENDPOINT    | HEALTH |    TOOK     | ERROR |
+-----------------+--------+-------------+-------+
| http://ip1:2379 |   true |  9.679688ms |       |
| http://ip2:2379 |   true | 11.282224ms |       |
| http://ip3:2379 |   true |  9.897207ms |       |
+-----------------+--------+-------------+-------+
# 查看集群成员状态
etcdctl member list -w table
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
# 4 k8s部署master1节点
## 4.1 创建工作目录与安装二进制文件
```
tar xf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/

# master节点执行
cp kube-apiserver kube-controller-manager kube-scheduler /opt/kubernetes/bin
cp kubectl /usr/bin/

# work节点执行
cp kubelet kube-proxy /opt/kubernetes/bin
```
## 4.2 master节点部署kube-apiserver
### 4.2.1 生成kube-apiserver证书
```
# ca-config.json内容和配置etcd时一致
cd ~/TLS/k8s
cat > kube-apiserver-csr.json <<"EOF"
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.0.2",
      "192.168.0.160",
      "192.168.0.191",
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
            "O": "kubernetes",
            "OU": "CN"
        }
    ]
}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
ls
# 结果应该如下
ca-config.json ca.csr ca-csr.json ca-key.pem ca.pem kube-apiserver.csr kube-apiserver-csr.json kube-apiserver-key.pem kube-apiserver.pem
```
### 4.2.2 创建TLS机制所需TOKEN
TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。  
为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。
集群内所有master节点保证token文件的内容一致
```
cat > /opt/kubernetes/cfg/token.csv << EOF
token,用户名,UID,用户组
例子：$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
### 4.2.3 配置kube-apiserver
```
cat > /etc/kubernetes/kube-apiserver.conf << "EOF"
KUBE_APISERVER_OPTS="--v=4 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=http://192.168.0.2:2379,http://192.168.0.160:2379,http://192.168.0.191:2379 \
--bind-address=192.168.0.2 \
--secure-port=6443 \
--advertise-address=192.168.0.2 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-32767 \
--kubelet-client-certificate=/opt/kubernetes/ssl/kube-apiserver.pem \
--kubelet-client-key=/opt/kubernetes/ssl/kube-apiserver-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem \
--tls-private-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--service-account-issuer=api \
--service-account-signing-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--proxy-client-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
# cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```
```
KUBE_APISERVER_OPTS说明
--logtostderr 启用日志，false为写入文件不写入stderr
--v 日志等级
--log-dir 日志目录
--etcd-servers etcd集群地址
--bind-address 主机ip
--secure-port 监听端口，默认为8080
--advertise-address 集群通告地址
--allow-privileged 启用授权，true为启用
--service-cluster-ip-range 集群中service虚拟ip地址范围，以CIDR格式表示，不能与物理机ip地址有重合
--enable-admission-plugins 集群准入控制设置
--authorization-mode 认证授权
--enable-bootstrap-token-auth 启用TLS bootstrap机制，true为启用
--token-auth-file bootstrap token文件
--service-node-port-range service可使用的物理机端口范围，默认为30000-32767
--kubelet-client-certificate 访问kubelet客户端证书
--kubelet-client-key apiserver访问kubelet客户端证书
--tls-cert-file apiserver的https证书
--tls-private-key-file apiserver的https证书
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \\
--etcd-cafile etcd 集群证书
--etcd-certfile etcd 集群证书
--etcd-keyfile etcd 集群证书
--requestheader-client-ca-file 聚合网关
--proxy-client-cert-file 聚合网关
--proxy-client-key-file 聚合网关
--requestheader-allowed-names 聚合网关
--requestheader-extra-headers-prefix 聚合网关
--requestheader-group-headers 聚合网关
--requestheader-username-headers 聚合网关
--enable-aggregator-routing 聚合网关
--audit-log-maxage 审计日志
--audit-log-maxbackup 审计日志
--audit-log-maxsize 审计日志
--audit-log-path 审计日志
```
### 4.2.4 systemd管理kube-apiserver
```
cat > /usr/lib/systemd/system/kube-apiserver.service << "EOF"
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl start kube-apiserver
```
### 4.2.5 验证apiserver访问
```
# 所有master节点完成上述操作后
curl --insecure https://ip1:6443/
curl --insecure https://ip2:6443/

# 结果都应为：
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```
## 4.5 kubectl
### 4.5.1 生成kubectl证书
```
# cd ~/TLS/k8s
# vi admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "system"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
```
### 4.5.3 生成kubeconfig
```
# kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.2:6443 --kubeconfig=kube.config
# kubectl config set-credentials admin --client-certificate=./admin.pem --client-key=./admin-key.pem --embed-certs=true --kubeconfig=kube.config
# kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
# kubectl config use-context kubernetes --kubeconfig=kube.config
# mkdir /root/.kube
# cp ./kube.config /root/.kube/config
# cat /root/.kube/config
```
### 4.5.2 通过kubectl查看当前集群组件状态
```
# kubectl get nodes
# kubectl cluster-info
# kubectl get cs
```
### 4.5.4 授权kubelet-bootstrap用户允许请求证书
```
# kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```
## 4.3 master节点部署kube-controller-manager
### 4.3.1 生成kube-controller-manager证书
```
# cd ~/TLS/k8s
cat > kube-controller-manager-csr.json << "EOF"
{
  "CN": "system:kube-controller-manager",
  "hosts": [    
      "127.0.0.1",
      "192.168.0.2",
      "192.168.0.160",
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:kube-controller-manager",
      "OU": "system"
    }
  ]
}
EOF
```
```
配置说明：
hosts：包含所有kube-controller-manager节点 ip。
```
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
ls kube-controller-manager*
# 结果应该如下
ca-kube-controller-manager.csr kube-controller-manager-key.pem kube-controller-manager-csr.json kube-controller-manager.pem
```
### 4.3.2 配置kube-controller-manager
```
# vi /opt/kubernetes/cfg/kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--allocate-node-cidrs=true \
--bind-address=127.0.0.1 \
--cluster-name=kubernetes \
--cluster-cidr=10.244.0.0/16 \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-duration=87600h0m0s \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
--controllers=*,bootstrapsigner,tokencleaner \
--feature-gates=RotateKubeletServerCertificate=true \
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
--leader-elect=true \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--secure-port=10257 \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--service-cluster-ip-range=10.0.0.0/24 \
--use-service-account-credentials=true \
--v=2
```
```
KUBE_CONTROLLER_MANAGER_OPTS说明
--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect 该组件启动多个时自动选举
--kubeconfig 设置与apiserver连接的相关设置
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file 自动为kubelet颁发证书的CA
--cluster-signing-key-file 自动为kubelet颁发证书的CA
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s
```
### 4.3.3 生成kube-controller-manager.kubeconfig
```
# kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.2:6443 --kubeconfig=kube-controller-manager.kubeconfig
# kubectl config set-credentials system:kube-controller-manager --client-certificate=./kube-controller-manager.pem --client-key=./kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
# kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
# kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
# cat /opt/kubernetes/cfg/kube-controller-manager.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t
    server: ip:port
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-controller-manager
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-controller-manager
  user:
    client-certificate-data: LS0t
    client-key-data: LS0t
```
### 4.3.4 systemd管理kube-controller-manager
```
# vi /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-controller-manager
```
## 4.4 部署kube-scheduler
### 4.4.1 生成kube-scheduler证书
```
# cd ~/TLS/k8s
# vi kube-scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:kube-scheduler",
      "OU": "system"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
# ls kube-scheduler*
kube-scheduler.csr  kube-scheduler-csr.json  kube-scheduler-key.pem  kube-scheduler.pem
```
### 4.4.2 配置kube-scheduler
```
vi /opt/kubernetes/cfg/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
--leader-elect=true \
--v=2
```
### 4.4.3 生成kube-scheduler.kubeconfig
```
# kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.2:6443 --kubeconfig=kube-scheduler.kubeconfig
# kubectl config set-credentials system:kube-scheduler --client-certificate=./kube-scheduler.pem --client-key=./kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
# kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
# kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
# cat /opt/kubernetes/cfg/kube-scheduler.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t
    server: ip:port
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kube-scheduler
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kube-scheduler
  user:
    client-certificate-data: LS0t
    client-key-data: LS0t
```
### 4.4.4 systemd管理kube-scheduler
```
# vi /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-scheduler
```
# 4 k8s部署work节点
## 4.1 master节点部署kubelet
### 4.1.1 配置kubelet
```
# vi /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-master1 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-arm64:3.0"
```
```
KUBELET_OPTS说明
--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override 显示名称，集群唯一(不可重复)，设置本Node的名称
--network-plugin 启用CNI
--kubeconfig 空路径，会自动生成，后面用于连接 apiserver。设置与 API Server 连接的相关配置，可以与 kube-controller-manager 使用的 kubeconfig 文件相同
--bootstrap-kubeconfig 首次启动向apiserver申请证书
--config 配置文件参数
--cert-dir kubelet证书目录
--pod-infra-container-image 管理Pod网络容器的镜像 init container
```
### 4.1.2 添加参数文件
```
# vi /opt/kubernetes/cfg/kubelet-config.yml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
```
### 4.1.3 kubelet加入集群引导kubeconfig文件
```
# KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
# KUBE_APISERVER="ip:port"
# TOKEN="与/opt/kubernetes/cfg/token.csv里保持一致 "
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
# cat bootstrap.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t
    server: https://192.168.176.140:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-auth
  user:
    client-certificate: /opt/kubernetes/ssl/kubelet-client-current.pem
    client-key: /opt/kubernetes/ssl/kubelet-client-current.pem
```
### 4.1.4 systemd管理kubelet
```
# vi /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kubelet
```
### 4.1.5 允许kubelet证书申请并加入集群
```
# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-_ljl7ZmiNPDjavyhJA6aMJ4bnZHp7WAml3XPEn8BzoM   28s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
# kubectl certificate approve node-csr-_ljl7ZmiNPDjavyhJA6aMJ4bnZHp7WAml3XPEn8BzoM
certificatesigningrequest.certificates.k8s.io/node-csr-_ljl7ZmiNPDjavyhJA6aMJ4bnZHp7WAml3XPEn8BzoM approved
# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-_ljl7ZmiNPDjavyhJA6aMJ4bnZHp7WAml3XPEn8BzoM   2m31s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
# kubectl get nodes
NAME          STATUS     ROLES    AGE   VERSION
k8s-master1   NotReady   <none>   62s   v1.20.15
```
## 4.2 master节点部署kube-proxy
### 4.2.1 生成kubectl证书
```
# cd ~/TLS/k8s
# vi kube-proxy-csr.json
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
# ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
```
### 4.2.2 配置kube-proxy
```
# vi /opt/kubernetes/cfg/kube-proxy.conf
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
```
### 4.2.3 配置参数文件
```
# vi /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master1
clusterCIDR: 10.244.0.0/16
```
### 4.2.4 生成kube-proxy.kubeconfig
```
# KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
# KUBE_APISERVER="ip:port"
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```
### 4.2.5 systemd管理kube-proxy
```
# vi /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
After=network.target
Requires=network.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-proxy
```
## 4.3 master节点部署网络组件(Calico)
```
# chmod +x calicoctl
# mv calicoctl /usr/local/bin/
# wget https://docs.projectcalico.org/archive/v3.20/manifests/calico.yaml
# kubectl apply -f calico.yaml
# kubectl get pods -n kube-system
# kubectl get pods -n kube-system
# kubectl get nodes
```
## 4.4 master节点授权apiserver访问kubelet
```
# vi apiserver-to-kubelet-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
# kubectl apply -f apiserver-to-kubelet-rbac.yaml
```
## 4.4 增加work节点
### 4.4.1 给work节点添加必要的文件
在master节点执行
```
# scp -rp /opt/kubernetes Work节点ip:/opt/
# scp -rp /usr/lib/systemd/system/{kubelet,kube-proxy} Work节点ip:/usr/lib/systemd/system
```
在work节点执行
```
# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
# rm -f /opt/kubernetes/ssl/kubelet*
# vi /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=workNode节点对应的主机名 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-arm64:3.0"
# vi /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: workNode节点对应的主机名
clusterCIDR: 10.244.0.0/16
```
### 4.4.2 systemd管理kubelet和kube-proxy
```
# systemctl daemon-reload
# systemctl start kubelet kube-proxy
```
## 4.5 在master节点同意新的节点kubelet证书申请
```
# kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-EAPxdWuBmwdb4CJQeWRfDLi2cvzmQZ9VIh3uSgcz1Lk   98s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-_ljl7ZmiNPDjavyhJA6aMJ4bnZHp7WAml3XPEn8BzoM   79m    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-e2BoSziWYL-gcJ-0BIcXXY1-wmR8YBlojENV6-FpIJU   102s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
# kubectl certificate approve node-csr-EAPxdWuBmwdb4CJQeWRfDLi2cvzmQZ9VIh3uSgcz1Lk
certificatesigningrequest.certificates.k8s.io/node-csr-EAPxdWuBmwdb4CJQeWRfDLi2cvzmQZ9VIh3uSgcz1Lk approved
# kubectl certificate approve node-csr-e2BoSziWYL-gcJ-0BIcXXY1-wmR8YBlojENV6-FpIJU
certificatesigningrequest.certificates.k8s.io/node-csr-e2BoSziWYL-gcJ-0BIcXXY1-wmR8YBlojENV6-FpIJU approved
# kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
k8s-master1   Ready    <none>   81m    v1.20.15
k8s-node1     Ready    <none>   98s    v1.20.15
k8s-node2     Ready    <none>   118s   v1.20.15
```
# 5 master部署Dashboard
## 5.1 配置参数文件
```
#vi kubernetes-dashboard.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.2.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.6
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
```
## 5.2 部署并查看结果
```
# kubectl apply -f kubernetes-dashboard.yaml
# kubectl get pods,svc -n kubernetes-dashboard
```
## 5.3 创建service account并绑定默认cluster-admin管理员集群角色
```
# kubectl create serviceaccount dashboard-admin -n kube-system
# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
# kubectl create serviceaccount dashboard-admin -n kube-system
serviceaccount/dashboard-admin created
# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created
# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-cd77q
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: c713b50a-c11d-4708-9c7b-be7835bf53b9
 
Type:  kubernetes.io/service-account-token
 
Data
====
ca.crt:     1359 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImJoMTlpTE1idkRiNHpqcHoxanctaGoxVFRrOU80S1pQaURqbTJnMy1xUE0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tY2Q3N3EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzcxM2I1MGEtYzExZC00NzA4LTljN2ItYmU3ODM1YmY1M2I5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.BV0LA43Wqi3Chxpz9GCyl1P0C924CZ5FAzFSfN6dNmC0kKEdK-7mPzWhoLNdRxXHTCV-INKU5ZRCnl5xekmIKHfONtC-DbDerFyhxdpeFbsDO93mfH6Xr1d0nVaapMh9FwQM0cOtyIWbBWkXzVKPu8EDVkS-coJZyJoDoaTlHFZxaTx3VkrxK_eLz-a9HfWfEjPVKgCf6XJgOt9y0tcCgAA9DEhpbNbOCTM7dvLD_mIwPm8HIId8-x0jkr4cNQq6wLcULhSPZg4gghYZ8tlLrjixzhYahz4Q1RuhYDCbHKLj4PLltXZTkJ3GHo4upG3ehto7A6zuhKGL21KzDW80NQ
```

https://blog.csdn.net/qq_54494363/article/details/131833096




y41完成k8s卸载
sudo find / -name kubeadm
cd .....
sudo ./kubeadm resst --force
sudo rm /var/lib/rpm/__db*
sudo rpmdb --rebuilddb
sudo yum remove -y kubelet.aarch64 kubeadm.aarch64 kubectl.aarch64

