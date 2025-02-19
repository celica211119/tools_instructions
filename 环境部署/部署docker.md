# 1 下载二进制文件
```
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
tar -zxvf docker-19.03.9.tgz
```

# 2 部署docker
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
