# 1. 本地机器
## 1.1 添加用户并验证
```
useradd -m -s /usr/bin/bash 用户名
su - 用户名
```
## 1.2 生成ssh密钥并上传公钥到目标机器
```
ssh-keygen
cat /home/用户名/.ssh/id_rsa.pub | ssh root@10.8.94.208 'mkdir -p /home/用户名/.ssh  && cat >> /home/用户名/.ssh/authorized_keys'
```
## 1.3 设置目标机器别名
创建config文件
```
su - 用户名
mkdir ~/.ssh
vi ~/.ssh/config
```
添加以下内容
```  
Host 目标别名1
  HostName 目标IP
  User 用户名
  Port 端口号
  StrictHostKeyChecking no
  IdentityFile /home/用户名/.ssh/id_rsa

Host 目标别名2
  HostName 目标IP
  User 用户名
  Port 端口号
  ProxyJump 代理机器别名
  IdentityFile ~/.ssh/id_rsa
```

# 2. 目标机器
## 2.1 添加用户并验证
```
useradd -m -s /usr/bin/bash 用户名
su - 用户名
```
## 2.2 设置用户无密码root权限
```
visudo
```
添加以下内容
```
用户名 ALL=(ALL) NOPASSWD: ALL
```
## 2.3 添加公钥
```
vi /home/用户名/.ssh/authorized_keys
```
