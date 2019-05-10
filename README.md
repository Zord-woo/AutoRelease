#### 准备host
```
# setenforce 0

# vim /etc/selinux/config
SELINUX=disabled

### 安装必要软件
# yum install git epel-release vim gcc -y
# yum install python python-pip -y
# pip install ansible

### clone自动发布流程代码
# cd /root/
# git clone https://github.com/Zord-woo/AutoRelease.git
```
#### 配置globals.yml
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

# 用户指定编译的镜像列表, 该变量注释时，则编译OpenStack部署必要的镜像
# 若需要编译所有镜像，只定义变量，内容为空
custom_images:
  - openstack-base

# docker images tag
docker_tag: "1.0.8"

# docker images namespace
namespace: "nocsys"

# github username,用于clone项目
githubuser: "Zord-woo"
```

#### 执行代码
```
# cd AutoRelease

### 查看使用帮助
# ./jarvis -h

### 构建基础环境
# ./jarvis -c /etc/jarvis bootstrap

### 执行config，此处会提示Enter your github password, 与上面githubuser对应的密码
# ./jarvis -c /etc/jarvis config

### 执行build,创建可执行kolla-ansible命令的镜像和kolla项目镜像
# ./jarvis -c /etc/jarvis build

### 执行compress，打包kolla镜像和/opt/registry/docker中所有镜像
# ./jarvis -c /etc/jarvis compress
# ls -l /root/
kolla.tar 
centos-binary-registry-ocata-1.0.8.tar.gz

### 删除所有操作
# ./jarvis -c /etc/jarvis destroy  --yes-i-really-really-mean-it
```

## 部署OpenStack
#### 准备环境

节点 | 操作系统 | 硬件配置 | ip
---|---|---|---
anode | centos 7 | CPU:2 Mem:4G Disk:40G | ens3: 92.0.0.10(内), ens6:192.168.200.10(外)
bnode | centos 7 | CPU:2 Mem:8G Disk:40G | ens3: 92.0.0.11(内), ens6:192.168.200.11(外)
cnode | centos 7 | CPU:2 Mem:2G Disk:40G | ens3: 92.0.0.12(内), ens6:192.168.200.12(外)

#### anode
安装docker
```
# tee /etc/yum.repos.d/docker.repo << 'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

# yum install docker-engine-1.12.5  vim -y
# systemctl start docker 
# systemctl enable docker
```
#### 加载kolla-ansible和kolla镜像
```
# docker load -i kolla.tar

# docker run -d -v /opt/registry:/var/lib/registry -p 4000:5000 --restart=always --name registry registry:2

# tar zxf centos-binary-registry-ocata-1.0.8.tar.gz -C /opt/registry/
```
#### 启动可执行kolla-ansible命令container，并修改配置文件
```
### 注意事项，如果kolla container与AutoRelease项目处于同一host，需要将host的Nginx停用，否则80端口占用，容器一直退出
# docker run -d --net=host --name=kolla kolla:v1

### 以下命令进入kolla容器执行命令过程不显示特殊颜色，纯白色
# docker exec -it kolla bash

### 以下命令进入kolla容器执行命令过程显示特殊色，错误显示红色
# docker exec -it kolla env TERM=xterm bash -l

#### 修改globals.yml配置文件，基本配置
# vim /etc/kolla/globals.yml
kolla_install_type: "binary"
openstack_release: "1.0.8"

kolla_internal_vip_address: "92.0.0.13"
kolla_additional_vip_address: "92.0.0.14"

docker_registry: "92.0.0.10:4000"
docker_namespace: "nocsys"

network_interface: "ens3"
neutron_external_interface: "ens6"

keepalived_virtual_router_id: "13"
keepalived_additional_virtual_router_id: "14"

openstack_logging_debug: "True"
pkg_offline_install: true
pkg_offline_install_baseurl: "http://92.0.0.10/RPMS"


# vim /etc/kolla/password.yml
keystone_admin_password: admin

### anode bnode cnode 和kolla container执行免密登录
# ssh-keygen -t rsa
# cat .ssh/id_rsa.pub >> .ssh/authorized_keys
# chmod 600 .ssh/authorized_keys

### 修改multinode
# vim multinode
[control]
92.0.0.11

[network]
92.0.0.11

[compute]
92.0.0.12

[monitoring]
92.0.0.11

[storage]
92.0.0.11
92.0.0.12

[gateway]
92.0.0.11

[apollo]
92.0.0.11
92.0.0.12

[extend]
92.0.0.10

```
### 执行部署OpenStack
```
### 安装基础运行环境，添加 -e “baremetal_reboot=false” 执行，节点不重启
# kolla-ansible -i multinode bootstrap-servers

### 安装前检查
# kolla-ansible -i multinode prechecks

### 将需要的images pull到对应的节点
# kolla-ansible -i multinode pull

### 安装openstack
# kolla-ansible -i multinode deploy
```
