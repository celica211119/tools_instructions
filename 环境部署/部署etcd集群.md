# 1 下载二进制文件
```
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
tar -zxvf etcd-v3.4.9-linux-amd64.tar.gz
```

# 2 部署etcd集群
## 2.1 添加hosts信息
```
vi /etc/hosts
```
添加以下内容
```
127.0.0.1 localhost
ip1 主机名1
ip2 主机名2
ip3 主机名3
ip4 主机名4
```
## 2.1 在master1节点部署
创建工作目录并复制二进制文件
```
mkdir /opt/etcd/{bin,cfg,ssl} -p
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```
创建etcd配置文件，位置为/opt/etcd/cfg/etcd.conf
```
[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/opt/etcd/data/default.etcd"
ETCD_INITIAL_CLUSTER="etcd-1=http://ip1:2380,etcd-2=http://ip2:2380,etcd-3=http://ip3:2380,etcd-4=http://ip4:2380"
ETCD_UNSUPPORTED_ARCH=amd6464
```
配置说明
```
ETCD_NAME：当前节点名称，集群中唯一。
ETCD_DATA_DIR：数据目录。
ETCD_INITIAL_CLUSTER：集群内所有节点地址。
ETCD_UNSUPPORTED_ARCH：集群架构。
```
## 2.2 注册etcd服务
创建etcd服务
```
vi /usr/lib/systemd/system/etcd.service
```
添加以下内容
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--listen-peer-urls http://本机ip:2380 \
--listen-client-urls http://本机ip:2379,http://127.0.0.1:2379 \
--initial-advertise-peer-urls http://本机ip:2380 \
--advertise-client-urls http://本机ip:2379 \
--initial-cluster-token etcd-cluster \
--initial-cluster-state new \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
重新加载服务
```
systemctl daemon-reload
```
## 2.3 在其他节点完成相同操作，并根据具体节点修改etcd.conf内的ETCD_NAME
## 2.4 在所有节点同时启动etcd服务并查看状态
```
systemctl start etcd
systemctl status etcd
```
查看日志观察状态排查错误
```
journalctl -u etcd
```
## 2.5 验证etcd集群成员与各节点状态
```
etcdctl member list -w table
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --endpoints="http://ip1:2379,http://ip2:2379,http://ip3:2379,http://ip4:2379" endpoint health --write-out=table
```

# 3 常见错误及解决方法
## 3.1 
错误信息
```
{"level":"info","ts":"2025-02-17T10:07:22.348Z","caller":"etcdmain/etcd.go:110","msg":"failed to detect default host","error":"could not find default route"}
```
错误原因
```
缺少必要的IP映射
```
解决方法
```
添加正确hosts信息
```
## 3.2 
错误信息
```
 {"level":"fatal","ts":"2025-02-18T01:43:53.773Z","caller":"etcdmain/etcd.go:271","msg":"discovery failed","error":"error setting up initial cluster: URL address does not have the form \"host:port\": http://10.8.94.210">
```
错误原因
```
etcd.conf信息错误
```
解决方法
```
etcd.conf对应节点信息添加对应端口号
```
