title: Centos7开放端口
author: gslg
tags:
  - linux
  - centos7
  - firewalld
categories:
  - linux
date: 2019-07-08 14:15:00
---
Centos7默认使用`firewalld`作为防火墙,如果需要开放某些端口，有以下步骤，注意下面都是使用root用户进行操作:

- 通过`firewall-cmd`命令添加需要开放的端口,例如我们部署`ElasticSearch`时需要开放9200端口
  ```
  firewall-cmd --zone=public --add-port=9200/tcp --permanent
  ```
   > --zone 作用域  
   > --add-port=9200/tcp 添加需要开放的端口,格式是`端口/协议`  
   > --permanent 永久生效
   
基本上所有的linux工具命令都可以通过`man 工具`查看文档,例如查看`firewall-cmd`的文档就是`man firewall-cmd`

- 添加好要开放的端口后，需要重启防火墙:
 ```
  systemctl restart firewalld
 ```
- 查看已经开放的端口列表:
```
firewall-cmd --list-ports
```
- 关闭开放的端口
```
firewall-cmd --zone=public --remove-port=9200/tcp --permanent
```
- 查看监听的端口
```
netstat -lntp
```
 > 如果安装的是`Centos7 Minimal`版本的话,需要安装`net-tools`才能使用  `netstat`命令：
 ```
  yum install net-tools
 ```
- 查看占用某个端口的进程
```
netstat -lnp|grep 9200
```
 