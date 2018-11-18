![](img/vagrant.png)

Vagrant是一款用来构建虚拟开发环境的外挂工具，可以简化虚拟机配置和管理。它底层支持VirtualBox、VMware、AWS等，非常适合使用php/python/ruby/java语言开发web应用，“代码在我机子上运行没有问题”这种说辞将成为历史。

## 1. 安装VirtualBox
去官网[https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)下载最新版的Virtualbox，然后双击安装，一直点击确认完成。 
## 2. 安装Vagrant
去官网 [https://www.vagrantup.com/downloads.html](https://www.vagrantup.com/downloads.html)下载最新版的Vagrant，然后双击安装，一直点击确认完成。在windows下安装vagrant，为了写入相应配置到环境变量，系统可能会要求重新启动。在命令行中，输入vagrant，查看程序是不是已经运行了。如果不行，请检查一下$PATH里面是否包含vagrant所在的路径 
## 3. Vagrant命令详解
命令|	作用
---|---
vagrant box add	|添加box的操作
vagrant init	|初始化box的操作，会生成vagrant的配置文件Vagrantfile
vagrant up	|启动本地环境
vagrant ssh	|通过ssh登录本地环境所在虚拟机
vagrant halt	|关闭本地环境
vagrant suspend	|暂停本地环境
vagrant resume	|恢复本地环境
vagrant reload	|修改了Vagrantfile后，使之生效（相当于先 halt，再 up）
vagrant destroy	|彻底移除本地环境
vagrant box list	|显示当前已经添加的box列表
vagrant box remove	|删除相应的box
vagrant package	|打包命令，可以把当前的运行的虚拟机环境进行打包
vagrant plugin	|用于安装卸载插件
vagrant status	|获取当前虚拟机的状态
vagrant global-status	|显示当前用户Vagrant的所有环境状态

## 4. 安装配置虚拟机
接下来，我们需要选择使用何种操作系统，这里以centos7.2为例。以前基于虚拟机的工作流，我们需要下载ISO镜像，安装系统，设置系统等操作。而Vagrant开源社区提供了许多已经打包好的操作系统，我们称之为box。你可以从box下载地址（下文列出），找到你想要的box，当然你也可以自己制作一个。

官方仓库：[https://atlas.hashicorp.com/boxes/search](https://atlas.hashicorp.com/boxes/search)

官方镜像：[https://vagrantcloud.com/boxes/search](https://vagrantcloud.com/boxes/search)

第三方仓库：[http://www.vagrantbox.es/](http://www.vagrantbox.es/) 

Vagrant提供在线安装服务，非常方便，但是在国内加载会很慢，建议先把box下载到本地，再导入安装，box链接：[https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box](https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box)

导入安装方法：

```bash
vagrant box add {title} {url}
vagrant init {title}
vagrant up
```
`vagrant box add`是添加box的命令，｛title｝是以后创建虚拟机的别名，｛url｝是下载到本地box的路径，也可以是服务器端的URL
> 假设你已经将centos7的box下载完成

**4.1** 安装box

```bash
# vagrant box add centos7 ~/Download/centos-7.0-x86_64.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'centos7' (v0) for provider: 
    box: Unpacking necessary files from: file:///Users/teishi/Downloads/centos-7.0-x86_64.box
==> box: Successfully added box 'centos7' (v0) for 'virtualbox'!

```
**4.2** 指定工作目录 & 初始化 & 启动虚拟机

```bash
# mkdir myvagrant
# cd myvagrant
# vagrant init centos7
# vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'centos7'...
==> default: Matching MAC address for NAT networking...
==> default: Setting the name of the VM: myvagrant_default_1542518492008_40687
==> default: Clearing any previously set forwarded ports...
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222...
...
==> default: Mounting shared folders...
    default: /vagrant => /Users/teishi/vagrant/myvagrant
```
**4.3** SSH连接虚拟机

在工作目录下执行如下指令可以直接进入虚拟机

```
vagrant ssh
```
默认虚拟机ssh端口会映射到本机的端口2200-2222(具体看前面的初始化提示), 用户名与密码默认都是vagrant，常用的是通过第三方工具例如secureCRT来连接

```
ssh -p 2200 vagrant@127.0.0.1
```
