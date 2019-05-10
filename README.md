#### AutoRelease中clone的项目、对应分支、目录
Repositories | branch | 对应clone目录 
---|---|---
https://github.com/TeamNocsys/alert | master | /root/kolla/docker/cloudalert/
https://github.com/TeamNocsys/kolla | nocsys/ocata | /root/
https://github.com/TeamNocsys/kolla-ansible | nocsys/ocata | /root/
https://github.com/TeamNocsys/kolla_packages | centos/327 | /root/
https://github.com/TeamNocsys/gather-agent | master | /root
https://github.com/TeamNocsys/networking-ovn | nocsys/ocata | /root/

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
# git clone https://github.com/TeamNocsys/kolla-auto-release.git

### 以下两个文件由前台提供，放置在root目录下
### 1.cloudview代码包(名称为"public.tar.gz"或"public.tar")
### 2.middleware可执行文件(名称为"pkg_middleware")

### 如果OVS代码有更新请参考补充说明
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

# kolla项目分支
kolla_branch: "nocsys/ocata"

# kolla-ansible分支
kolla_ansible_branch: "nocsys/ocata"

# alert 分支
alert_branch: "master"

# gather-agent 分支
gather_agent_branch: "master"

# kolla_packages分支
kolla_packages_branch: "centos/327"

# networking-ovn分支
networking_ovn_branch: "nocsys/ocata"
```

#### 执行代码
```
# cd kolla-auto-release

### 查看使用帮助
# ./kolla-auto-release -h

### 构建基础环境
# ./kolla-auto-release -c /etc/jarvis bootstrap

### 执行config，此处会提示Enter your github password, 与上面githubuser对应的密码
# ./kolla-auto-release -c /etc/jarvis config

### 执行build,创建可执行kolla-ansible命令的镜像和kolla项目镜像
# ./kolla-auto-release -c /etc/jarvis build

### 执行compress，打包kolla镜像和/opt/registry/docker中所有镜像
# ./kolla-auto-release -c /etc/jarvis compress
# ls -l /root/
kolla.tar 
centos-binary-registry-ocata-1.0.8.tar.gz

### 删除所有操作
# ./kolla-auto-release -c /etc/jarvis destroy  --yes-i-really-really-mean-it
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

## 补充说明
#### 手动执行ovs代码更新
```
# git clone https://github.com/TeamNocsys/ovs.git
# yum install -y epel-release
# yum install -y autoconf automake libtool gcc clang openssl-devel libcap-ng-devel rpm-build yum-utils python-pip libpcap-devel numactl-devel
# pip install six
### 编译ovs
# cd ovs
# ./boot.sh
# ./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc --with-linux=/lib/modules/$(uname -r)/build
# make rpm-fedora RPMBUILD_OPT="--without check"
### 如果缺失依赖则只需
# yum-builddep rhel/openvswitch-fedora.spec
### 编译ovs内核模块
# make rpm-fedora-kmod

### 更新kolla_packages项目
# git clone https://github.com/TeamNocsys/kolla_packages.git
### 创建新分支，并将上面编译出的openvswitch-*.rpm软件包拷贝到kolla_packages目录并提交
# cd kolla_packages
# git branch centos/xxx

### 更新离线YUM源
# cd /root/kolla
### 备份离线源
# tar acf 2019-5-xx.tar.gz RPMS
### 删除旧的ovs软件，并将上面编译出的openvswitch-*.rpm软件包拷贝到RPMS目录，然后重启YUM源
# rm -rf RPMS/openvswitch-*.rpm
# ./start_myrepo
```

