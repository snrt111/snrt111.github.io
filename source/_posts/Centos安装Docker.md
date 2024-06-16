---
title: Centos安装Docker
date: 2024-06-15 11:46:37
tags:
---

### 安装Docker

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce
```

### 启动Docker

```bash
sudo systemctl start docker
```

### 设置开机启动

```bash
sudo systemctl enable docker
```

### 查看当前docker状态

```bash
sudo systemctl status docker
```

### 验证Docker

```bash
sudo docker run hello-world
```

### 卸载Docker

```bash
sudo yum remove docker-ce
``` 


### 配置阿里云镜像加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
``` 



