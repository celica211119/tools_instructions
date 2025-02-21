# 1 下载二进制文件
```
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1.amd64.tgz
tar -zxvf docker-19.03.9.tgz
tar -zxvf cri-dockerd-0.3.1.amd64.tgz
```

# 2 安装docker
## 2.1 复制二进制文件
```
mv docker/* /usr/bin/
```
## 2.2 配置镜像加速
新建目录及配置文件
```
mkdir -p /etc/docker
vi /etc/docker/daemon.json
```
添加以下内容
```
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
```
## 2.3 注册docker服务
创建docker服务
```
vi /usr/lib/systemd/system/docker.service
```
添加以下内容
```
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
```
重新加载服务
```
systemctl daemon-reload
```
## 2.4 启动docker服务并查看状态
```
systemctl start docker
systemctl status docker
```

# 3 安装cri-dockerd
## 3.1 复制二进制文件
```
install -o root -g root -m 0755 ./cri-dockerd /usr/local/bin/cri-dockerd
```
## 3.2 注册cri-dockerd服务
创建cri-dockerd服务
```
vi /usr/lib/systemd/system/cri-docker.service
```
添加以下内容
```
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
```
创建依赖服务
```
vi /usr/lib/systemd/system/cri-docker.socket
```
添加以下内容
```
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service
[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
[Install]
WantedBy=sockets.target
```
重新加载服务
```
systemctl daemon-reload
```
## 3.3 启动cri-dockerd服务并查看状态
```
systemctl start cri-dockerd
systemctl is-active cri-docker.socket
```


# 4 常见错误及解决方法
## 4.1 
错误信息
```
cri-docker.socket: Failed to resolve group docker: No such process
```
错误原因
```
用户组不存在
```
解决方法
```
groupadd docker
usermod -aG docker $(whoami)
exec sudo su -l $USER
```
