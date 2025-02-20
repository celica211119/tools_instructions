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
