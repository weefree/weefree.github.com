---
title:  "Ubuntu14.04 安装 Parse 私有云环境"
date:   2016-12-30 10:26:03
categories: [Nodejs]
tags: [Nodejs]
---

### 开始
作为 MBaaS（ Mobile Backend as a Service）的代表，Parse 自从关闭开源后，我们完全可以用开源的 Parse 代码搭建私有云环境，国内的leancloud、bomb就是提供这样云服务运营商，但是由于 MBaaS 云服务商无法做到应用隔离，服务很难做到稳定可靠，搭建私有云环境就可以解决这个问题，并且维护也不复杂。

官方文档：http://www.parse.com/

本文在 Ubuntu 环境搭建服务端模块，为了服务安全、简洁，管理后台 Dashboard 不放在服务端。

### 资源准备
Ubuntu v14.04
Mongodb v3.4.1 (https://www.mongodb.com/download-center?jmp=nav#community)
Nodejs v6.9.2 (https://nodejs.org/en/download/)
Parse Sever v2.3.1
Pase Dashboard v1.0.22

### 安装 NodeJS
NodeJS 有三种安装方式：包管理器安装、源码编译安装、用编译好的二进制文件安装。
包管理器安装方式网络环境有时不稳定，需要配置科学上网，具体安装方式可以参考官网文档（https://nodejs.org/en/download/package-manager/ ）
源码编译由于服务器配置不高，编译非常缓慢，本文使用官方提供的编译好的文件安装。

下载安装包并解压

``` shell
wget https://nodejs.org/dist/v6.9.2/node-v6.9.2-linux-x64.tar.xz
tar xvf node-v6.9.2-linux-x64.tar.xz
```

把 NodeJS 配置到环境变量

``` shell
vim /etc/profile

#在最后添加
export PATH="$PATH:/usr/app/node-v6.9.2/bin"
soruce /etc/profile
```

测试是否安装成功

``` shell
node --version
v6.9.2  #输出版本号，则安装成功
```

### 安装 MongoDB
MongoDB 也有多种安装方式，此处同样用编译后的文件安装

下载并解压

``` shell
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1404-3.4.1.tgz
tar zxvf mongodb-linux-x86_64-ubuntu1404-3.4.1.tgz
```

启动 MongoDB

``` shell
cd mongodb-linux-x86_64-ubuntu1404-3.4.1
mkdir data #创建数据存放目录
./bin/mongod --dbpath=data #启动成功会有等待客户端登录的提示
```

### 安装 Parse Server

``` shell
npm install -g parse-server
parse-server --appId APPLICATION_ID --masterKey MASTER_KEY --databaseURI mongodb://127.0.0.1:27017/dev
```

启动成功，访问http://[内网ip]:1337/parse ，会提示 {"error":"unauthorized"}

也可以集成到 express 中安装，具体见开源项目：git@github.com:weefree/parse_test.git

### 安装 Dashboard 管理工具

管理平台不需要在服务端安装，客户端远程访问更安全。本文是在 windows 下安装的管理工具

``` shell
npm install -g parse-dashboard
parse-dashboard --appId myAppId --masterKey myMasterKey --serverURL "http://[ip]:[port]/parse"
```

### 结束
至此，服务端工具已经安装完成，我们可以用几乎所有平台的 parse sdk 访问了，具体支持的SDK,见 Parse 开源项目：https://parseplatform.github.io/