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
ntpdate -su 192.168.0.245 //
echo $?
# 同步成功显示为
0


#创建工作目录
mkdir -p ~/TLS/{etcd,k8s}
```
