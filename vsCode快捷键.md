|快捷键 |功能 |说明 |
|--- |--- |--- |
|F1 |显示所有命令 | |
|Ctrl+Shift+P |显示所有命令 | |
|Ctrl+Shift+N |新建窗口 | |
|Ctrl+Shift+W |关闭窗口 | |
|Ctrl+， |在工作区打开设置页面 | |
|Ctrl+K Ctrl+S |在工作区打开键盘快捷键页面 | |
|Ctrl+X |剪切当前行 |光标未选择内容 |
|Ctrl+C |复制当前行 |光标未选择内容 |
|Alt+↑ |所选行上移一行 | |
|Alt+↓ |所选行下移一行 | |
|Shift+Alt+↑ |复制所选行内容插入到上一行 | |
|Shift+Alt+↓ |复制所选行内容插入到下一行 | |
|Ctrl+Enter |在光标所在行下方插入空行 | |
|Ctrl+Shift+Enter |在光标所在行上方插入空行 | |
|Ctrl+Shift+\\ |转到光标所在括号作用域的末尾 |已在末尾跳转到开头 |
|Ctrl+] |所选行全部向左缩进一个单位 | |
|Ctrl+[ |所选行全部向右缩进一个单位 | |
|Home |光标移动到所在行开头 | |
|End |光标移动到所在行末尾 | |
|Ctrl+Home |光标移动到文件开头 | |
|Ctrl+End |光标移动到文件末尾 | |
|Ctrl+↑ |工作区向上滚动一行显示 |不移动光标位置 |
|Ctrl+↓ |工作区向下滚动一行显示 |不移动光标位置 |
|Alt+PgUp |工作区向上滚动一页显示 |不移动光标位置 |
|Alt+PgDn |工作区向下滚动一页显示 |不移动光标位置 |
|Ctrl+Shift+[ |折叠当前区域 | |
|Ctrl+Shift+] |展开当前区域 | |
|Ctrl+K Ctrl+[ |递归折叠当前区域与所有子区域 | |
|Ctrl+K Ctrl+] |递归展开当前区域与所有子区域 | |
|Ctrl+K Ctrl+0 |递归折叠当前文件所有区域 | |
|Ctrl+K Ctrl+J |递归展开当前文件所有区域 | |
|Ctrl+K Ctrl+C |所有选中行开头添加注释标记 | |
|Ctrl+K Ctrl+U |所有选中行开头删除注释标记 | |
|Shift+Alt+A |对选中部分添加块注释标记 |已有标记则删除当前标记 |
|Alt+Z |切换过长文本自动换行 | |
|Ctrl+T |选择并转到项目目录中的指定符号 | |
|Ctrl+P |选择并转到项目目录中的指定文件 | |
|Ctrl+G |转到当前文件指定位置 |行号,字符号 |
|Ctrl+Shift+O |转到当前文件指定符号 | |
|Ctrl+Shift+M |跳转到问题面板 |若该面板隐藏，选择其他面板后将重新隐藏 |
|F8 |跳转到下一个错误或警告的位置 | |
|Shift+F8 |跳转到上一个错误或警告的位置 | |
|Ctrl+Shift+Tab |查看工作区所有文件 | |
|Alt+← |移动到当前文件工作区左边的文件 | |
|Alt+→ |移动到当前文件工作区右边的文件 | |
















y41完成k8s卸载
sudo find / -name kubeadm
cd .....
sudo ./kubeadm resst --force
sudo rm /var/lib/rpm/__db*
sudo rpmdb --rebuilddb
sudo yum remove -y kubelet.aarch64 kubeadm.aarch64 kubectl.aarch64



# 下载k8s
```
# wget https://storage.googleapis.com/kubernetes-release/release/v1.29.3/kubernetes-server-linux-arm64.tar.gz
# tar -zxvf ./kubernetes-server-linux-arm64.tar.gz
# cp ./kubernetes/server/bin/kube-apiserver ./kubernetes/server/bin/kube-controller-manager ./kubernetes/server/bin/kube-scheduler /opt/kubernetes/bin
# cp ./kubernetes/server/bin/kubectl /usr/bin

```
# 查看版本信息
```
# ./kubectl version --client --output=yaml
```
# 安装 kubectl
```
# chmod +x kubectl
# mkdir -p ~/.local/bin
# mv ./kubectl ~/.local/bin/kubectl
将 ~/.local/bin 附加到 $PATH
```
# 生成自签证书(CA)
```
# cd ~/TLS/k8s
# vi ca-config.json
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
# vi ca-csr.json
{
    "CN": "kubernetes",
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
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

# 1 部署master节点
## 1.1 部署kube-apiserver
### 1.1.1 生成kube-apiserver证书
```
# cd ~/TLS/k8s
# vi server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.0.160",
      "192.168.0.2",
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
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
# ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  server.csr  server-csr.json  server-key.pem  server.pem
```
### 1.1.2 配置kube-apiserver
```
# vi /opt/kubernetes/cfg/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=ip1:port,ip2:port \\
--bind-address=ip \\
--secure-port=6443 \\
--advertise-address=ip \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
# cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```
```
KUBE_APISERVER_OPTS
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
### 1.1.3 配置自动签发证书
```
# vi /opt/kubernetes/cfg/token.csv
token,用户名,UID,用户组
例子：4136692876ad4b01bb9dd0988480ebba,kubelet-bootstrap,10001,"system:node-bootstrapper"
```
### 1.1.4 systemd管理kube-apiserver
```
# vi /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-apiserver
```

## 1.2 部署kube-controller-manager
### 1.2.1 生成kube-controller-manager证书
```
# cd ~/TLS/k8s
# vi kube-controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
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
      "OU": "System"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
# ls kube-controller-manager
ca-kube-controller-manager.csr kube-controller-manager-key.pem kube-controller-manager-csr.json kube-controller-manager.pem
```
### 1.2.2 配置kube-controller-manager
```
# vi /opt/kubernetes/cfg/kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s"
```
```
KUBE_CONTROLLER_MANAGER_OPTS
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
### 1.2.3 生成kubeconfig
```
# KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
# KUBE_APISERVER="ip:port"
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
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
### 1.2.4 systemd管理kube-controller-manager
```
# vi usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service
 
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-controller-manager
```
## 1.3 部署kube-scheduler
### 1.3.1 生成kube-scheduler证书
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
# ls kube-scheduler*
kube-scheduler.csr  kube-scheduler-csr.json  kube-scheduler-key.pem  kube-scheduler.pem
```
### 1.3.2 配置kube-scheduler
```
vi /opt/kubernetes/cfg/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1"
```
### 1.3.3 生成kubeconfig
```
# KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
# KUBE_APISERVER="ip:port"
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
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
### 1.3.4 systemd管理kube-scheduler
```
# vi usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service
 
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl start kube-scheduler
```
## 1.4 kubectl
### 1.4.1 生成kubectl证书
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
      "OU": "System"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
```
### 1.4.2 通过kubectl查看当前集群组件状态
```
# kubectl get cs
```
### 1.4.3 生成kubeconfig
```
# mkdir /root/.kube
# KUBE_CONFIG="/root/.kube/config"
# KUBE_APISERVER="https://192.168.176.140:6443"
# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
# kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
# cat /root/.kube/config
```
### 1.4.4 授权kubelet-bootstrap用户允许请求证书
```
# kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

# 2 部署work节点
## 2.1 部署kubelet
### 2.1.1 配置kubelet
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
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
```

https://blog.csdn.net/qq_54494363/article/details/131833096
