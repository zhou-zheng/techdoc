# **Mongodb 学习文档**

## **Mongodb 安装**
以下内容摘自 [Install MongoDB Community Edition on Red Hat Enterprise or CentOS Linux](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/)<br />
### **MongoDB 社区版/企业版安装**
MongoDB 分为社区版和企业版<br />
实操环境使用的是 Vagrant Centos/7，安装 Mongodb 3.6 社区版/企业版的主要步骤如下：<br />
1）社区版：以 root 用户身份创建 /etc/yum.repos.d/mongodb-org-3.6.repo 文件以便于用 yum 安装 mongodb，文件内容如下<br />
```
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```
企业版：以 root 用户身份创建 /etc/yum.repos.d/mongodb-org-3.6.repo 文件以便于用 yum 安装 mongodb，文件内容如下<br />
```
[mongodb-enterprise]
name=MongoDB Enterprise Repository
baseurl=https://repo.mongodb.com/yum/redhat/$releasever/mongodb-enterprise/3.6/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```
2）用 yum 安装 mongodb<br />
社区版：` sudo yum install -y mongodb-org `<br />
如果想安装特定发行版（如 3.6.2 中的 .2）的 MongoDB，可以对每个组件包指定版本号<br />
` sudo yum install -y mongodb-org-3.6.2 mongodb-org-server-3.6.2 mongodb-org-shell-3.6.2 mongodb-org-mongos-3.6.2 mongodb-org-tools-3.6.2 `<br />
如果要禁用 yum 自动升级 mongodb，可以在 /etc/yum.conf 中添加<br />
` exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools `<br />
企业版：` sudo yum install -y mongodb-enterprise `<br />
如果想安装特定发行版（如 3.6.2 中的 .2）的 MongoDB，可以对每个组件包指定版本号<br />
` sudo yum install -y mongodb-enterprise-3.6.2 mongodb-enterprise-server-3.6.2 mongodb-enterprise-shell-3.6.2 mongodb-enterprise-mongos-3.6.2 mongodb-enterprise-tools-3.6.2 `<br />
如果要禁用 yum 自动升级 mongodb，可以在 /etc/yum.conf 中添加<br />
` exclude=mongodb-enterprise,mongodb-enterprise-server,mongodb-enterprise-shell,mongodb-enterprise-mongos,mongodb-enterprise-tools `<br />

### **Mongodb 运行**
1）运行 mongodb<br />
注意：实测 centos/7.4 下，SELinux 已经配置了 mongod_port_t，所以可以省略 1.1 步。<br />
` mongod_port_t                  tcp      27017-27019, 28017-28019 `<br />
1.1）在 Red Hat Enterprise Linux 或者 CentOS Linux 中，如果开启了 SELinux，必须配置 SELinux 开放对 Mongodb 启动的允许，有如下三种方式：<br />
- SELinux 处于 enforcing 模式<br />
` semanage port -a -t mongod_port_t -p tcp 27017 `</br>
CentOS7.4下，已经内置对 mongodb 的支持：<br />
```
[vagrant@localhost yum.repos.d]$ sudo semanage port -l | grep 27017
mongod_port_t                  tcp      27017-27019, 28017-28019
```
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
必须重启系统使配置生效<br />
- 在 /etc/selinux/config 文件中设置 SELinux 为 permissive 模式<br />
` SELINUX=permissive `<br />
必须重启系统使配置生效<br />
也可以使用 setenforce 来切换到 permissive 模式，这样不需要重启，但是在重启后会失效。<br />

1.2）Mongodb 实例默认的数据目录为 /var/lib/mongo，默认的日志目录为 /var/log/mongodb，而且默认以 mongod 用户运行。可以在 /etc/mongod.conf 中指定不同的数据（storage.dbPath）和日志（systemLog.path）目录。<br />
如果修改了运行 MongoDB 的用户，那**必须**对应修改该用户对 /var/lib/mongo 和 /var/log/mongodb 目录的访问控制权限。

1.3）启动 Mongodb<br />
` sudo service mongod start `<br />
或<br />
` sudo systemctl start mongod.service `<br />
可以通过查看 /var/log/mongodb/mongod.log 来确认 mongod 已成功启动。<br />

1.4）Mongodb 自启动<br />
` sudo chkconfig mongod on `<br />
或<br />
` sudo systemctl enable mongod.service `<br />

1.5）停止 Mongodb<br />
` sudo service mongod stop `<br />
或<br />
` sudo systemctl stop mongod.service `<br />

1.6）重启 Mongodb<br />
` sudo service mongod restart `<br />
或<br />
` sudo systemctl restart mongod.service `<br />

1.7）开始使用 MongoDB<br />
在运行 mongod 的本机上打开一个 mongo shell<br />
` mongo --host 127.0.0.1:27017 `<br />

### **Mongodb 卸载**
1）停止 Mongodb<br />
` sudo service mongod stop `<br />
或<br />
` sudo systemctl stop mongod.service `<br />

2）卸载包<br />
社区版：` sudo yum erase $(rpm -qa | grep mongodb-org) `<br />
企业版：` sudo yum erase $(rpm -qa | grep mongodb-enterprise) `<br />

3）删除数据和日志目录<br />
` sudo rm -r /var/log/mongodb `<br />
` sudo rm -r /var/lib/mongo `<br />

## **Mongodb 设置**
1）UNIX ulimit 推荐设置
- -f (file size): unlimited
- -t (cpu time): unlimited
- -v (virtual memory): unlimited
- -n (open files): 64000
- -m (memory size): unlimited
- -u (processes/threads): 64000

可以通过 cat /proc/$(pidof mongod)/limits 来查看

## **Mongodb 入门**
### **Schema Validation**
以下内容摘自[Schema Validation](https://docs.mongodb.com/manual/core/schema-validation/)<br />
New in version 3.2.<br />
MongoDB provides the capability to perform schema validation during updates and insertions.<br />
#### **Specify Validation Rules**
Validation rules are on a per-collection basis.

To specify validation rules when creating a new collection, use `db.createCollection()` with the validator option.

To add document validation to an existing collection, use `collMod` command with the validator option.

MongoDB also provides the following related options:

- validationLevel option, which determines how strictly MongoDB applies validation rules to existing documents during an update, and
- validationAction option, which determines whether MongoDB should error and reject documents that violate the validation rules or warn about the violations in the log but allow invalid documents.

## **Mongodb 待整理**
db.createCollection()
db.getCollectionInfos()
