# **Vagrant 学习文档**
Vagrant 非常适合用于构造开发、运维或设计环境，详情请访问 [官方网站](https://www.vagrantup.com/)。

## **Vagrant 安装**
进入官方网站，根据操作系统下载安装文件，然后进行安装即可。
### **Vagrant 提供者**
Vagrant 依赖于后端虚拟机提供者，现已支持 VirtualBox、VMware、AWS等；</br>
每当安装完毕一种虚拟机提供者，只要带上对应参数运行 vagrant up 就可以启动相应虚拟机下的环境：
```
$ vagrant up --provider=vmware_fusion
$ vagrant up --provider=aws
```

## **Vagrant 环境**
### **Vagrant 环境操作**
可以从 [HashiCorp's Vagrant Cloud box catalog](https://app.vagrantup.com/boxes/search) 获取各种 vagrant boxes。

| 命令                                                 | 描述                                         | 实例                                                                                                              |
| ---------------------------------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| vagrant -h                                           | 查看有关 vagrant 的帮助                      | vagrant -h                                                                                                        |
| vagrant version                                      | 查看 vagrant 版本                            | vagrant version                                                                                                   |
| vagrant init  [options] [name [url]]                 | 在当前目录初始化 Vagrantfile 文件            | vagrant init                                                                                                      |
| vagrant validate                                     | 验证当前目录下的 Vagrantfile 文件            | vagrant validate                                                                                                  |
| vagrant box -h                                       | 查看有关 vagrant box 帮助                    | vagrant box -h 以及 vagrant box add -h 等                                                                         |
| vagrant box add [options] \<name, url, or path\>     | 安装 vagrant box                             | vagrant box add hashicorp/precise64 或 vagrant box add centos/7 c:\CentOS-7-x86_64-Vagrant-1710_01.VirtualBox.box |
| vagrant box remove \<name\>                          | 移除 vagrant box                             | vagrant box remove hashicorp/precise64                                                                            |
| vagrant box list [options]                           | 查看 vagrant box 已安装列表                  | vagrant box list                                                                                                  |
| vagrant up [options] [name\|id]                      | 启动 vagrant 环境                            | vagrant up hashicorp/precise64                                                                                    |
| vagrant suspend [name\|id]                           | 挂起 vagrant 环境                            | vagrant suspend hashicorp/precise64                                                                               |
| vagrant resume [vm-name]                             | 恢复 vagrant 环境                            | vagrant resume hashicorp/precise64                                                                                |
| vagrant reload [vm-name]                             | 重新加载 Vagrantfile 文件并重启 vagrant 环境 | vagrant reload hashicorp/precise64                                                                                |
| vagrant halt [options] [name\|id]                    | 停止 vagrant 环境                            | vagrant halt hashicorp/precise64                                                                                  |
| vagrant destroy [options] [name\|id]                 | 停止并删除 vagrant 环境                      | vagrant destroy hashicorp/precise64                                                                               |
| vagrant ssh [options] [name\|id] [-- extra ssh args] | 通过 ssh 连接到 vagrant 环境                 | vagrant ssh hashicorp/precise64                                                                                   |
| vagrant status [name\|id]                            | 查看 vagrant 环境的状态                      | vagrant status hashicorp/precise64                                                                                |

Vagrant 通过 Vagrantfile 来指定从哪个 box 构建环境（当该box不存在时会尝试从网络下载），然后就可以启动、挂起、恢复、重启、停止乃至删除环境，当然也可以通过 ssh 连接到环境。

### **Vagrant 环境配置**

#### **同步文件夹**
Vagrant 会缺省地将当前环境目录同步到虚拟机的 /vagrant 目录；</br>
可以通过修改 Vagrantfile 文件中 config.vm.synced_folder 选项来进行目录同步的添加、删除和修改。<br />

ALEX ZHOU 注：<br />
在 VirtualBox 中使用 Vagrant Cloud 的 centos boxes 时遇到有关同步文件夹的坑，如果想成功使用同步文件夹<br />
a）使用 centos/7 最小化安装的 base box，同步文件夹例如 /vagrant 目录将无法自动同步，而且虚拟机中创建的文件会自动丢失，但是 Ubuntu base box 不会有此问题。<br />
&emsp;&emsp; · 宿主机确保安装了 vbguest 插件<br />
&emsp;&emsp; `vagrant plugin install vagrant-vbguest`<br />
&emsp;&emsp; · 启动虚拟机，期间会自动安装 VirtualBox Guest Additions<br />
&emsp;&emsp; `vagrant up`<br />
&emsp;&emsp; · 关闭虚拟机，再重开虚拟机，观察启动过程中是否已经启动 VirtualBox Guest Additions<br />
&emsp;&emsp; `vagrant halt`<br />
&emsp;&emsp; `vagrant up`<br />
&emsp;&emsp; · 确认无误后再次关闭虚拟机<br />
&emsp;&emsp; `vagrant halt`<br />
&emsp;&emsp; · 重新打包成 package.box，当然可以重命名<br />
&emsp;&emsp; `vagrant package`<br />
&emsp;&emsp; · 用新生成的 box 文件再次创建虚拟机，这个虚拟机启动后，同步文件夹就不会有问题了。<br />
b）使用 bento/centos-7.2 的 base box<br />
无需安装 VirtualBox Guest Addition，即可创建虚拟机并使用同步文件夹功能。<br />

#### **自动设置**
Vagrant 内置支持自动设置，也就是在 vagrant up 时虚拟机会自动按配置安装软件。</br>下面以在 hashicorp/precise64 环境安装 Apache 为例：</br>
1）在放置 Vagrantfile 文件的目录下创建 bootstrap.sh 脚本
``` sh
#!/usr/bin/env bash

apt-get update
apt-get install -y apache2
if ! [ -L /var/www ]; then
  rm -rf /var/www
  ln -fs /vagrant /var/www
fi
```
2）修改 Vagrantfile 文件让 Vagrant 在创建环境时自动运行 bootstrap.sh 脚本
``` sh
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
end
```
3）运行 vagrant up 启动环境，Vagrant 将自动设置安装 Apache</br>
4）注意此方法适用于简单的安装脚本，如果要进行比较复杂的自动设置，应该使用自定义 box 的方法

#### **网络**
##### 端口转发
修改 Vagrantfile 文件来进行从宿主机到虚拟机的端口转发，以 访问虚拟机中的 Apache 为例：
``` sh
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
  config.vm.provision :shell, path: "bootstrap.sh"
  config.vm.network :forwarded_port, guest: 80, host: 4567
end
```
修改完毕后可以用 vagrant reload 来重新加载 Vagrantfile 文件并重启。

##### 网络配置
Vagrant 也支持通过静态IP地址、桥接等方式连接宿主机和虚拟机

## **Vagrant 共享**
Vagrant 共享让你可以把你创建的环境通过互联网共享给全世界，它将提供一个可以从任何连上因特网的设备都能访问到你的环境的 URL;</br>
当前版本的 Vagrant 使用 [ngrok](https://ngrok.com/) 来进行共享，使用共享之前请先确认 ngrok 已安装。
``` 
$ vagrant share
Vagrant Share now defaults to using the `ngrok` driver.
The `classic` driver has been deprecated.

For more information about the `ngrok` driver, please
refer to the documentation:

  https://www.vagrantup.com/docs/share/
==> default: Detecting network information for machine...
    default: Local machine address: 127.0.0.1
    default:
    default: Note: With the local address (127.0.0.1), Vagrant Share can only
    default: share any ports you have forwarded. Assign an IP or address to your
    default: machine to expose all TCP ports. Consult the documentation
    default: for your provider ('virtualbox') for more information.
    default:
    default: Local HTTP port: 4567
    default: Local HTTPS port: disabled
    default: Port: 2200
    default: Port: 4567
==> default: Creating Vagrant Share session...
==> default: HTTP URL: http://3283c7f1.ngrok.io
==> default:
```

## **Vagrant 实用技巧**
### **Vagrant VirtualBox Guest Additions 自动下载安装升级**
以下内容摘自 [Automatically Download and Install VirtualBox Guest Additions in Vagrant](https://dzone.com/articles/automatically-download-and)<br />
So, are you already using  Vagrant to manage your VirtualBox VMs?<br />
Then you probably have realized already how annoying is to keep the VBox guest additions up to date in your VMs.<br />
Don’t worry, you can update them with just one command or automatically on each start using the Vagrant-vbguest plugin.<br />
- > 安装<br />
`vagrant plugin install vagrant-vbguest`
- > 使用<br />
By default the plugin will check what version of the guest additions is installed in the VM every time it is started with vagrant start. Note that it won’t be checked when resuming a box.<br />
In any case, it can be disabled in the Vagrantfile<br />
```
Vagrant::Config.run do |config|
  # set auto_update to false, if do NOT want to check the correct additions
  # version when booting this machine
  config.vbguest.auto_update = false
end
```
If it detects an outdated version, it will automatically install the matching version from the VirtualBox installation, located at<br />
&emsp;&emsp; · linux : /usr/share/virtualbox/VBoxGuestAdditions.iso<br />
&emsp;&emsp; · Mac : /Applications/VirtualBox.app/Contents/MacOS/VBoxGuestAdditions.iso<br />
&emsp;&emsp; · Windows : %PROGRAMFILES%/Oracle/VirtualBox/VBoxGuestAdditions.iso<br />
The location can be overridden with the iso_path parameter in your Vagrantfile, and can point to a http server<br />
```
Vagrant::Config.run do |config|
  config.vbguest.iso_path = "#{ENV['HOME']}/Downloads/VBoxGuestAdditions.iso"
  # or
  config.vbguest.iso_path = "http://company.server/VirtualBox/$VBOX_VERSION/VBoxGuestAdditions.iso"
end
```
If you have disabled the automatic update, it still easy to manually update the VirtualBox Guest Additions version, just running from the command line<br />
`vagrant vbguest`

### **Vagrant 快照备份**
使用 Vagrant 的快照功能可以很方便快速地创建当前虚拟机的一个临时备份状态，在进行重要操作时可以先创建一个快照以便在操作失误后快速恢复。

安装 Vagrant 快照插件：<br />
` vagrant plugin install vagrant-vbox-snapshot `

运行 Vagrant 快照插件：
```
$ vagrant snapshot
Usage: vagrant snapshot <command> [<args>]

Available subcommands:
     back
     delete
     go
     list
     take

For help on any individual command run `vagrant snapshot <command> -h
```
- 创建一个快照<br />
` vagrant snapshot take [vm-name] <SNAPSHOT_NAME> `
- 查看快照列表<br />
` vagrant snapshot list `
- 恢复到上一个最近的快照<br />
` vagrant snapshot back [vm-name] `
- 从指定快照恢复<br />
` vagrant snapshot go [vm-name] <SNAPSHOT_NAME> `
- 删除一个快照<br />
` vagrant snapshot delete [vm-name] <SNAPSHOT_NAME> `
