FROM centos:latest

ADD nginx.repo /etc/yum.repos.d

RUN yum -y update

RUN yum install make  -y

RUN yum install -y epel-release git gcc python-devel libevent-devel

RUN yum install -y python  python-pip nginx createrepo

RUN pip install ansible==2.3

RUN pip install --upgrade pip

ADD kolla-ansible /root/kolla-ansible

ADD kolla_packages /usr/share/nginx/html/RPMS

RUN createrepo /usr/share/nginx/html/RPMS

WORKDIR /root/kolla-ansible

RUN cp -r etc/kolla /etc

RUN cp ansible/inventory/* ~/

RUN pip install .

WORKDIR /root

RUN rm -rf kolla-ansible

RUN echo "daemon off;" >> /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx"]
