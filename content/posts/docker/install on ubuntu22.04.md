---
title: "安装 Docker"
date: 2024-08-24T11:36:21+08:00
draft: false
categories: ["docker"]
tags: [Docker]
---
记录在 Ubuntu 上安装 Docker 的流程。
<!--more-->

## 1. 替换 apt 软件仓库

~~~shell
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

# 以下安全更新软件源包含了官方源与镜像站配置，如有需要可自行修改注释切换
deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# # deb-src https://mirrors4.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
~~~

## 2. 安装 docker

~~~shell
# 这里的 gpg 文件应该可以换成对应的清华镜像中的 gpg，不过这里直接用官方的也是可以的
#https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 这里使用了清华镜像的 Docker CE 软件仓库替换官方的 Docker CE 软件仓库
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 官方的 Docker CE 软件仓库  
#echo \
#  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
#  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
#  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
~~~

## 3. 添加用户到 Docker 用户组
~~~shell
sudo usermod -aG docker <username>

# 修改守护进程绑定的套接字的权限，能够被docker分组访问
sudo chmod a+rw /var/run/docker.sock
sudo systemctl restart docker
~~~


## 4. 代理配置

在国内拉取镜像经常失败，所以添加本地代理配置。添加后即可拉取成功。

~~~shell
sudo vim /etc/docker/daemon.json
~~~

~~~ json

{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7897",
    "https-proxy": "http://127.0.0.1:7897",
    "no-proxy": "*.test.example.com,.example.org,127.0.0.0/8"
  }
}
~~~

## 5. 参考

1. [清华大学开源软件镜像站 Ubuntu 软件仓库](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)
2. [清华大学开源软件镜像站 Docker CE 软件仓库](https://mirror.tuna.tsinghua.edu.cn/help/docker-ce/)
3. [将普通用户加入 Docker 用户组](https://murphypei.github.io/blog/2018/12/docker-add-group)
4. [配置守护程序以使用代理](https://docs.docker.com/engine/daemon/proxy/)