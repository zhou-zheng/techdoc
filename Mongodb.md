# **Mongodb 学习文档**

## **Mongodb 安装**
实操环境使用的是 Vagrant Centos/7，安装 Mongodb 3.6 社区版的主要步骤如下：<br />
1）创建 /etc/yum.repos.d/mongodb-org-3.6.repo 文件以便于用 yum 安装 mongodb，文件内容如下
```
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```
2）用 yum 安装 mongodb<br />
` sudo yum install -y mongodb-org `<br />
如果要禁用 yum 自动升级 mongodb，可以在 /etc/yum.conf 中添加<br />
` exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools `<br />

## **Mongodb 运行**
1）运行 mongodb<br />
注意：实测 centos/7 下，SELinux 已经配置了 mongod_port_t，所以可以省略 1.1 步。<br />
` mongod_port_t                  tcp      27017-27019, 28017-28019 `<br />
1.1）在 Red Hat Enterprise Linux 或者 CentOS Linux 中，如果开启了 SELinux，必须配置 SELinux 开放对 Mongodb 启动的允许，有如下三种方式：<br />
- SELinux 处于 enforcing 模式<br />
` semanage port -a -t mongod_port_t -p tcp 27017 `</br>
如果你的 centos 是最小化安装，有可能需要先安装 semanage<br /> 
```
sudo yum -y install semanage
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * extras: mirrors.163.com
 * updates: mirrors.163.com
No package semanage available.
Error: Nothing to do

sudo yum provides semanage
...
policycoreutils-python-2.5-17.1.el7.x86_64 : SELinux policy core python utilities
Repo        : base
Matched from:
Filename    : /usr/sbin/semanage

此处被安装包的名字可用 Tab 补全
sudo yum -y install policycoreutils-python.x86_64
```
- 在 /etc/selinux/config 文件中关闭 SELinux<br />
` SELINUX=disabled `<br />
必须重启系统使配置生效
- 在 /etc/selinux/config 文件中设置 SELinux 为 permissive 模式<br />
` SELINUX=permissive `<br />
必须重启系统使配置生效

也可以使用 setenforce 来关闭或者设置 SELINUX 为 permissive，这样不需要重启，但是在重启后会失效。<br />
1.2）Mongodb 默认的数据目录为 /var/lib/mongo，默认的日志目录为 /var/log/mongodb。<br />
1.3）启动 Mongodb<br />
` sudo service mongod start `<br />
可以通过查看 /var/log/mongodb/mongod.log 来确认 mongod 已成功启动。<br />
1.4）Mongodb 自启动<br />
` sudo chkconfig mongod on `<br />
1.5）停止 Mongodb<br />
` sudo service mongod stop `<br />
1.6）重启 Mongodb<br />
` sudo service mongod restart `<br />

## **Mongodb 卸载**
1）停止 Mongodb<br />
` sudo service mongod stop `<br />
2）卸载包<br />
` sudo yum erase $(rpm -qa | grep mongodb-org) `<br />
3）删除相关目录<br />
` sudo rm -r /var/log/mongodb `<br />
` sudo rm -r /var/lib/mongo `<br />

