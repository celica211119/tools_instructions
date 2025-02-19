# 1 下载二进制文件
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
```

# 2 安装cfssl
## 2.1 复制二进制文件
```
cp cfssl_linux-amd64 /usr/local/bin/cfssl
cp cfssljson_linux-amd64 /usr/local/bin/cfssljson
cp cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```
