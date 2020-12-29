---
title: 后端开发需要掌握的SSH命令和技巧
date: 2020-12-29 21:09:12
index_img: https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229134915683.png
categories:
  - tools
tags:
  - ssh
  - tools
---

# 后端开发需要掌握的 SSH 命令和技巧

> 本教程仅针对`MacOS`、`Linux`平台，不适用`Windows`

## 1. 常用 SSH 命令

- 生成本机上的 ssh 密钥对

```shell
ssh-keygen -t rsa -C "xxxxx@xxxxx.com"
```

生成的公钥、私钥在`~/.ssh`目录下:

![image-20201229134915683](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229134915683.png)

1. `id_rsa`是私钥
2. `id_rsa.pub`是公钥，可以拷贝里面的字符串放到 Github 的后台，使用 Git 协议可以免密
3. `config`是配置文件，后面会详细讲
4. `know_hosts`是你信任的服务器列表，连接服务器，输入`yes`后会加一条

- 连接服务器

```shell
ssh -p port user@server_ip
```

- 文件拷贝

```shell
# 注意这里的P用的大写的
scp -P port local_file user@server_ip:server_folder
scp -P port user@server_ip:server_file local_folder
```

## 2. 命令简化和免密

可以看到每次连接服务器或者拷贝文件都需要输入完整的用户名、ip 之后还要输入密码，非常麻烦

先使用配置文件来简化命令：

```shell
vi .ssh/config
```

这样配置：

```
Host dns
    HostName config.rigerwu.com
    User riger
    Port 22
```

`Host`后是你给服务器取的别名，我这里用的`“dns”`，`HostName`填域名或`ip`都可以，然后是用户名和端口。

以后我们直接使用`ssh dns`就可以访问服务器，名字还可以用`tab`来代码提示：

![image-20201229143645153](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229143645153.png)

包括`scp`命令也可以来简化：

```shell
scp local_file dns:server_folder
```

再来做免密登录，使用`ssh-copy-id`命令吧自己的公钥，拷贝到服务器端的 `.ssh/authorized_keys`中，这样以后就不用输入密码了：

![image-20201229144130799](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229144130799.png)

这个再`ssh dns`就已经可以直接免密登录了：

![image-20201229144411394](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229144411394.png)

甚至`scp`命令都能`tab`提示远程的路径：

![image-20201229144342128](https://riger.oss-cn-shanghai.aliyuncs.com/img/image-20201229144342128.png)

配合上`zsh`的代码提示或 alias，常用的文件拷贝就变得非常方便！

## 3. 解决 SSH 超时中断

```shell
vi .ssh/config
```

这样配置：

```
Host *
    TCPKeepAlive yes
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

但是有些服务端还是会一段时间无输入就终端连接，在服务端的`.bash_profile`中增加一行:

```shell
export TMOUT=0
```

设置不限制超时时间，这样就可以保持连接不中断了

Enjoy it ！
