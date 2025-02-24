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

# 2 部署k8s-worker
## 2.1 在worker3节点部署
创建工作目录并复制二进制文件
```
mkdir /opt/kubernetes/{bin,cfg,ssl} -p
cp kubelet kube-proxy /opt/kubernetes/bin
cp kubectl /usr/bin/
```
## 2.2 部署kubelet
### 2.2.1 配置kubelet
创建kubelet.conf
```
vi /opt/kubernetes/cfg/kubelet.conf
```
添加以下内容
```
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "master节点ip",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "cgroupfs",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.255.0.2"]
}
```
### 2.2.2 生成kubelet初次加入集群引导kubeconfig文件
```
KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
KUBE_APISERVER="apiserver地址"
TOKEN=与apiserver一致
```
和master节点内容保持一致
```
vi /opt/kubernetes/ssl/ca.pem
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
kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}
```
设置上下文参数
```
kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}
```
设置默认上下文
```
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```
### 2.2.3 生成kubeconfig文件
```
KUBE_CONFIG="/opt/kubernetes/cfg/kubelet.kubeconfig"
KUBE_APISERVER="apiserver地址"
TOKEN=与apiserver一致
KUBE_APISERVER="https://10.8.94.208:6443" # apiserver IP:PORT
```
设置集群参数
```
kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem --embed-certs=true --server=${KUBE_APISERVER} --kubeconfig=${KUBE_CONFIG}
```
设置客户端认证参数
```
kubectl config set-credentials kubelet-bootstrap --token=${TOKEN} --kubeconfig=${KUBE_CONFIG}
```
设置上下文参数
```
kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=${KUBE_CONFIG}
```
设置默认上下文
```
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```
创建角色绑定
```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
