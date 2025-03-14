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
      "ST": "shanghai",
      "O": "kubernetes",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "87600h"
  }
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
创建kube-apiserver-csr.json
```
{
  "CN": "kubernetes",
  "hosts": [
    "10.0.0.1",
    "127.0.0.1",
    集群ip1,
    集群ip2,
    集群ip3,
    集群ip4,
    预留ip1,
    预留ip2,
    预留ip3,
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
      "L": "shanghai",
      "ST": "shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```
生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
```
### 2.2.2 创建token文件
```
cat > /opt/kubernetes/cfg/token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
### 2.2.3 配置kube-apiserver
创建kube-apiserver.conf
```
vi /opt/kubernetes/cfg/kube-apiserver.conf
```
添加以下内容
```
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
    --anonymous-auth=false \
    --bind-address=本机ip \
    --secure-port=6443 \
    --advertise-address=本机ip \
    --authorization-mode=RBAC,Node \
    --runtime-config=api/all=true \
    --enable-bootstrap-token-auth=true \
    --service-cluster-ip-range=10.0.0.0/24 \
    --token-auth-file=/opt/kubernetes/cfg/token.csv \
    --service-node-port-range=30000-50000 \
    --kubelet-client-certificate=/opt/kubernetes/ssl/kube-apiserver.pem \
    --kubelet-client-key=/opt/kubernetes/ssl/kube-apiserver-key.pem \
    --tls-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem \
    --tls-private-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
    --client-ca-file=/opt/kubernetes/ssl/ca.pem \
    --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
    --service-account-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
    --service-account-issuer=https://kubernetes.default.svc.cluster.local \
    --etcd-servers=etcd服务地址1,etcd服务地址2,etcd服务地址3,etcd服务地址4 \
    --etcd-cafile=/opt/etcd/ssl/ca.pem \
    --etcd-certfile=/opt/etcd/ssl/server.pem \
    --etcd-keyfile=/opt/etcd/ssl/server-key.pem \
    --allow-privileged=true \
    --audit-log-maxage=30 \
    --audit-log-maxbackup=3 \
    --audit-log-maxsize=100 \
    --audit-log-path=/opt/kubernetes/logs/k8s-audit.log \
    --v=4 \
    --event-ttl=1h \
    --requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
    --proxy-client-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem \
    --proxy-client-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \
    --requestheader-allowed-names=kubernetes \
    --requestheader-extra-headers-prefix=X-Remote-Extra- \
    --requestheader-group-headers=X-Remote-Group \
    --requestheader-username-headers=X-Remote-User \
    --enable-aggregator-routing=true"
```
KUBE_APISERVER_OPTS说明
```
--logtostderr 启用日志，false为写入文件不写入stderr
--v 日志等级
--log-dir 日志目录
--etcd-servers etcd集群地址
--bind-address 本机ip
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
### 2.2.4 注册kube-apiserver服务
创建kube-apiserver服务
```
vi /usr/lib/systemd/system/kube-apiserver.service
```
添加以下内容
```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
重新加载服务
```
systemctl daemon-reload
```
### 2.2.5 启动kube-apiserver服务并查看状态
```
systemctl start kube-apiserver
systemctl status kube-apiserver
```
查看日志观察状态排查错误
```
journalctl -u kube-apiserver
```
### 2.2.6 验证kube-apiserver服务状态
```
curl --insecure https://ip:6443/
```
服务正常则会显示以下内容
```
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
## 2.3 部署kube-controller-manager
### 2.2.1 在/opt/kubernetes/ssl/目录下创建所需证书
创建kube-controller-manager-csr.json
```
{
  "CN": "system:kube-controller-manager",
  "hosts": [ 
    "10.0.0.1",
    "127.0.0.1",
    集群ip1,
    集群ip2,
    集群ip3,
    集群ip4,
    预留ip1,
    预留ip2,
    预留ip3,
    "10.10.10.1",
    "10.255.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluste.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "shanghai",
      "ST": "shanghai",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
```
生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```
### 2.3.2 配置kube-controller-manager
创建kube-controller-manager.conf
```
vi /opt/kubernetes/cfg/kube-controller-manager.conf
```
添加以下内容
```
KUBE_CONTROLLER_MANAGER_OPTS=" \
  --bind-address=127.0.0.1 \
  --kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.0.0.0/24 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/opt/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --v=2"
```
KUBE_CONTROLLER_MANAGER_OPTS说明
```
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
### 2.3.3 生成kubeconfig文件
```
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://本机ip:6443"
```
设置集群参数
```
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
```
设置客户端认证参数
```
kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/opt/kubernetes/ssl/kube-controller-manager.pem \
  --client-key=/opt/kubernetes/ssl/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
```
设置上下文参数
```
kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
```
设置默认上下文
```
kubectl config use-context system:kube-controller-manager --kubeconfig=${KUBE_CONFIG}
```
### 2.3.4 注册kube-controller-manager服务
创建kube-controller-manager服务
```
vi /usr/lib/systemd/system/kube-controller-manager.service
```
添加以下内容
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
重新加载服务
```
systemctl daemon-reload
```

### 2.3.5 启动kube-controller-manager服务并查看状态
```
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```
查看日志观察状态排查错误
```
journalctl -u kube-controller-manager
```
## 2.4 部署kube-scheduler
### 2.4.1 在/opt/kubernetes/ssl/目录下创建所需证书
创建kube-scheduler-csr.json
```
{
  "CN": "system:kube-scheduler",
  "hosts": [ 
    "10.0.0.1",
    "127.0.0.1",
    集群ip1,
    集群ip2,
    集群ip3,
    集群ip4,
    预留ip1,
    预留ip2,
    预留ip3,
    "10.10.10.1",
    "10.255.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluste.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "shanghai",
      "ST": "shanghai",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
```
生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
### 2.4.2 配置kube-scheduler
创建kube-scheduler.conf
```
vi /opt/kubernetes/cfg/kube-scheduler.conf
```
添加以下内容
```
KUBE_SCHEDULER_OPTS="--bind-address=127.0.0.1 \
  --kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --v=2"
```
### 2.4.3 生成kubeconfig文件
```
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://本机ip:6443"
```
设置集群参数
```
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
```
设置客户端认证参数
```
kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/opt/kubernetes/ssl/kube-scheduler.pem \
  --client-key=/opt/kubernetes/ssl/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
```
设置上下文参数
```
kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
```
设置默认上下文
```
kubectl config use-context system:kube-scheduler --kubeconfig=${KUBE_CONFIG}
```
### 2.4.4 注册kube-scheduler服务
创建kube-controller-manager服务
```
vi /usr/lib/systemd/system/kube-scheduler.service
```
添加以下内容
```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
重新加载服务
```
systemctl daemon-reload
```
### 2.4.5 启动kube-scheduler服务并查看状态
```
systemctl start kube-scheduler
systemctl status kube-scheduler
```
查看日志观察状态排查错误
```
journalctl -u kube-controller-manager
```
## 2.5 配置kubectl
### 2.5.1 在/opt/kubernetes/ssl/目录下创建所需证书
创建admin-csr.json
```
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
      "L": "shanghai",
      "ST": "shanghai",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```
生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```
### 2.5.2 生成kubeconfig文件
```
KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://本机ip:6443"
```
设置集群参数
```
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
```
设置客户端认证参数
```
kubectl config set-credentials admin \
  --client-certificate=/opt/kubernetes/ssl/admin.pem \
  --client-key=/opt/kubernetes/ssl/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
```
设置上下文参数
```
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=${KUBE_CONFIG}
```
设置默认上下文
```
kubectl config use-context kubernetes --kubeconfig=${KUBE_CONFIG}
```
授权kubelet-bootstrap用户允许请求证书（api-service token文件中用户）
```
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```
授权kubernetes证书访问kubelet api权限
```
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
```
### 2.5.3 查看集群组件状态
```
kubectl get cs
kubectl cluster-info
kubectl get componentstatuses
kubectl get all --all-namespaces
```
# 3 master节点扩容
## 3.1 复制相关文件到目标机器
```
scp -r /opt/kubernetes root@扩容机器ip:/opt/
scp /usr/lib/systemd/system/kube* root@扩容机器ip:/usr/lib/systemd/system
scp -r ~/.kube root@目标ip:~
```
## 3.2 在扩容机器上修改相关文件
修改apiserver配置文件为本机IP
```
vi /opt/kubernetes/cfg/kube-apiserver.conf
```
修改连接master为本机IP
```
vi ~/.kube/config
```
在第5行修改为本机ip
```
apiVersion: v1
clusters:
- cluster:
  certificate-authority-data:...
  server: https://本机ip:6443
```
## 3.3 启动服务
```
systemctl daemon-reload
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```
