---
title: Vagrant踩坑
date: 2020-01-12 00:00:00
categories:
 - 开发环境
---

Vagrant是用来管理虚拟机的，如VirtualBox、VMware、AWS等，主要好处是可以提供一个可配置、可移植和复用的软件环境，可以使用shell、chef、puppet等工具部署。

## 安装vagrant/virtualbox

> 由于`apt install`安装的版本太老, 所以从官网下载`vagrant_2.2.5_x86_64.deb`

``` bash
king@king:~/data/vagrantbox$ vagrant up --provider=virtualbox
The provider 'virtualbox' that was requested to back the machine
'default' is reporting that it isn't usable on this system. The
reason is shown below:

Vagrant has detected that you have a version of VirtualBox installed
that is not supported by this version of Vagrant. Please install one of
the supported versions listed below to use Vagrant:

4.0, 4.1, 4.2, 4.3, 5.0, 5.1

A Vagrant update may also be available that adds support for the version
you specified. Please check www.vagrantup.com/downloads.html to download
the latest version.

```

## 添加box
``` bash
$ vagrant box add ubuntu/bionic https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cloud-images/bionic/current/bionic-server-cloudimg-amd64-vagrant.box
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'ubuntu/bionic' (v0) for provider: 
    box: Downloading: https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cloud-images/bionic/current/bionic-server-cloudimg-amd64-vagrant.box
==> box: Successfully added box 'ubuntu/bionic' (v0) for 'virtualbox'!
```

## 更新
``` bash
$ vagrant box update
```

## 启动
``` bash
$ mkdir vagrantbox
$ cd vagrantbox/
$ vagrant init ubuntu/bionic
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
$ ls
Vagrantfile

```
``` bash
$ vagrant up --provider=virtualbox
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu/bionic'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/bionic' version '20190729' is up to date...
==> default: Setting the name of the VM: vagrantbox_default_1564392888222_56525
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: 
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default: 
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
    default: The guest additions on this VM do not match the installed version of
    default: VirtualBox! In most cases this is fine, but in rare cases it can
    default: prevent things such as shared folders from working properly. If you see
    default: shared folder errors, please make sure the guest additions within the
    default: virtual machine match the version of VirtualBox you have installed on
    default: your host and reload your VM.
    default: 
    default: Guest Additions Version: 5.2.22
    default: VirtualBox Version: 6.0
==> default: Mounting shared folders...
    default: /vagrant => /home/king/data/vagrantbox

```

## 查看进程
``` bash
$ ps aufx | grep virtualbox
king \_ grep --color=auto virtualbox
king  \_ /usr/lib/virtualbox/VBoxXPCOMIPCD
king    \_ /usr/lib/virtualbox/VBoxSVC --auto-shutdown
king     \_ /usr/lib/virtualbox/VBoxHeadless --comment vagrantbox_default_1564392888222_56525 --startvm 315ba5d9-fb4c-40ca-acf4-97dc75db70c7 --vrde config
```

## 进入shell
``` bash
$ vagrant ssh
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-56-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

vagrant@ubuntu:~$ pwd
/home/vagrant
vagrant@ubuntu:~$ logout
Connection to 127.0.0.1 closed.
```

## 关机&销毁
``` bash
$ vagrant halt
==> default: Attempting graceful shutdown of VM...
$ vagrant destroy
    default: Are you sure you want to destroy the 'default' VM? [y/N] y
==> default: Destroying VM and associated drives...
```

## 参考
+ <https://www.vagrantup.com/>
+ <https://www.cnblogs.com/hafiz/p/9175484.html>
+ <https://segmentfault.com/a/1190000000264347>
+ <https://github.com/astaxie/go-best-practice/blob/master/ebook/zh/01.3.md>
