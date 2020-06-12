---
title: Ansible实践
date: 2020-06-11 00:00:00
tags:
 - ansible
categories:
 - 运营运维
---

# 安装
``` bash
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

## 配置
``` bash
$ cat /etc/ansible/hosts  | sed -e "/^#/d" -e "/^$/d"
localhost ansible_connection=local
```

## 运行
```
$ ansible all -a "whoami"
localhost | SUCCESS | rc=0 >>
king
```

# 配置
文件路径`/etc/ansible/hosts`

``` ini
192.168.1.50
aserver.example.org
bserver.example.org
```

ansible 典型参数

+ 可用性`-m ping`
+ 执行commands `-a cmd`
+ 全局替换ssh登录用户 `-u`
+ 全局替换ssh key `--private-key`
+ sudo `--sudo`, `--sudo-user root2`
+ `--list-hosts` 列出hosts

``` bash
# 第一个参数有个特例，all表示所有
$ ansible localhost  -m ping
localhost | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
$ ansible all -a "whoami"
localhost | SUCCESS | rc=0 >>
king

$ sudo ansible all -a "whoami" -u root --sudo
localhost | SUCCESS | rc=0 >>
root

# 列出特定name的hosts
$ ansible test --list-hosts
  hosts (2):
    virtualbox
    localhost
```

## ssh key
``` bash
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/king/.ssh/id_rsa): /home/king/.ssh/ansible
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/king/.ssh/ansible.
Your public key has been saved in /home/king/.ssh/ansible.pub.
The key fingerprint is:
SHA256:T1liGGmbLRZI4lKIIdWn3gvWg8h/6F7hi/tgpFsNbPk king@king
The key's randomart image is:
+---[RSA 2048]----+
|o+.oo.....       |
|o .o..o +o       |
|  . .o ..=o .    |
|   o..  =..+     |
| . o*+..S.o      |
|  o+==+. o       |
|  .o+oEo  .      |
|   +o+o.         |
|  .o=+o          |
+----[SHA256]-----+
$ ssh-copy-id -i ~/.ssh/ansible -o "PubkeyAuthentication no" king@192.168.2.131
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/king/.ssh/ansible.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
king@192.168.2.131's password: 
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh -o 'PubkeyAuthentication no' 'king@192.168.2.131'"
and check to make sure that only the key(s) you wanted were added.
# 注意`SSH_AUTH_SOCK`
$ SSH_AUTH_SOCK=0  ssh -i /home/king/.ssh/ansible  king@192.168.2.131
```
```
$ SSH_AUTH_SOCK=0 ansible 192.168.2.131 -a whoami --private-key=/home/king/.ssh/ansible 
192.168.2.131 | SUCCESS | rc=0 >>
king
```

# 配置
+ ansible_ssh_host 将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.
+ ansible_ssh_port ssh端口号.如果不是默认的端口号,通过此变量设置.
+ ansible_ssh_user 默认的 ssh 用户名
+ ansible_ssh_pass ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)
+ ansible_sudo_pass sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)
+ ansible_sudo_exe (new in version 1.8) sudo 命令路径(适用于1.8及以上版本)
+ ansible_connection 与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.
+ ansible_ssh_private_key_file ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.
+ ansible_shell_type 目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.

修改`/etc/ansible/hosts`
``` ini
192.168.2.131 ansible_ssh_user=king ansible_ssh_private_key_file=/home/king/.ssh/ansible
```
``` bash
$ SSH_AUTH_SOCK=0 ansible all -a whoami 
localhost | SUCCESS | rc=0 >>
king

192.168.2.131 | SUCCESS | rc=0 >>
king

```

## 组和别名
``` ini
[test]
#192.168.2.131 ansible_ssh_user=king ansible_ssh_private_key_file=/home/king/ansible
virtualbox ansible_ssh_host=192.168.2.131 ansible_ssh_user=king ansible_ssh_private_key_file=/home/king/ansible
localhost              ansible_connection=local
```
``` bash
$ SSH_AUTH_SOCK=0 ansible test -a whoami 
localhost | SUCCESS | rc=0 >>
king

192.168.2.131 | SUCCESS | rc=0 >>
king

$ SSH_AUTH_SOCK=0 ansible test -a whoami 
localhost | SUCCESS | rc=0 >>
king

virtualbox | SUCCESS | rc=0 >>
king
```

### 主机&组变量
`/etc/ansible/hosts` 每个主机支持一系列参数, 如果一个组的参数有重复, 可以做成`组变量`
``` ini
[bastion]
b1 ansible_ssh_user=admin ansible_ssh_private_key_file=/home/king/.ssh/bastion
b2 ansible_ssh_user=admin ansible_ssh_private_key_file=/home/king/.ssh/bastion
b3 ansible_ssh_user=admin ansible_ssh_private_key_file=/home/king/.ssh/bastion
```
``` ini
[bastion]
b1 
b2 
b3 

[bastion:vars]
ansible_ssh_user=admin 
ansible_ssh_private_key_file=/home/king/.ssh/bastion
```

# 模块
+ `ansible-doc -l` 来列出支持的模块
+ `ansible-doc -s command` 列出特定模块的指引

> 默认ansible使用的模块是`command`，即可以执行一些`shell`命令。shell和command的用法基本一样，实际上shell模块执行命令的方式是在远程使用`/bin/sh`来执行的

``` bash
$ ansible test -m command -a whoami
localhost | SUCCESS | rc=0 >>
king

virtualbox | SUCCESS | rc=0 >>
king

$ ansible test -m shell -a whoami
localhost | SUCCESS | rc=0 >>
king

virtualbox | SUCCESS | rc=0 >>
king
```

> 最开始的`ping`也是一个特定的模块

## copy
拷贝文件到服务器, 会自动判断是否有修改

```
$ ansible virtualbox -m copy -a src="~/1.pdf dest=/tmp mode=0770 owner=king group=fa backup=yes"
virtualbox | SUCCESS => {
    "changed": true, 
    "checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "dest": "/tmp/1.pdf", 
    "gid": 1000, 
    "group": "fa", 
    "mode": "0770", 
    "owner": "king", 
    "path": "/tmp/1.pdf", 
    "size": 56111, 
    "state": "file", 
    "uid": 1000
}
$ ansible virtualbox -m copy -a src="~/1.pdf dest=/tmp mode=0770 owner=king group=fa backup=yes"
virtualbox | SUCCESS => {
    "changed": false, 
    "checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "dest": "/tmp/1.pdf", 
    "gid": 1000, 
    "group": "fa", 
    "mode": "0770", 
    "owner": "king", 
    "path": "/tmp/1.pdf", 
    "size": 56111, 
    "state": "file", 
    "uid": 1000
}

# 目录也支持
$ ansible virtualbox -m copy -a src="~/s dest=/tmp"
virtualbox | SUCCESS => {
    "changed": true, 
    "dest": "/tmp/", 
    "src": "/home/king/s"
}
```

> 如果使用"/"结尾，则拷贝的是目录中的文件，如果不以斜杠结尾，则拷贝的是目录加目录中的文件。

## file
管理文件、目录的属性，也可以创建文件或目录。

```
ansible-doc -s file
- name: Sets attributes of files
  action: file
      group       # file/directory的所属组
      owner       # file/directory的所有者
      mode        # 修改权限，格式可以是0644、'u+rwx'或'u=rw,g=r,o=r'等
      path=       # 指定待操作的文件，可使用别名'dest'或'name'来替代path
      recurse     # (默认no)递归修改文件的属性信息，要求state=directory
      src         # 创建链接时使用，指定链接的源文件
      state       # directory:如果目录不存在则递归创建
                  # file:文件不存在时，不会被创建(默认值)
                  # touch:touch由path指定的文件，即创建一个新文件，或修改其mtime和atime
                  # link:修改或创建软链接
                  # hard:修改或创建硬链接
```

``` bash
$ ansible virtualbox -m file -a 'path=/tmp/x/y/z state=directory owner=king group=fa mode=0755 recurse=yes'
virtualbox | SUCCESS => {
    "changed": true, 
    "gid": 1000, 
    "group": "fa", 
    "mode": "0755", 
    "owner": "king", 
    "path": "/tmp/x/y/z", 
    "size": 4096, 
    "state": "directory", 
    "uid": 1000
}
```

## fetch
和`copy`工作方式类似，只不过是从远程主机将文件拉取到本地端，存储时使用主机名作为目录树，且只能拉取文件不能拉取目录。

``` bash
ansible-doc -s fetch
- name: Fetches a file from remote nodes
  action: fetch
      dest=               # 本地存储拉取文件的目录。例如dest=/data，src=/etc/fstab，
                          # 远程主机名host.exp.com，则保存的路径为/data/host.exp.com/etc/fstab。
      fail_on_missing     # 当设置为yes时，如果拉取的源文件不存在，则此任务失败。默认为no。
      flat                # 改变拉取后的路径存储方式。如果设置为yes，且当dest以"/"结尾时，将直接把源文件
                          # 的basename存储在dest下。显然，应该考虑多个主机拉取时的文件覆盖情况。
      src=                # 远程主机上的源文件。只能是文件，不支持目录。在未来的版本中可能会支持目录递归拉取。
      validate_checksum   # fetch到文件后，检查其md5和源文件是否相同。
```

``` bash
$ ansible virtualbox -m fetch -a "src=/tmp/1.pdf dest=/tmp/"
virtualbox | SUCCESS => {
    "changed": true, 
    "checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "dest": "/tmp/virtualbox/tmp/1.pdf", 
    "md5sum": "d7fa85f4857527c3a58290a0be2b679a", 
    "remote_checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "remote_md5sum": null
}
$ ansible virtualbox -m fetch -a "src=/tmp/1.pdf dest=/tmp/ flat=yes"
virtualbox | SUCCESS => {
    "changed": true, 
    "checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "dest": "/tmp/1.pdf", 
    "md5sum": "d7fa85f4857527c3a58290a0be2b679a", 
    "remote_checksum": "429b1cd33dcb3e71e9975e424994906f10c6d98d", 
    "remote_md5sum": null
}

```

## apt
``` bash
# 安装 yes/safe/full/dist
$ ansible virtualbox -m apt -a "name=dos2unix update_cache=yes" --sudo --ask-sudo-pass
SUDO password: 
virtualbox | SUCCESS => {
    "cache_update_time": 1591944887, 
    "cache_updated": true, 
    "changed": true, 
    "stderr": "", 
    "stdout": "正在读取软件包列表..."
# 移除
$ ansible virtualbox -m apt -a "name=dos2unix state=absent" --sudo --ask-sudo-pass
# 安装 latest/absent/present
$ ansible virtualbox -m apt -a "name=dos2unix state=present" --sudo --ask-sudo-pass

```

## service

``` bash
# running,started,stopped,restarted,reloaded
$ ansible virtualbox -m service -a "name=nginx state=stopped " --sudo --ask-sudo-pass
SUDO password: 
virtualbox | SUCCESS => {
    "changed": true, 
    "name": "nginx", 
    "state": "stopped"
}
$ ansible virtualbox -m service -a "name=nginx state=stopped " --sudo --ask-sudo-pass
SUDO password: 
virtualbox | SUCCESS => {
    "changed": false, 
    "name": "nginx", 
    "state": "stopped"
}
$ ansible virtualbox -m service -a "name=nginx state=started " --sudo --ask-sudo-pass
SUDO password: 
virtualbox | SUCCESS => {
    "changed": true, 
    "name": "nginx", 
    "state": "started"
}

```

## playbook

一个playboook包含多个play, 每个play包含多个节

``` yaml
---
- hosts: virtualbox
  tasks: 
    - name: execute date cmd
      command: /bin/date
      register: date # 注册到一个变量, 以便打印stdout
    - name: execute shell whoami
      shell: whoami
      register: whoami
    - debug: var=date.stdout_lines
    - debug: msg="{{ whoami.stdout }}"
- hosts: virtualbox
  tasks: 
    - name: execute date cmd
      service: name=nginx state=started
      register: nginx
    - debug: msg="{{ nginx }}"
```
``` bash
$ ansible-playbook a.yaml 

PLAY ***************************************************************************

TASK [setup] *******************************************************************
ok: [virtualbox]

TASK [execute date cmd] ********************************************************
changed: [virtualbox]

TASK [execute shell whoami] ****************************************************
changed: [virtualbox]

TASK [debug] *******************************************************************
ok: [virtualbox] => {
    "date.stdout_lines": [
        "2020年 06月 12日 星期五 15:50:36 CST"
    ]
}

TASK [debug] *******************************************************************
ok: [virtualbox] => {
    "msg": "king"
}

PLAY ***************************************************************************

TASK [setup] *******************************************************************
ok: [virtualbox]

TASK [execute date cmd] ********************************************************
ok: [virtualbox]

TASK [debug] *******************************************************************
ok: [virtualbox] => {
    "msg": {
        "changed": false, 
        "name": "nginx", 
        "state": "started"
    }
}

PLAY RECAP *********************************************************************
virtualbox                 : ok=8    changed=2    unreachable=0    failed=0 
```

# Bastion
```
[server]
server1 ansible_ssh_host=10.2.40.11
server2 ansible_ssh_host=10.2.40.22 
server3 ansible_ssh_host=10.2.40.33


[server:vars]
ansible_ssh_user=admin 
ansible_ssh_private_key_file=/home/king/.ssh/bastion
ansible_ssh_common_args=' -o ProxyCommand="ssh -W %h:%p bastion_admin@bastion_ip"'
```


# 参考
1. [新手上路](https://ansible-tran.readthedocs.io/en/latest/docs/intro_getting_started.html#id1)
2. <https://askubuntu.com/questions/762541/ubuntu-16-04-ssh-sign-and-send-pubkey-signing-failed-agent-refused-operation>
3. [Running Ansible Through an SSH Bastion Host](https://blog.scottlowe.org/2015/12/24/running-ansible-through-ssh-bastion-host/)
4. [Ansible系列(一)：基本配置和使用](https://www.cnblogs.com/f-ck-need-u/p/7553186.html)
5. [Ansible系列(二)：选项和常用模块](https://www.cnblogs.com/f-ck-need-u/p/7550603.html)
6. <https://www.w3cschool.cn/automate_with_ansible/automate_with_ansible-db6727oq.html>
7. <https://www.jianshu.com/p/446f0ab2e131>
