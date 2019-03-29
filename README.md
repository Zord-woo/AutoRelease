#自动发布流程

### 准备host
```
# setenforce 0

# vim /etc/selinux/config
SELINUX=disabled

### 安装必要软件
# yum install git epel-release vim gcc -y
# yum install python python-pip -y
# pip install ansible

### ssh-key
# ssh-keygen -t id_rsa
# cat .ssh/id_rsa.pub

### clone自动发布流程代码
# git clone https://github.com/Zord-woo/AutoRelease.git
```
### 准备globals.yml
```
# mkdir -p /etc/jarvis

# vim /etc/jarvis/globals.yml
---
# 用户指定的pip包
# custom_requirements:
#   - six

# 用户指定的rpm包
# custom_packages:
#   - vim

# 是否编译所有镜像, yes or no
build_all_images: "no"

# 用户指定编译的镜像列表, 该变量注释时，则编译OpenStack部署必要的镜像
custom_images:
  - openstack-base

# docker images tag
docker_tag: "1.0.8"

# docker images namespace
namespace: "nocsys"
```

### 执行代码
```
# cd AutoRelease

### 构建基础环境
# ./jarvis -c /etc/jarvis bootstarp

### 执行config
# ./jarvis -c /etc/jarvis config

### 执行build
# ./jarvis -c /etc/jarvis build

### 执行compress，打包kolla镜像和/opt/registry/docker中所有镜像
# ./jarvis -c /etc/jarvis compress

### 删除所有操作
# ./jarvis -c /etc/jarvis destroy
```
