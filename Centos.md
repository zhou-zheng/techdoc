# **CentOS 学习笔记**
## **更换 CentOS 的下载源为阿里云**
1、备份<br />
`mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`<br />
2、下载新的CentOS-Base.repo 到 /etc/yum.repos.d/<br />
&emsp;&emsp; · CentOS 5<br />
&emsp;&emsp; `wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-5.repo`<br />
&emsp;&emsp; · CentOS 6<br />
&emsp;&emsp; `wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo`<br />
&emsp;&emsp; · CentOS 7<br />
&emsp;&emsp; `wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo`<br />
3、之后运行 `yum makecache` 生成缓存<br />